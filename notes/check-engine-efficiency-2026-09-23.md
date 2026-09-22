# Check engine efficiency: current-state analysis

Scope: `nagioscore` @ `al2023-4.4.14` (tag `kuriyama12`). Analysis-only, no
code changes. Framing: single large-instance deployment ("結構ホスト、
サービス入れてる"), not distributed/clustered — so anything that only
matters for multi-satellite setups is out of scope, and anything O(N) in
total object count is worth flagging even if each individual operation is
cheap.

Overall picture, stated up front: **this codebase's core scheduling/
notification hot path is already well-optimized** — proper binary heap for
the event queue, skiplist-indexed name lookups, and object graphs that are
pointer-linked once at config-load time rather than re-searched by name on
every check/notification. That's not what I expected going in (Nagios has
a reputation for this being weak), and it's worth saying plainly so effort
isn't wasted re-solving already-solved problems. The one clearly
real, still-synchronous, O(N)-in-total-objects cost on the single-threaded
main loop is the periodic full `status.dat` rewrite (#1 below). Everything
else is either a config-tuning opportunity (#2) or a minor/theoretical
concern not worth code changes at realistic scale.

---

## 1. `status.dat` full rewrite blocks the entire single-threaded core, every `status_update_interval` seconds — HIGHEST IMPACT

**Where:** `base/events.c:1295-1301` (`EVENT_STATUS_SAVE` handler) →
`common/statusdata.c:66-81` (`update_all_status_data`) →
`xdata/xsddefault.c:115-`~`600` (`xsddefault_save_status_data`).

**What happens:** `base/events.c:584` schedules `EVENT_STATUS_SAVE` as a
*recurring* event at `status_update_interval` (default **10 seconds** —
`sample-config/nagios.cfg.in:114`). Nagios is single-threaded and
single-process for its main event loop (`base/nagios.c:846` →
`event_execution_loop()` in `base/events.c:1063`); there is no fork, no
thread, no async I/O around this event (`base/events.c:1295-1301` calls
`update_all_status_data()` directly, inline, in the loop). While it runs,
**no other event in the queue is processed** — no check dispatch, no check
result reaping, no external command processing, no notification dispatch.

`xsddefault_save_status_data()` (`xdata/xsddefault.c:115`) does a full
`mkstemp` + buffered-stdio write of the *entire* current object state:
one `hoststatus { ... }` block per host (~58 `fprintf` calls each,
`xdata/xsddefault.c:230-293`) and one `servicestatus { ... }` block per
service (similar field count, `:297-` onward), looping linearly over
`host_list`/`service_list`. This is O(N) in total hosts+services — not
quadratic — but the constant factor is real: dozens of separate `fprintf`
calls per object, each of which parses its format string at call time
(no precompiled/binary format), plus custom-variable sub-loops per host/
service (`:288-292`).

**Why this matters for this deployment specifically:** with "結構" many
hosts/services, this is potentially tens of thousands of `fprintf` calls
every 10 seconds, all on the single thread that's also responsible for
dispatching checks and reaping results. The practical symptom, if this is
actually biting, would be periodic micro-stalls in check throughput /
notification latency that correlate with `status_update_interval`
boundaries — worth confirming empirically before assuming it's currently a
problem (see "how to measure" below), since the codebase doesn't make it
obvious this write is synchronous unless you read the source.

**Effort / risk to improve, ranked cheapest first:**
- **S, zero code, zero upstream divergence:** raise `status_update_interval`
  in `nagios.cfg` (e.g. 10 → 30-60s). Directly trades "how stale can
  status.dat/CGI-visible state be" for "how often does the core stall."
  Only affects consumers that read `status.dat` (the CGIs / our own
  `xsddefault`-backed status reads) — live state is unaffected since
  individual state changes go through NEB broker callbacks
  (`common/statusdata.c:92-129`), not this file, so notifications/checks/
  alerting are NOT delayed by a larger interval — only the *displayed*
  staleness in status.dat-reading tools is. This is the obvious first
  thing to try and needs no code at all.
- **M, local patch, low upstream-divergence risk:** batch each object's
  fields into a stack/heap buffer and do one `fwrite()` per object instead
  of ~58 `fprintf()` calls, or at minimum `setvbuf()` the stream with a
  larger buffer than stdio's default (~4KB) to cut the number of underlying
  `write()` syscalls. Mechanical, testable in isolation, low risk — a
  reasonable candidate to actually implement and could plausibly be
  upstreamed too since it doesn't change output format.
  - **L, real architecture change, high divergence risk:** avoid the
  periodic full-file rewrite model entirely (e.g. an NEB-module-based live
  status export, or writing only *changed* objects). This is a genuine
  redesign of a file format other tools (this project's own CGI included)
  depend on — not something to take on lightly, flag as a "someday, big"
  item rather than a near-term task.

**How to measure on the real deployment** (since this sandbox has no
running Nagios instance to profile): the codebase already has a
`timing_point()` instrumentation hook used throughout `xdata/xodtemplate.c`
config parsing (gated on `test_scheduling`/`-v`-style flags — see
`base/nagios.c` for how `test_scheduling` gets set) — that's for startup,
not this steady-state path, but it shows there's already a project
convention for this kind of instrumentation to imitate. For the status.dat
write specifically, the simplest empirical check: `strace -T -p <nagios
pid>` briefly, or just compare `date +%s.%N` before/after by temporarily
adding one `logit()` timing line around the `update_all_status_data()` call
— trivial, low-risk, and would turn this from "plausible cost model" into
a real number for the actual config size in production.

---

## 2. Check concurrency: existing knobs are probably under-provisioned by default, not a code problem

**Where:** `base/workers.c:1061-1087` (`get_desired_workers`),
`base/config.c:189-191` (`check_workers` directive),
`base/config.c:780-784` (`max_concurrent_checks`),
`base/utils.c:464-` (`loadctl` adaptive throttling, `set_loadctl_options`).

The worker-process pool that actually executes plugins defaults to
`cpus * 1.5`, floor 4, cap 48 (`base/workers.c:1072-1082`) when
`check_workers` isn't set in `nagios.cfg`. On a modest-core single server
this can easily be the binding constraint on check throughput before
anything else in the engine is. There's also an adaptive load-control
system already built in (`loadctl`, driven by `loadctl_options` in
`nagios.cfg`, load-average-based ramp-up/back-off of concurrent jobs) that
may already be active and self-tuning — worth checking its current
settings/state (`nagiostats` reports loadctl state) before assuming manual
tuning is even needed.

**Recommendation:** this is 100% a config-tuning exercise, not a code
change — check the current `check_workers`/`max_concurrent_checks`/
`loadctl_options` values against actual CPU count and current check
volume/latency (`nagiostats` gives queue depth and latency numbers
directly) before writing any code here. **Effort: S (config only).**

---

## 3. Event scheduling queue (`lib/squeue.c`) — already efficient, not a target

Backed by `prqueue` (binary heap), confirmed by reading the actual
implementation (`lib/squeue.c:1-14` docstring is accurate): `peek()` O(1),
`add()`/`pop()`/`remove()` O(log n) (`lib/squeue.c:105-217`). No linear
scans anywhere in the insert/pop/reschedule path. This is correct and
appropriately efficient for thousands of scheduled events; nothing to do
here.

---

## 4. Dependency checking on check-result completion — already efficient, not a target

`check_service_dependencies()`/`check_host_dependencies()`
(`base/checks.c:1946`, `:2690`) walk `svc->exec_deps`/`svc->notify_deps` —
**pre-linked, per-object dependency lists** built once at config-load/
object-linking time, not a scan over the global dependency list or a
name-based lookup. Cost per check completion is O(depth of that service's
own dependency chain), which is what you'd want. Nothing to do here.

---

## 5. `xdata/xodtemplate.c` config parsing/template resolution — already efficient, contrary to Nagios's historical reputation

This is the part I expected to find the classic "Nagios reload is slow for
big configs" quadratic behavior in, based on general reputation — the
actual code doesn't support that expectation:

- **Object duplication** (hostgroup/servicegroup → per-host/per-service
  expansion, `xodtemplate_duplicate_services`
  `xdata/xodtemplate.c:4067`, `xodtemplate_duplicate_objects:4371`) uses
  `bitmap_set`/`bitmap_isset` (`:4078`, `:4145-4154`) for O(1)
  already-seen/rejected checks instead of nested linear scans. This looks
  like the product of a real past optimization pass (plausibly the
  Nagios 3→4 duplication-logic rewrite this codebase inherited) — it is
  not something a naive implementation would produce by accident.
- **Name-based object lookups** (`xodtemplate_find_real_host` and siblings,
  `xdata/xodtemplate.c:6928-6950` and throughout `:5085-6250`) go through
  `skiplist_find_first()` against pre-built skiplists
  (`xobject_skiplists[...]`, populated during parsing at
  `xdata/xodtemplate.c:1278-1650`), i.e. O(log n), not linear search —
  including the per-service host lookup in
  `xodtemplate_inherit_object_properties` (`:4715`) that would otherwise
  have been the classic O(services × hosts) trap.

**Conclusion:** config parse/reload time scaling with object count is
already handled reasonably here. If reload latency is actually a pain
point in practice, profile the real config first (again, the
`timing_point()` calls already sprinkled through this file are meant for
exactly that — check what output they produce with the right debug/verbose
flags on an actual reload) rather than assuming this file is the culprit.
**No code changes recommended here.**

---

## 6. Notification fan-out — already efficient, not a target

`service_notification()`/`host_notification()`
(`base/notifications.c:70`, `:1046`) iterate the *service's/host's own*
pre-resolved `contacts`/`contact_groups` linked-list members
(`:1006-1017`, `:1908-1921`), populated once at object-linking time — not
a scan of the global contact list per notification. Fan-out cost is
proportional to the actual number of recipients for that one host/service,
which is correct. Nothing to do here.

---

## 7. Logging hot path — minor, not worth code changes at realistic scale

`write_to_log()` (`base/logging.c:181-215`) keeps the log file descriptor
open across calls (`open_log_file()`, `base/logging.c:113-150`, only
reopens on rotation) — no per-call open/close cost, that part's fine. It
does call `fflush(fp)` on every single log line (`:210`), meaning one
`write()` syscall per logged event (not `fsync`, so no disk-durability
sync cost, just no batching across lines). This is a deliberate and
reasonable tradeoff for live-tailable logs and crash-safety of recent
history, not a bug. At realistic single-server log volumes this is a
microseconds-per-event cost, several orders of magnitude below the
status.dat rewrite in #1. **Not worth touching.**

---

## Swept but not covered in depth (noted for completeness)

- `base/workers.c` / `lib/iobroker.c` IPC mechanics beyond worker-count
  sizing (job dispatch protocol, kvvec-based message framing) — read
  enough to confirm the worker pool itself isn't obviously bottlenecked
  structurally, but did not do a deep line-by-line audit of the IPC
  read/write loop itself. If check latency/throughput profiling (per #2)
  points here after config tuning is ruled out, this deserves a deeper
  look next.
- `base/checks.c`'s check *initiation* path (scheduling → actually
  forking/dispatching to a worker) was read at the dependency-check level
  but not exhaustively past that; no obvious per-check O(N) scans were
  spotted in what was read.
