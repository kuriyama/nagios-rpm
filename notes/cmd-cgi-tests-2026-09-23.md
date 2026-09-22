# t-tap tests for cgi/cmd.c (2026-09-23)

Branch: `feature/cmd-cgi-tests` (worktree `/claude/nagios-core/nagioscore-wt-cmd-tests`,
based on `al2023-4.4.14` @ `933c766b` / kuriyama15)
Commit: `82a7e976` — "Add t-tap regression tests for cmd.cgi's command-pipe defenses"

## Why this, why scoped this way

Two earlier reports both pointed here:
- `test-coverage-2026-09-23.md` named `cgi/cmd.c` the highest-value untested target
  from a security standpoint (POST body → external command pipe), but flagged it as
  "the most entangled with the full CGI/auth/object-graph stack."
- `security-review-2026-09-23.md` did a careful manual trace of the command-pipe
  injection defenses and concluded they're currently correct — but only as prose.
  Nothing would catch a future regression.

Confirmed the entanglement claim directly: compiling `cgi/cmd.c` standalone and
listing its undefined symbols shows ~40 non-libc dependencies (`find_host`,
`find_service`, `is_authorized_for_*_commands`, CGI config loading, object-data
reading, ...). Testing `commit_command()`/`commit_command_data()` (the actual
command-type dispatch) properly would need real host/service/contact fixtures,
not just stubs — the same call `test_cgiauth.c` made for `cgiauth.c`'s
object-graph-dependent half. Not attempted here; see Follow-up below.

Instead, scoped to the two small, pure, security-relevant functions that need
no object graph at all:

- **`write_command_to_file()`**: the single, global, last-mile defense against
  command-pipe injection via an embedded newline
  (`if(!cmd || !*cmd || strchr(cmd, '\n')) return ERROR;`). This is what actually
  blocks injection regardless of which field carries the newline — the thing
  most worth pinning down, since a future refactor could plausibly "simplify"
  it away without realizing what it's protecting.
- **`clean_comment_data()`**: strips `;` from `comment_data`/`comment_author`
  before they reach command construction (`commit_command()` rejects a literal
  `;` in every other field, but relies on this function for these two).

## Implementation notes

- `cgi/cmd.c` is `NSCGI` code; t-tap's shared `CFLAGS` targets `NSCORE`. Under
  plain `NSCORE`, `downtime.h` (one of `cmd.c`'s own includes) pulls in
  `nagios.h`, whose `command_file`/`read_main_config_file` declarations
  conflict with the CGI-side ones in `cgiutils.h`. Fixed with a dedicated
  `test_cmd.o` build rule that adds `-DNSCGI`.
- `cgi/cmd.c` is a full CGI program with its own `main()`. Since nothing in this
  test calls it, `cmd-fortest.o` (a separate object, not the one the real
  `cmd.cgi` build produces) is compiled with `-Dmain=cmd_main_unused` to rename
  it out of the way at compile time — a build-flag trick, zero source changes.
- New `t-tap/stub_cmd_deps.c` supplies the ~30 remaining stub symbols/globals
  `cmd.o` references but this test doesn't exercise (reusing the existing
  `stub_objects.c`/`stub_downtime.c`/`stub_comments.c` for the object-lookup
  functions they already cover). Note: `cmd.c`'s `free_memory(void)` (CGI-side,
  from `cgiutils.h`) is a different function from `base/utils.c`'s
  `free_memory(nagios_macros *)` that `t-tap/stub_utils.c` stubs — don't
  `#include` that file here, it stubs the wrong one and won't compile under
  `-DNSCGI` (`nagios_macros` isn't in scope).
- `include/getcgi.h` has no include guard — `stub_cmd_deps.c` relies on
  `test_cmd.c`'s own includes rather than re-including headers itself, since a
  second inclusion in the same translation unit is a hard error.

## What's covered / test content (17 assertions)

- `write_command_to_file()`: NULL, empty string, trailing/embedded/leading/bare
  newline all rejected; a well-formed command with no newline is accepted AND
  written byte-for-byte (verified against a real temp file, not just a return
  code) with the trailing newline the function itself appends.
- `clean_comment_data()`: no-op on text with no `;`; every `;` replaced with a
  space, including consecutive ones; a full command-shaped string has all `;`
  removed; newlines are explicitly NOT touched (documents the division of
  labor with `write_command_to_file()`); NULL and empty string don't crash;
  length never changes.

## Follow-up (not done here)

- `commit_command()`/`commit_command_data()`'s field-level validation (the `;`
  rejection in `host_name`/`service_desc`/`hostgroup_name`/`servicegroup_name`/
  `comment_author`, the `MAX_INPUT_BUFFER` length guards on
  `plugin_output`/`performance_data`) — needs real object-graph fixtures or a
  purpose-built fixture-friendly stub layer, same as `test_cgiauth.c`'s deferred
  object-graph half.
- The ~30-symbol stub file (`stub_cmd_deps.c`) is disposable scaffolding for
  this narrow test; if the object-graph-dependent half is tackled later, expect
  to extend rather than fully reuse it (most stubs there return `FALSE`/`NULL`
  deliberately, to keep this test from accidentally exercising untested paths).

## Verification

fedora:40 `nagios-repro` container, real rpmbuild-generated CFLAGS (LTO, PIE,
stack protector, `-DHAVE_CONFIG_H` etc.) via a fresh `rpmbuild -bc --noclean`
extraction. `cd t-tap && make test`:

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
./test_cmd ............ ok
All tests successful.
Files=12, Tests=6744, Result: PASS
```

All 12 targets (11 pre-existing + `test_cmd`) pass, 6744 total assertions, 0
failures, no regressions.
