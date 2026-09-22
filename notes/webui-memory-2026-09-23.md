# webui browser memory investigation (2026-09-23)

## TL;DR

No leak found, at 2000 or 5000 synthetic hosts. DOM node count is exactly
identical before/after 40 navigation cycles at both scales; JS heap grows
by a couple MB total (consistent with GC noise / in-flight JSON payloads,
not unbounded growth) and then stops growing. The code that matters for
this (`main.ts`'s `activeViewCleanup`, `hosts.ts`/`services.ts`'s
interval teardown, `modal.ts`'s `document`-level keydown listener
removal) is correctly paired everywhere it was checked.

Per the fallback plan, added lightweight always-available instrumentation
(`window.__nagiosDebug()` / `?debug=1`) on branch
`feature/webui-memory-instrumentation`, commit `fdd4f611`, in a separate
worktree at `/claude/nagios-core/nagioscore-wt-webui-memory` — **not**
merged into `al2023-4.4.14`, left for review.

Do not add virtual scrolling on the strength of this investigation — the
numbers here don't contradict the user's existing "current counts are
fine, I like Ctrl-F" position. See "When would this change" below for
the one thing that might.

## Static audit

Read all of `webui/src/*.ts` (2172 lines total). Findings:

- **`main.ts`**: `renderRoute()` calls the previous view's cleanup
  function (`activeViewCleanup()`) before mounting a new one, and this is
  wired for every hash change via a single `hashchange` listener
  installed once in `main()`. No listener accumulation on repeated
  navigation.
- **`hosts.ts` / `services.ts`**: identical structure. `renderTableBody()`
  does `tbody.innerHTML = ''` before rebuilding rows every time (initial
  load, sort click, filter change, and refresh), so old `<tr>` elements
  (and their per-row `actionLink` click listeners) are fully detached and
  eligible for GC on every render — nothing accumulates. Header-cell sort
  listeners are attached exactly once (only inside the `if (initial)`
  block), not on every re-render. The `setInterval` auto-refresh timer id
  is captured in a closure and cleared by the function returned from
  `renderHosts`/`renderServices`, which `main.ts` invokes before mounting
  any other view; a `stopped` flag additionally guards against a
  still-in-flight fetch rendering into a torn-down view. This is the
  correct pattern and it's applied consistently in both files.
- **`modal.ts`**: the one place a listener is attached to `document`
  rather than to a to-be-discarded element (`document.addEventListener
  ('keydown', onKeydown)`, for closing the modal on Escape) — this is
  exactly the shape of bug that leaks one global listener per dialog if
  not paired. It is paired: `close()` calls `document.removeEventListener
  ('keydown', onKeydown)` before removing the overlay, on every exit path
  (Cancel button, overlay-click-outside, Escape itself, and successful
  submit all funnel through `close()`).
- **`dashboard.ts`**: search debouncing uses a single `setTimeout` handle
  cleared via `clearTimeout` on each keystroke (standard debounce, self-
  cleaning, no accumulation). No `setInterval` here, and `renderDashboard`
  doesn't return a cleanup function, but there's nothing to clean up —
  a stray in-flight 200ms debounce timeout firing after navigating away
  just no-ops against a still-alive-but-unmounted DOM subtree; negligible.
- **`commands.ts` / `actions.ts`**: no listeners, no timers, nothing
  retained across calls; `submitCommand()` does a single fetch per action.

No code changes were needed as a result of the static audit — it was
clean going in.

## Empirical measurement

No live Nagios backend exists in this sandbox, so I built a synthetic
harness instead of relying on static analysis alone:

- `webui/` built normally (`npm install && npm run build`).
- A small Node mock server (`server.mjs`, no dependencies) serves the
  built `html/` statically and answers `statusjson.cgi`/`objectjson.cgi`
  with synthetic data shaped exactly like the real JSON envelope (checked
  against `webui/src/api.ts`'s types) — configurable host/service counts
  via env vars.
- `google-chrome --headless=new --remote-debugging-port=...` plus a
  ~150-line hand-rolled Chrome DevTools Protocol client (`measure.mjs`,
  no npm dependencies — Node 24's built-in `WebSocket`/`fetch` were
  enough; `npm install puppeteer-core` was not attempted since the raw
  CDP approach worked fine and avoids a network/registry dependency).
  Heap is read via `Performance.getMetrics`'s `JSHeapUsedSize` after
  forcing `HeapProfiler.collectGarbage`, so numbers reflect *retained*
  memory, not just transient allocation.

### Method

For each scale, in one browser tab:
1. Navigate to the page, read baseline heap.
2. Navigate to `#hosts`, read heap + `document.querySelectorAll('*').length`.
3. Loop 40x: hash to `#services`, wait 120ms, hash to `#hosts`, wait
   120ms (exercises full unmount/remount + interval teardown/setup on
   every iteration — a reasonable proxy for "many refresh cycles over a
   long session", since a refresh and a remount both do
   `tbody.innerHTML=''` + full repopulate).
4. Read heap + DOM node count again.
5. Loop 30x: click the first "Downtime" action link, click "Cancel" in
   the resulting modal (this exercises `modal.ts`'s open/close and the
   `document` keydown listener add/remove specifically). Count leftover
   `.modalOverlay` elements afterward (should be 0).
6. Read heap again.

### Results

| Scale | after #hosts load | after 40 nav cycles | after 30 modal cycles | DOM nodes (constant across all 3) |
|---|---|---|---|---|
| 2000 hosts × 6 services | 1.75 MB | 1.95 MB | 2.00 MB | 26209 |
| 5000 hosts × 8 services | 2.75 MB | 3.37 MB | 3.42 MB | 65346 |

DOM node count is **bit-for-bit identical** before and after the 40
navigation cycles at both scales (26209→26209, 65346→65346) — if
`tbody.innerHTML=''` weren't clearing old rows, or if the header
listeners were being re-added per render, this number would grow
linearly with cycle count; it doesn't. Heap grows by under 1MB total
across the whole sequence at either scale, which is well within normal
GC/allocation noise for ~80 fetch+render cycles each moving hundreds of
KB of JSON. Zero stray `.modalOverlay` elements after 30 modal
open/Cancel cycles, and zero eval errors during the modal loop.

Reproduce with:
```
cd webui && npm install && npm run build
node server.mjs                    # MOCK_HOSTS/MOCK_SVC_PER_HOST/MOCK_PORT env vars
google-chrome --headless=new --remote-debugging-port=9333 --disable-gpu --no-sandbox --disable-dev-shm-usage about:blank &
node measure.mjs                   # CDP_BASE/SITE_URL env vars
```
(`server.mjs`/`measure.mjs`/`smoketest.mjs` are in this session's
scratchpad, not committed — happy to move them into the repo, e.g. under
`webui/`, as a permanent load-test harness if that's useful going
forward; ask if so.)

## What this does and doesn't tell you

This confirms the *architecture* doesn't leak under repeated
interaction at a fixed data size, at up to 5000 hosts / 40000 service
rows. It does **not** measure:
- Real Chrome on a real user's machine over a real multi-hour/day
  session with the actual auto-refresh interval (90s) rather than
  synthetic rapid-fire navigation — the instrumentation added below is
  meant to close that gap cheaply.
- Initial-render *time* at large scale in a way that matters practically
  (5000 hosts took noticeably longer to finish 40 render cycles than
  2000 did in headless Chrome — this wasn't rigorously timed since it
  wasn't the ask, but if reload/refresh latency at your actual host
  count feels sluggish, that's a real, separate, measurable thing worth
  a follow-up look, distinct from "memory leak").
- Memory in non-Chrome browsers (`performance.memory` is Chrome-only;
  the added instrumentation degrades gracefully to `null` elsewhere, but
  can't tell you anything there).

## When would this change the "no virtual scrolling" calculus

The user's existing position (Ctrl-F search over the full page is more
valuable than the performance win, and current counts are fine) is not
contradicted by anything found here. It would be worth revisiting if:
real deployment data shows either (a) `?debug=1`'s `scaffoldMs` or
subjective load feel gets bad at your actual host/service count, or (b)
you're routinely at a scale meaningfully past the 5000-host / 40000-row
tier tested here. Neither was observed — this is a "keep an eye on it"
note, not a recommendation.

## Instrumentation added

Branch `feature/webui-memory-instrumentation`, commit `fdd4f611`, in an
isolated worktree at `/claude/nagios-core/nagioscore-wt-webui-memory`
(created via `git worktree add`, not merged into `al2023-4.4.14`, review
and merge at your discretion). `npm run typecheck && npm run build` both
pass; built output grew from 32.3kb to 32.9kb.

- New `webui/src/debug.ts`: `snapshot()` reads `performance.memory`
  (Chrome-only, `null` elsewhere), current DOM node count, and the
  current route hash.
- `window.__nagiosDebug()` — call from the browser console any time,
  returns and `console.table()`s a snapshot. No default UI, does nothing
  unless invoked.
- Load the page with `?debug=1` in the URL to also `console.log()` a
  snapshot (plus a rough render duration — see caveat below) on every
  route change, i.e. every hash navigation. This is the cheap way to get
  a real trend line from an actual browser tab left open against actual
  production data for hours: if the numbers climb steadily instead of
  settling, that's a real leak to come back and find.
- Caveat: the logged `scaffoldMs` only times `renderHosts`/
  `renderServices`'s *synchronous* DOM-scaffold construction (they kick
  off `load(true)` without awaiting it), not the full time until data is
  visible. Getting a true "time to populated table" number would need
  `renderHosts`/`renderServices` to expose a promise that resolves after
  their first successful `load()`, which felt more invasive than the
  brief called for ("lightweight... must not affect the page for normal
  users"); flagging this as a known gap rather than silently overclaiming
  accuracy. Easy follow-up if it turns out to matter.

## Files

- Report: this file.
- Instrumentation: `feature/webui-memory-instrumentation` branch,
  worktree at `/claude/nagios-core/nagioscore-wt-webui-memory`.
- Measurement harness (not committed, scratchpad only): mock server +
  CDP driver scripts, methodology described above in enough detail to
  reproduce from scratch if the scratchpad is gone by the time this is
  read.
