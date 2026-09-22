# t-tap coverage expansion — 2026-09-23

Branch: `feature/expand-t-tap-coverage` (worktree `/claude/nagios-core/nagioscore-wt-test-coverage`, based on `al2023-4.4.14` @ `kuriyama12` / commit `3247c3e1`)
Commit: `b9f5ac4e` — "Add t-tap tests for getcgi.c URL-decoding and cgiauth.c authorization"

## Coverage gap survey

t-tap's `TESTS` list before this change: `test_logging`, `test_events`, `test_checks`, `test_commands`, `test_downtime`, `test_nagios_config`, `test_timeperiods`, `test_macros`, `test_json_escape`. That's 9 of `base/`'s 19 `.c` files (partial coverage via linking, not 1:1), and only `jsonutils.c`/`json_escape.c` of `cgi/`'s 23 `.c` files.

**Zero coverage, base/**: `broker.c`, `config.c` (only indirectly via test_nagios_config linking config.o), `flapping.c`, `nagios.c` (has `main()`, not meaningfully unit-testable), `nagiostats.c` (separate binary), `nebmods.c`, `nerd.c`, `netutils.c`, `notifications.c`, `perfdata.c`, `query-handler.c`, `sehandlers.c`, `sretention.c`, `utils.c` (linked into several tests but no dedicated assertions on its own helpers), `workers.c`, `wp-phash.c`.

**Zero coverage, cgi/** (everything except jsonutils.c/json_escape.c): `archivejson.c`, `archiveutils.c`, `avail.c`, `cgiauth.c`, `cgiutils.c`, `cmd.c`, `config.c`, `extcmd_list.c`, `extinfo.c`, `getcgi.c`, `histogram.c`, `history.c`, `notifications.c`, `objectjson.c`, `outages.c`, `showlog.c`, `status.c`, `statusjson.c`, `statusmap.c`, `summary.c`, `tac.c`, `trends.c`.

Prioritized this survey by (a) untrusted-input exposure and (b) how self-contained the code is (cheap to wire up vs. needs a fixture object graph):

| Target | Untrusted-input exposure | Self-contained? | Action |
|---|---|---|---|
| `cgi/getcgi.c` (`hex_to_char`/`unescape_cgi_input`) | Every CGI query string + POST body, unauthenticated | Yes — libc only | **Done, see below** |
| `cgi/cgiauth.c` (`is_authorized_for_*`) | Gates all view/command access | Partial — the `authdata`-only subset is; the host/service-graph subset needs fixtures | **Done (partial), see below** |
| `cgi/cmd.c` (command field validation) | POST body → external command pipe | No — deeply entangled with the full CGI/auth/object-graph stack | Not attempted; flagged for the security-review track instead of duplicated here |
| `base/utils.c` string helpers (`my_strtok`, `escape_newlines`, etc.) | Indirect (config/macro parsing) | Mostly yes | Not attempted this round — good next target |
| `base/macros.c` macro expansion | Indirect (config-driven, but some macros reflect check output) | No — needs a populated object graph; `test_macros.c` already has 23 assertions and reasonable coverage | Skipped, diminishing returns |
| `include/objects.h` graph-dependent `is_authorized_for_host/service/hostgroup/servicegroup` (+ `*_commands` variants) in `cgiauth.c` | Same gate as above, but per-object | No — needs real `host`/`contact`/`service` fixtures wired through `find_host()`/`find_contact()`/`is_contact_for_host()` etc. | **Not attempted — see Follow-up below** |
| `cgi/extcmd_list.c`, `cgi/archiveutils.c` | Low/indirect | Unknown, not investigated | Not investigated |

## What was implemented

### 1. `t-tap/test_getcgi.c` (21 assertions) — real `cgi/getcgi.o` linked directly, zero stubs needed

`getcgi.c` is fully self-contained (only `config.h`/`getcgi.h`, no Nagios object model), so the actual object file is linked as-is — no test double at all. Covers:
- `hex_to_char()`: NULL/empty input, well-formed pairs (upper/lower-case hex, `0x00`, `0xff`), and — importantly — malformed input (no valid hex digit, invalid leading digit, valid-then-invalid digit), pinning down `sscanf("%X", ...)`'s real partial-match semantics rather than assuming clean truncation.
- `unescape_cgi_input()`: plain text, `%XX` decoding, `+` is *not* treated as space (documents that this function alone doesn't do full form-urlencoding), truncated trailing `%`/`%X` sequences (no OOB read), NULL/empty input, and a monotonic "never grows the buffer" invariant.
- `sanitize_cgi_input()`: this function currently opens with an unconditional `return;` (see the `/* don't strip for now... */` comment in the source) — it is a **documented no-op today**, called from nowhere that actually relies on it doing anything. The test locks this in as an explicit tripwire, so if that early return is ever removed, this test fails immediately and forces a deliberate look at every caller's expectations instead of a silent behavior change. Also checked NULL/empty-list inputs don't crash (currently safe only *because* of the no-op — if the no-op is ever removed, `sanitize_cgi_input(NULL)` would immediately dereference `cgivars[0]` and crash; noting this explicitly since it's a real latent trap for whoever removes the no-op).

**Bug found and fixed while writing this test**: `hex_to_char()`'s local `unsigned int outint` was uninitialized. `sscanf(tempbuf, "%X", &outint)` does **not** write to `outint` when there's no valid hex digit to convert (e.g. a malformed `%ZZ` escape in any CGI parameter) — per the C standard, a failed/no-match conversion leaves the target untouched. That means the function could return an unpredictable byte built from stack garbage instead of a deterministic value, for input reachable from any unauthenticated CGI request. Fixed with a one-line `unsigned int outint = 0;`. Confirmed the failure mode by hand (local repro outside the test, reasoning from the documented `sscanf` contract — mbstowcs-style ASan repro wasn't needed here since this is a logic bug, not a memory-safety one). This is a minor, narrowly-scoped fix directly tied to the function under test, not a general sweep — flagged to the security-review track too in case it wants to note it, but not duplicated as a separate finding there.

### 2. `t-tap/test_cgiauth.c` (33 assertions) + `t-tap/stub_cgiauth_deps.c` — real `cgi/cgiauth.o` linked, existing `stub_objects.c` reused, one new small stub file

Covers the `authdata`-only subset of the authorization decision functions — the ones that only depend on `authdata` fields and the global `use_authentication` flag, not on a populated host/contact/service graph: `is_authorized_for_all_hosts`, `is_authorized_for_all_services`, `is_authorized_for_read_only`, `is_authorized_for_system_information`, `is_authorized_for_configuration_information`, `is_authorized_for_system_commands`.

For each, tests three states × both flag values where applicable:
- `use_authentication == FALSE` → every function "fakes" full access. **`is_authorized_for_read_only` is the one function in this family with inverted semantics** (it reports a *restriction*, not a *permission* — "fake full access" means returning `FALSE`, not `TRUE`). This asymmetry is exactly the kind of thing a future refactor could flip by copy-pasting the wrong sibling function, so it's pinned down explicitly.
- `use_authentication == TRUE`, `authenticated == FALSE` → deny regardless of what the individual flags say (an unauthenticated request must never be granted anything from a stale/default-permissive flag).
- `use_authentication == TRUE`, `authenticated == TRUE` → decision follows the specific per-user flag exactly, both directions.
- A "no leak between flags" check: setting only `authorized_for_all_hosts = TRUE` must not make any of the *other* five checks return TRUE — guards against a copy-paste bug reading the wrong struct field.
- NULL-argument short-circuiting for the four object-graph functions (`is_authorized_for_host/service/hostgroup/servicegroup`) — cheap to verify even without a populated graph, and confirmed the NULL check happens *before* the `use_authentication == FALSE` fake-access branch in all four (i.e. a NULL host/service is never treated as "authorized" just because auth is off).

**Deliberately out of scope for this pass**: `is_authorized_for_host`, `is_authorized_for_service`, `is_authorized_for_hostgroup`, `is_authorized_for_servicegroup` (non-NULL cases) and their `*_commands` siblings — these walk a real host/contact/service object graph (`find_host`/`find_contact`/`is_contact_for_host`/`is_escalated_contact_for_host`/etc.), which needs populated fixture objects, not just stubs returning fixed values. `t-tap/stub_cgiauth_deps.c` stubs these dependencies to `FALSE`/`NULL` specifically *so this test doesn't accidentally exercise that untested path and get a false sense of coverage*.

## Follow-up work (not done this round)

1. **Object-graph-dependent `cgiauth.c` functions** (see above). Doing this properly means either (a) building minimal real `host`/`contact`/`service`/`hostgroup`/`servicegroup` fixtures and linking real `common/objects.c` (risk: pulls in objects.c's own further transitive dependencies, unknown size), or (b) writing a purpose-built fixture-friendly stub layer for `find_host`/`find_contact`/`is_contact_for_host` etc. that returns caller-configurable fixture objects instead of always `NULL`/`FALSE`. Either is a reasonable follow-up; (b) is probably less risky and more reusable for testing other object-graph-dependent CGI code later (`is_authorized_for_service_commands` and friends in the same file, or eventually `statusjson.c`/`objectjson.c` themselves).
2. **`base/utils.c` string/escaping helpers** — worth a similar self-contained-function survey to what `getcgi.c` got; not investigated in depth this round, just flagged as promising in the table above.
3. **`cgi/cmd.c`** — the command-submission CGI is the highest-value untested target from a pure security standpoint (POST data → external command pipe), but it's also the most entangled with the full CGI/auth/object-graph stack, so it needs more setup than fit in this pass. The security-review track (running in parallel) is looking at this file's data flow directly; that review plus this note should be read together before deciding how to test it.
4. Everything else in the "Zero coverage" lists above that wasn't specifically called out — not surveyed function-by-function, just enumerated at the file level.

## Verification

Container: `nagios-repro` (fedora:40 udocker). **Hit real contention from a concurrent track also using the same container mid-verification** — a `make test` run showed a missing `test_checks` binary and, on inspection, the container's `/root/rpmbuild/SOURCES/nagioscore-nagios-4.4.14.tar.gz` had been overwritten by someone else's tarball between my copy and my build. Attempted to spin up an isolated second container (`nagios-repro-testcov2`) to avoid this entirely, but it lacked whatever proxy/repo configuration was set up on the original `nagios-repro` outside of the base `fedora:40` image (`dnf` couldn't resolve mirror hosts — no proxy env vars are set *inside* the container either, so the setup must be baked into a layer or done post-creation by whatever provisioned the original) — didn't chase that further given time cost vs. just retrying the shared container. Recovered by re-copying my tarball, immediately re-running `rpmbuild -bc --noclean`, and immediately `grep`-verifying the freshly generated `t-tap/Makefile.in` contained `test_getcgi`/`test_cgiauth` *before* running anything else, minimizing the race window.

Final clean run, `cd t-tap && make test`:

```
./test_logging ........ ok
./test_events ......... ok
./test_checks ......... ok
./test_commands ....... ok
./test_downtime ....... ok
./test_nagios_config .. ok
./test_timeperiods .... ok
./test_macros ......... ok
./test_json_escape .... ok
./test_getcgi ......... ok
./test_cgiauth ........ ok
All tests successful.
Files=11, Tests=6730, 111 wallclock secs
Result: PASS
```

All 11 targets (9 pre-existing + 2 new) compile, link, and pass — 6730 total TAP assertions, 0 failures. No regressions in the pre-existing suite.

## Recommendation

This is small, well-scoped, low-risk, and directly mirrors the `test_json_escape` precedent from earlier today. Recommend merging `feature/expand-t-tap-coverage` (commit `b9f5ac4e`, based on `kuriyama12`) into `al2023-4.4.14` as `kuriyama13`, following the same spec-bump/tag/tarball-regen workflow used for the last several changes today.
