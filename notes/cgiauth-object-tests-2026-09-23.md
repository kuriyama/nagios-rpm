# t-tap tests for cgiauth.c's object-graph authorization functions (2026-09-23)

Branch: `feature/cgiauth-object-tests` (worktree `/claude/nagios-core/nagioscore-wt-cgiauth-objects`,
based on `al2023-4.4.14` @ `82a7e976` / kuriyama16)
Commit: `7f55dcc1`

## Why this, why now

Both `test-coverage-2026-09-23.md` and `cmd-cgi-tests-2026-09-23.md` named this the
highest-value remaining test-coverage gap: `cgi/cgiauth.c`'s object-graph-dependent
authorization functions (`is_authorized_for_host/service/hostgroup/servicegroup` and
their `*_commands()` siblings) gate every CGI's per-host/per-service view and command
access. `test_cgiauth.c` (added earlier this session) deliberately covered only the
`authdata`-only subset that needs no object graph, leaving this half as documented
follow-up.

## Fixture strategy

Chose (b) from the two options in the brief: link the REAL `common/objects.c` (as
`cgi/objects-cgi.o`, the same translation unit `cgi/` programs already link) and build
fixtures with its actual object-creation API (`add_host`, `add_service`, `add_contact`,
`add_contact_to_host`, `add_contactgroup_to_host`, `add_host_to_hostgroup`,
`add_service_to_servicegroup`, `add_hostescalation`, `add_contact_to_hostescalation`,
...) — the same functions the real config parser calls. This means `find_host()`,
`find_contact()`, `is_contact_for_host()`, `is_escalated_contact_for_host()`, etc. all
run as their genuine shipped implementations against a small in-memory object graph,
rather than a hand-rolled parallel stub layer that could silently drift from the real
linking semantics.

This turned out to be very tractable: `add_host()`/`add_service()`/`add_contact()`
`calloc()` their own struct and insert directly into a `dkhash`-backed table
(`create_object_tables(ocount)` sets up 12 such tables, sized generously at 16 entries
each for this test); no config-file parsing or `xodtemplate.c` involvement needed.
`add_hostescalation()` self-links into `host->escalation_list` internally, so no
separate linking pass was required for the escalation-contact scenario either.

Needed two small compile-time additions beyond the object-creation calls: `stub_logging.c`
for `logit()` (declared *before* its `#include`, since C requires `debug_level`/
`debug_verbosity`/`DEBUGL_ALL` to already be visible when the stub's body references
them), and `#include "../include/logging.h"` for the `DEBUGL_ALL` macro.

## What's covered (27 assertions)

- `is_authorized_for_host()`: direct contact, contactgroup membership, escalation-only
  contact, an unrelated contact (denied), `use_authentication == FALSE` overriding all
  of the above.
- `is_authorized_for_service()`: the "authorized for the host implies authorized for
  its services" shortcut, a direct service contact, an unrelated contact (denied).
- `is_authorized_for_hostgroup()`/`is_authorized_for_servicegroup()`: confirmed the
  ANY-member semantics (a group is viewable if the contact is authorized for even one
  member, not all) that the earlier security review flagged as a real, documented
  upstream quirk (`CHANGED in 2.0 ... Reverted for 3.3.2`).
- `is_authorized_for_host_commands()`/`is_authorized_for_service_commands()`: the
  `can_submit_commands` gate (blocks command authorization even for an otherwise-valid
  contact).
- **A real behavior found while writing the `*_all_host_commands`/
  `*_all_service_commands` test**: these flags alone do nothing. Both functions gate
  their entire body behind `is_authorized_for_host()`/`is_authorized_for_service()`
  succeeding *first* — so `authorized_for_all_host_commands` only has any effect once
  the contact is *already* granted view access by some other means (typically
  `authorized_for_all_hosts`/`authorized_for_all_services`). My first draft of this
  test assumed the flag alone was sufficient and failed against the real code; fixed
  the test (not the source — this is upstream's actual, if easy-to-misread, design) and
  added an explicit assertion pinning down the "alone is insufficient, paired with
  `authorized_for_all_hosts` it works" behavior for both the host and service variants.
  This is exactly the kind of subtlety a cgi.cfg author could get wrong (setting
  `authorized_for_all_host_commands=1` without also setting
  `authorized_for_all_hosts=1` and being surprised it does nothing) and is now pinned
  down rather than just implicit in the source.
- `is_authorized_for_hostgroup_commands()`/`is_authorized_for_servicegroup_commands()`:
  confirmed the ALL-members semantics (every member must be authorized, unlike plain
  view access's ANY-member semantics) — the exact asymmetry the security review noted
  as worth knowing but not fixing, now locked in by a test that would fail if the two
  ever silently converged.

## Deliberately not covered

- `serviceescalation`-based authorization — the `hostescalation` path already exercises
  the same underlying `is_contact_for_*_escalation()` logic; a `serviceescalation`
  fixture would be close to duplicate coverage for the effort involved.
- `hostdependency`/`servicedependency` — unrelated to authorization.

## Verification

`nagios-repro` fedora:40 udocker container. Iterated the new target alone first
(`make test_cgiauth_objects`) against an already-extracted build tree to debug two real
setup bugs quickly (see below), then did a from-scratch verification: fresh tarball via
`git archive` from the committed HEAD, fresh `SOURCES`/`SPECS`, full `rpmbuild -bc`, then
`cd t-tap && make test`:

```
./test_logging .......... ok
./test_events ........... ok
./test_checks ........... ok
./test_commands ......... ok
./test_downtime ......... ok
./test_nagios_config .... ok
./test_timeperiods ...... ok
./test_macros ........... ok
./test_json_escape ...... ok
./test_getcgi ........... ok
./test_cgiauth .......... ok
./test_cmd .............. ok
./test_cgiauth_objects .. ok
All tests successful.
Files=13, Tests=6771, Result: PASS
```

All 13 targets (12 pre-existing + `test_cgiauth_objects`) pass, 6771 total assertions,
0 failures, no regressions.

Two real bugs hit and fixed *in the test's own fixture setup* while iterating (not in
shipped source):
1. `add_service()`'s `retry_interval` parameter must be `> 0` (strictly), unlike
   `add_host()`'s equivalent which only rejects `< 0` — passing `0` made every
   `add_service()` call silently return `NULL`, causing a segfault three calls later
   when the (unchecked, in my WIP code) `NULL` was dereferenced. Found via `gdb -batch
   -ex run -ex bt` against the container binary.
2. The `authorized_for_all_*_commands` test design bug described above.
