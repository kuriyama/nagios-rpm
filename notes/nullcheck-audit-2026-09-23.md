# nullPointerRedundantCheck cluster: full trace (2026-09-23)

Follow-up to `static-analysis-2026-09-23.md`, which spot-checked 2 of 25
`nullPointerRedundantCheck` hits (plus 1 `ctunullpointer`) and left the remaining
~23 untraced, flagging it as "lower-priority, expect low yield." This pass traces
every one of them.

Branch: `feature/nullcheck-audit-fixes` (worktree `nagioscore-wt-nullcheck-audit`,
based on `al2023-4.4.14` @ `82a7e976` / kuriyama16). Commit `4a124dce`.

## Result: the 2-sample prediction was wrong. 2 real bugs found, 3 fix sites.

Re-ran cppcheck (same command as the original report) against a fresh
`rpmbuild -bc --noclean` tree from the current tip. Got the identical 25+1=26
hits. Traced all of them:

| Location | Hits | Disposition |
|---|---|---|
| `cgi/archivejson.c:4021` (`service_status`) | 1 | **REAL BUG — fixed** |
| `cgi/status.c:3511-3512` / `:4756-4757`* | 4 | **REAL BUG — fixed** (1 bug pattern, 2 sites) |
| `cgi/archivejson.c:2930,2952` (`current_host`/`current_service`) | 2 | Vestigial — confirmed |
| `cgi/statusjson.c:2881` (`temp_host`) | 1 | Vestigial — confirmed |
| `cgi/statusmap.c:1661,1670,1672,1677,1769` (`temp_host`) | 5 | Vestigial — confirmed |
| `base/checks.c:706-736` (`cr`) | 12 | Vestigial — confirmed (matches original report's sample) |
| `base/utils.c:458` (`ctunullpointer`, `check_result_source`) | 1 | Vestigial — confirmed (deeper trace than original report) |

\* Line numbers are from cppcheck's build (which analyzes the *patched* source
tree — the RPM spec applies `nagios-0010-remove-information-leak.patch`, which
deletes 23 lines from `status.c` around line 555, shifting everything after it).
The actual bug in the raw git source is at lines 3534/3535 and 4779/4780 — found
by matching the flagged code's content, not trusting the container's line numbers
verbatim, after noticing the ~23-line discrepancy and tracing it to that patch.

**Actionable rate for this cluster: 2 bugs / 26 hits (~8%)**, not the 0% the
2-sample trace predicted — worth remembering that a small sample of a
cross-translation-unit-analysis cluster can look uniformly like noise and still
be hiding something; the vestigial cases and the real bugs have an *identical*
surface shape to cppcheck (a NULL check next to an unconditional dereference of
the same variable), so severity can't be judged from the tool's output alone at
all, only from tracing each one back to its actual guarantees.

## The 2 real bugs

### 1. `cgi/archivejson.c` — inverted NULL check, guaranteed crash on the branch that should be safe

`compute_service_availability()`, in the `AU_STATE_CURRENT_STATE` case of
`assumed_initial_service_state` (a CGI-parameter-controlled enum, set via
`cgi_data->assumed_initial_service_state` parsed from the request):

```c
case AU_STATE_CURRENT_STATE:
    if(service_status == NULL) {          /* was: == NULL */
        switch(service_status->status) {  /* unconditional deref inside the NULL branch */
```

The condition was backwards: the code enters the `switch` (dereferencing
`service_status->status`) exactly when `service_status` **is** NULL. A few dozen
lines earlier in the same function, the correct pattern appears
(`(service_status != NULL)` guarding an identical `switch`) — this looks like a
copy-paste of that earlier block with the condition flipped, rather than a typo
that changes behavior subtly; it inverts the *entire* branch's reachability.

**Reachable**: `service_status = find_servicestatus(...)` returns NULL whenever
the named service has no live status data (e.g. recently removed from config,
or status data not yet loaded) — plausible in normal operation, not just under
OOM. Triggering requires an `archivejson.cgi` availability-report request with
`assumeinitialstate` (or however the CGI form field name maps to
`assume_initial_state`/`assumed_initial_service_state`) set to request the
"current state" fallback for such a service.

**Fix**: `==` → `!=` (one character).

### 2. `cgi/status.c` — status-class fallback computed, then ignored, in `show_servicegroup_grid()` and `show_hostgroup_grid()`

Both functions (the "grid" display style of `status.cgi`, two independent
call sites with the same bug) do:

```c
temp_servicestatus = find_servicestatus(temp_member2->host_name, temp_member2->service_description);
if(temp_servicestatus == NULL)
    service_status_class = "NULL";
else if(temp_servicestatus->status == SERVICE_OK)
    service_status_class = "OK";
/* ... */

printf("<a href='%s?type=%d&host=%s", EXTINFO_CGI, DISPLAY_SERVICE_INFO,
        url_encode(temp_servicestatus->host_name));   /* <- unconditional, even when NULL */
printf("&service=%s' ...>%s</a>&nbsp;",
        url_encode(temp_servicestatus->description), service_status_class,
        temp_servicestatus->description);              /* <- same */
```

The `if/else-if` chain correctly handles `temp_servicestatus == NULL` for the
purpose of choosing a CSS class string (falling back to `"NULL"`), but the two
`printf`s immediately after **unconditionally** dereference `temp_servicestatus`
regardless of which branch was taken — including the NULL one. Whenever a
servicegroup/hostgroup grid includes a service with no live status data, this
crashes the CGI.

**Fix**: use the loop variable that produced the lookup key instead of the
lookup result — `temp_member2->host_name`/`temp_member2->service_description`
in `show_servicegroup_grid()`, `temp_service->host_name`/`temp_service->description`
in `show_hostgroup_grid()`. Both loop variables are guaranteed non-NULL by their
enclosing `for` loop's continuation condition, and hold the identical
host/service identity that was passed to `find_servicestatus()`, so this changes
nothing about what's displayed in the non-NULL case while eliminating the crash
in the NULL case.

## The vestigial cluster, traced in full

- **`cgi/archivejson.c:2930,2952`** (`current_host->hostp`/`current_service->servicep`
  dereferenced before a later `NULL != current_host` check, inside
  `json_archive_statechange_passes_selection()`): traced the single call site
  (`json_archive_statechangelist()`) and the log-entry construction path
  (`parse_states_and_alerts()` in `archiveutils.c`) that produces the `au_host
  *`/`au_service *` values passed in. That function looks up the host/service via
  `au_find_host()`/`au_find_service()` and, on a miss, **creates** the entry via
  `au_add_host_and_sort()`/`au_add_service_and_sort()` before ever adding the log
  entry to the list — returning early (skipping the entry) if that allocation
  also fails. So every entry that reaches the selection-filter function carries a
  non-NULL host/service object by construction. The later `NULL != current_host`
  check is genuinely dead code, not a guard for a reachable case.
- **`cgi/statusjson.c:2881`** (`find_hoststatus(temp_host->name)`, inside
  `json_status_service_passes_host_selection(host *temp_host, ...)`): the very
  first statement in the function is `is_authorized_for_host(temp_host, ...)`,
  which was independently confirmed and unit-tested earlier this session
  (`t-tap/test_cgiauth.c`) to return `FALSE` on a NULL host argument without
  dereferencing it — so a NULL `temp_host` bails out via `return 0` before ever
  reaching the flagged line. Several `NULL != temp_host` checks further down in
  this same function are similarly redundant once this is understood.
- **`cgi/statusmap.c:1661,1670,1672,1677,1769`**: all five are inside a single
  `for(temp_host = host_list; temp_host != NULL; temp_host = temp_host->next)`
  loop — the loop's own continuation condition guarantees non-NULL for the
  entire body, including at line 1769 (confirmed by reading the intervening code,
  no early-exit or reassignment of `temp_host` inside the loop). cppcheck's
  cross-translation-unit analysis doesn't track this multi-hundred-line
  guarantee; all five are the same non-issue.
- **`base/checks.c:706-736`** (12 hits, `cr` in `debug_async_service`/
  `debug_async_host`): matches the original report's traced sample exactly — the
  functions dereference `cr->check_type`/`cr->check_options`/etc. unconditionally
  as several sibling arguments to the same `log_debug_info()` call that also
  contains a `(cr == NULL) ? "NULL" : cr->output` ternary. Since C does not
  guarantee left-to-right argument evaluation, this ternary can never actually
  prevent a crash (a sibling argument dereferences `cr` regardless of which
  argument evaluates first) — it's dead by construction, independent of whether
  `cr` can ever be NULL at the call site. It also never can be: both call sites
  (`handle_async_service_check_result()`/`handle_async_host_check_result()`) are
  preceded by `is_valid_check_result_data(hst, cr)`, which explicitly checks
  `cr == NULL` and returns `FALSE` (causing the caller to bail via `return
  ERROR`) first.
- **`base/utils.c:458`** (`ctunullpointer`, `check_result_source()`): the
  function itself has no internal NULL check on its `cr` parameter. Traced all
  4 call sites in the tree: 2 are inside `debug_async_service`/`debug_async_host`
  (covered above — `cr` already guaranteed non-NULL by `is_valid_check_result_data`
  before either debug function is even called), 1 (`base/utils.c:2288`,
  `process_check_result()`) has its own explicit `if (!cr) return ERROR;`
  immediately before the call, and 1 (`base/nerd.c:363`) dereferences
  `cr->finish_time.tv_sec` as a sibling argument in the same `asprintf()` call
  that also calls `check_result_source(cr)` — same "can't be protected by
  evaluation order" reasoning as the `checks.c` case. All 4 call sites are safe;
  `check_result_source()`'s missing internal check is unreachable in every
  actual caller, not just the one cppcheck's local trace happened to examine.

## Verification

fedora:40 `nagios-repro` container. Fresh `rpmbuild -ba` (full build + install +
package, `--define '%_lto_cflags %{nil}'` workaround for this container's
LTO/posix_spawn resource contention) — exit 0, zero `error:` lines, all 8 RPMs +
SRPM produced, zero new warnings on `status.c`/`archivejson.c` (the one warning
on `status.c` — unused variable `vidurl` — is pre-existing, unrelated line).
Then `rpmbuild -bc --noclean` for a persistent build tree, `cd t-tap && make
test`: all 12 targets pass, `Files=12, Tests=6744, Result: PASS`, 0 failures —
matches the established baseline exactly, no regressions.
