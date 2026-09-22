# Static analysis follow-up: nagioscore cgi/ + base/ (2026-09-23)

Follow-up to `security-review-2026-09-23.md`, which explicitly did not exhaustively
hand-review the ~150 `strcpy`/~145 `sprintf` call sites across the full `cgi/`/`base/`
tree (mostly in the older non-JSON CGIs: `status.c`, `extinfo.c`, `showlog.c`,
`history.c`, `avail.c`, `summary.c`, `outages.c`, `trends.c`, `histogram.c`, `tac.c`,
`statusmap.c`, `config.c`) and recommended static-analysis tooling instead. This pass
does that.

Scope: `/claude/nagios-core/nagioscore` @ `al2023-4.4.14` tip (commit `e0d92db8`, tag
`kuriyama14`). Read-only — no fixes implemented, no git state touched.

## Tooling

`cppcheck`/`clang-tidy`/`scan-build` are not installed in this sandbox; installed
`cppcheck` 2.17.1 inside the `nagios-repro` fedora:40 udocker container via
`dnf install -y cppcheck` (in Fedora's default repos, no proxy issues). Did not pursue
`clang-tidy`/`clang-analyzer` — cppcheck alone produced a manageable, mostly-signal
result set and the time was better spent triaging than adding a second tool.

Built a source tarball from the current `al2023-4.4.14` tip the same way every
verification this session has (`git archive` matching
`scripts/generate-source-tarball.sh`'s convention), copied it into the container, and
ran `rpmbuild -bc --noclean` to get a real `include/config.h` (cppcheck is
meaningfully more accurate against the actual build configuration than a
config-less run). **Note**: the full link stage of that build failed with
`gcc: fatal error: cannot execute 'lto1'/'as': posix_spawn: Resource temporarily
unavailable` — this is host-level resource contention (multiple parallel agents doing
heavy `-flto=auto` builds in this session), not a code problem; every `.c` file had
already compiled to an object successfully by that point, and `include/config.h`
existed, which is all cppcheck needed. Did not chase a full link.

Command:
```
cppcheck --enable=warning,portability,performance --inline-suppr \
  -I include -I . -I cgi -I base \
  -DHAVE_CONFIG_H -DNSCGI -DJSON_NAGIOS_4X -DHAVE_SSL=1 \
  --suppress=missingIncludeSystem --max-ctu-depth=4 \
  cgi/ base/
```
(`style` checks deliberately NOT enabled — they're pure noise for a triage pass like
this; everything below is at `warning`/`portability`/`performance` severity or above.)

## Noise summary

**59 total findings**, zero pure-style noise (not enabled). Breakdown by check ID:

| Check ID | Count | Disposition |
|---|---|---|
| `nullPointerRedundantCheck` | 25 | Spot-checked 2 representative instances (see below); both were vestigial/dead defensive checks, not reachable bugs. Did **not** exhaustively trace all 25 — flagging the cluster's likely nature rather than claiming full triage. |
| `nullPointerOutOfMemory` | 15 | Traced all. **1 new CONFIRMED real bug cluster** (`status.c`, directly CGI-reachable), 1 independently-reconfirmed prior finding (`workers.c`, already known from this session's own memory-safety pass), 3 lower-priority same-class instances in `base/utils.c`. |
| `invalidPrintfArgType_uint` | 9 | Traced representative samples. Real but cosmetic (signed/unsigned format-specifier mismatch, no memory-safety impact). |
| `invalidPrintfArgType_sint` | 5 | Same as above. |
| `arrayIndexOutOfBoundsCond` | 2 | Traced. **1 CONFIRMED real off-by-one bug** (`config.c`, both hits are the same bug hitting two sibling arrays). |
| `uselessAssignmentPtrArg` | 1 | Traced. Confirmed a true no-op, confirmed NOT a use-after-free risk. Pure dead-code nit. |
| `invalidscanf` | 1 | Traced. Real code smell (unbounded `sscanf %s` into a 5-byte buffer) but zero current exploitability — traced every call site, all pass compiler-fixed `__DATE__`/`__TIME__`, never attacker input. |
| `ctunullpointer` | 1 | Traced. Same vestigial-check pattern as the `nullPointerRedundantCheck` cluster, not a reachable bug. |

**Actionable rate: ~3 distinct real bug locations out of 59 raw hits** (the `status.c`
cluster, the `config.c` off-by-one, and the `workers.c` calloc — the last already
partially addressed this session, see below) plus ~14 cosmetic format-string nits
worth a trivial cleanup sometime. The `nullPointerRedundantCheck`/`ctunullpointer`
family (26 combined) appears to be mostly noise from cppcheck's cross-translation-unit
analysis not seeing call-site guarantees — 2/2 traced samples were non-issues, so a
full trace of the remaining ~24 is likely low-yield, but wasn't exhaustively done.

## Findings, most severe first

### 1. CONFIRMED, MEDIUM-HIGH — `status.c`: unchecked `malloc`/`strdup` immediately dereferenced, in the CGI's own `host` query-parameter handling

**File:** `cgi/status.c:246-274`, function building a regex-style host filter from
`host_name` (a `hoststatus.cgi`/`status.cgi` query-string parameter — genuinely
attacker/client-controlled, unlike most of this session's other OOM findings which
required a resource race during internal worker registration).

```c
host_filter = malloc(sizeof(char) * (strlen(host_name) * 2 + 3));
len = strlen(host_name);
for(i = 0; i < len; i++, regex_i++) {
    if(host_name[i] == '*') {
        host_filter[regex_i++] = '.';   /* <- NULL deref if malloc failed */
        ...
```
and, a few lines later, a second instance on the same pattern:
```c
host_address = strdup(temp_host->address);
host_filter = malloc(sizeof(char) * (strlen(host_address) * 2 + 3));
len = strlen(host_address);
for(i = 0; i < len; i++, regex_i++) {
    host_filter[regex_i] = host_address[i];   /* <- NULL deref if malloc failed */
```

Neither `host_filter` nor `host_address` is checked for `NULL` before being
dereferenced. Same bug class as this session's already-fixed
`base/workers.c`/`lib/worker.c`/`cgi/getcgi.c` findings (kuriyama13) — OOM-triggered,
not attacker-controlled-content, but here the *path to the code* is a plain CGI query
parameter rather than an internal worker-registration race, so it's marginally more
reachable in principle (any request to `status.cgi?host=...` exercises this function).

**Confidence:** CONFIRMED (traced both allocation sites to their immediate
unconditional dereferences).

**Suggested fix:** same shape as the kuriyama13 fixes — check for `NULL` right after
each `malloc`/`strdup`, bail out (this function is deep in HTML-rendering code, so the
existing codebase's convention elsewhere in `status.c` for a fatal allocation failure
during page rendering is worth matching rather than inventing a new pattern here).

### 2. CONFIRMED, MEDIUM — `config.c`: off-by-one array bounds in `$ARGn$` macro display

**File:** `cgi/config.c:2210-2213` (declarations) and `:2304-2307` (the bug):

```c
char *command_args[MAX_COMMAND_ARGUMENTS];      /* MAX_COMMAND_ARGUMENTS == 32, include/macros.h:32 */
int arg_count[MAX_COMMAND_ARGUMENTS], lead_space[MAX_COMMAND_ARGUMENTS], trail_space[MAX_COMMAND_ARGUMENTS];
...
if((i > 0) && (i <= MAX_COMMAND_ARGUMENTS)) {   /* allows i == 32, valid range is 0..31 */
    arg_count[i]++;                              /* <- one-past-the-end write when i==32 */
    if(command_args[i]) {                        /* <- one-past-the-end read when i==32 */
```

`i` comes from `atoi(cc)` where `cc` is the numeric portion parsed out of a `$ARGn$`
macro found inside a *command definition's command line* (`temp_command->command_line`,
displayed by `config.cgi`'s object-configuration viewer). This means triggering it
requires a command definition containing `$ARG32$` or higher in `nagios.cfg`-style
config — i.e. admin/config-file trust level, not a raw unauthenticated HTTP request —
but it's still a genuine stack out-of-bounds read+write once triggered (all four arrays
are stack-local `int`/`char*` arrays in the same function), and it's a trivially clean
fix.

**Confidence:** CONFIRMED (`MAX_COMMAND_ARGUMENTS` is unambiguously the array size;
`i <= MAX_COMMAND_ARGUMENTS` unambiguously allows the one-past-the-end index).

**Suggested fix:** `i <= MAX_COMMAND_ARGUMENTS` → `i < MAX_COMMAND_ARGUMENTS` (matches
every other loop bound against this same constant elsewhere in the file, e.g.
`cgi/config.c:2224`, which correctly uses `i < MAX_COMMAND_ARGUMENTS`).

### 3. CONFIRMED, LOW-MEDIUM — `base/workers.c`: unchecked `calloc()` in `register_worker()` (independently reconfirms a gap already noted, not yet fixed, this session)

**File:** `base/workers.c:971-974`, inside the same `register_worker()` function this
session already hardened (kuriyama13, commit "Fix realloc()-failure memory-safety bugs
across cgi/base/lib") — that fix specifically addressed the `realloc()` call sites in
this function and deliberately left the neighboring `calloc()` sites unaddressed
("scope-limiting decision... not something else in this exact function", per that
commit's own reasoning), noting it as pre-existing and out of scope for that pass.
cppcheck's cross-translation-unit analysis independently flags exactly that gap:

```c
command_handlers = calloc(1, sizeof(struct wproc_list));
command_handlers->wps = calloc(1, sizeof(struct wproc_worker**));  /* <- NULL deref if first calloc failed */
command_handlers->len = 1;
command_handlers->wps[0] = worker;
```

Same OOM-during-worker-registration reachability and severity as the already-fixed
`realloc()` sites two lines below this block. Worth closing out as a small, consistent
follow-up to kuriyama13 rather than a new investigation — the fix shape is identical
to what's already there for the `realloc()` calls in the same function (temp variable,
NULL check, `free(worker); kvvec_destroy(info, 0); return 500;`).

**Confidence:** CONFIRMED (tool finding + manual confirmation the calloc result is
unconditionally dereferenced two lines later, no NULL check anywhere in between).

### 4. CONFIRMED but NOT currently exploitable — `jsonutils.c`: unbounded `sscanf("%s", ...)` into a 5-byte buffer

**File:** `cgi/jsonutils.c:1332-1345`, `compile_time()`:

```c
time_t compile_time(const char *date, const char *time) {
    char buf[5];
    ...
    sscanf(date, "%s %d %d", buf, &day, &year);   /* unbounded %s into a 5-byte buffer */
```

This is a real, textbook "unbounded `%s` into a fixed buffer" pattern — the same class
of bug as `gets()` — and would be a genuine stack buffer overflow **if** `date` were
ever attacker-influenced text longer than 4 characters. Traced every call site
(`cgi/statusjson.c:1138`, `cgi/objectjson.c:1133`, `cgi/archivejson.c:764`): all three
call it as `compile_time(__DATE__, __TIME__)` — `__DATE__` is a C compiler builtin
macro that expands to the build timestamp (format always `"Mon DD YYYY"`, 3-letter
month) at *compile time*, never anything request- or config-derived. Since the month
abbreviation is always exactly 3 characters, `buf[5]` is always sufficient in every
actual call (`"Jan\0"` = 4 bytes). **Not currently reachable from any external input.**

**Confidence:** CONFIRMED as a real pattern, CONFIRMED not currently exploitable
(exhaustively checked all 3 call sites, the function's only callers).

**Suggested fix:** worth hardening anyway since it's fragile-by-construction (any
future caller passing real user/config data here would silently reintroduce a stack
overflow with no compiler warning) — bound the format to `%4s` (or reuse
`MAX_INPUT_BUFFER`-style project convention), independent of current non-exploitability.
Low priority given zero current risk, but cheap and worth doing opportunistically.

### 5. CONFIRMED, LOW — `base/utils.c`: two more unchecked-`strdup()` OOM crashes, lower reachability

**File:** `base/utils.c:1897` (`homedir = strdup(log_file);` then immediately
`strrchr(homedir, '/')` with no NULL check — daemon-startup working-directory
selection, not network-reachable) and `base/utils.c:2413-2414` (`var =
strdup(vartok); val = strdup(valtok);` then immediately `strcmp(var, "file_time")`
with no NULL check — passive check-result file parsing, config/filesystem-driven, not
directly network-reachable). Same bug class and severity tier as the rest, lower
priority given the reachability is startup-time / internal-file-format parsing rather
than a live CGI request path.

**Confidence:** CONFIRMED (tool finding + manual confirmation of immediate
unconditional dereference in both cases).

### 6. CONFIRMED, cosmetic only — 14 signed/unsigned `printf`-family format-specifier mismatches

`cgi/jsonutils.c:1069` (`%u` for a `signed int` — the `format_duration()` helper),
`cgi/extinfo.c:1813,1891,1893` (`%ld` for an `unsigned long`), `base/events.c:718,724`
(`%d` for an `unsigned int`), plus similar hits cppcheck grouped under the same two
check IDs elsewhere. All are same-width signed/unsigned mismatches — the compiled
output only differs from the "correct" specifier when the underlying value's sign bit
would actually matter (e.g. a duration/uptime value that's never actually negative in
practice), so these are correctness nits with no buffer-sizing or memory-safety
implication (`printf`-family functions read the argument by width, not by
signedness, for `%d`/`%u`/`%ld`/`%lu`). Worth a trivial batch cleanup sometime, not
urgent.

### 7. Traced and confirmed NOT bugs (representative samples from the two largest clusters)

- **`base/checks.c:706-736` (`debug_async_service`/`debug_async_host`)**: cppcheck
  flags a `(cr == NULL) ? "NULL" : cr->output` check as "redundant or a possible null
  deref" because the *same function* unconditionally dereferences `cr->check_type` at
  its very first statement. Traced the (single) call site
  (`base/checks.c:1217,1236`, inside `handle_async_service_check_result()`): `cr` is
  already validated non-NULL via `is_valid_check_result_data(hst, cr)` and directly
  dereferenced (`cr->check_type`) several lines before `debug_async_service()` is ever
  called. The `(cr == NULL) ? ...` check is genuinely dead/vestigial defensive code
  left over from some earlier version of the function, not a reachable bug. Low-value
  cleanup candidate (remove the vestigial ternary), not a security finding.
- **`cgi/archivejson.c:2930-2952`**: same shape — `current_host->hostp` dereferenced
  unconditionally, then `NULL != current_host` checked a few lines later in the same
  `switch` case. Same likely explanation (the caller's contract guarantees
  `current_host` is non-NULL whenever `current_object_type == AU_OBJTYPE_HOST`), not
  independently re-verified as thoroughly as the `checks.c` case above given time
  budget, but the pattern match is exact enough to treat with the same low-confidence
  disposition. **Not exhaustively re-traced across the other ~23
  `nullPointerRedundantCheck` hits in `status.c`/`statusmap.c`/`statusjson.c`** — flag
  as a possible follow-up if a deeper pass on this specific cluster is wanted, but
  based on 2/2 traced samples being non-issues, expect low yield.
- **`cgi/avail.c:2923` (`uselessAssignmentPtrArg`)**: `free_archived_state_list(archived_state
  *as_list)` does `as_list = NULL;` on a pass-by-value parameter right before
  `return` — confirmed genuinely useless (the caller's own pointer is never touched,
  since C passes this by value). Confirmed **not** a use-after-free risk either: both
  call sites (`cgi/avail.c:2901-2902`) immediately `free()` the containing struct that
  held the now-stale pointer, so nothing ever reads it again. Pure dead-code/misleading
  line, not a bug.

## What this does and doesn't tell you

cppcheck's signal was concentrated almost entirely in the `nullPointerOutOfMemory`/
`arrayIndexOutOfBoundsCond`/`invalidscanf` categories — every one of those was worth
tracing and mostly turned out real (if generally low-exploitability, matching this
session's established pattern of "real bugs, mostly OOM-gated or admin-trust-gated,
not directly remotely exploitable by an unauthenticated attacker"). The
`nullPointerRedundantCheck` cluster (the largest single bucket, 25 hits) looks like
mostly tool noise from missing call-site context, based on a 2-sample trace — a
deeper pass there is a reasonable but lower-priority follow-up, not something this
pass claims to have resolved. Did not run `--check-level=exhaustive` (cppcheck's own
suggestion in its output, `normalCheckLevelMaxBranches` notes on `config.c`/`avail.c`)
— that mode analyzes every branch combination rather than a bounded subset and would
likely take substantially longer; worth trying if a future pass wants maximum
coverage over this same code.
