# Trends/histogram SPA port (2026-09-23)

Branch: `feature/trends-histogram-spa` (worktree `/claude/nagios-core/nagioscore-wt-trends-histogram`, based on `al2023-4.4.14` @ `142cb6fc`)
Commit: see below.

Per modernization-survey-2026-09-23.md item 1 ("DO IT", highest value-for-effort item).

## Scope: trends only, not histogram

Ported `trends.html`/`trends-form.js`/`trends-graph.js` (AngularJS 1.3.9 + D3 +
Bootstrap 3.3.7) onto a new `webui/src/trends.ts` in the TS SPA. **Histogram was
not attempted** -- reading both originals first made clear that doing both
properly (each has its own report-type/parameter set and its own D3
visualization shape -- histogram is a distribution/bar-count view, not a
timeline) would take meaningfully longer than one focused pass allows for.
Shipping trends well and documenting histogram as deferred, per the
directive's explicit permission to scope down honestly rather than do both
shallowly.

## Correction to the survey's framing

The survey described the vendor stack as ".gitignore'd... not ours to
maintain." That's only half true: the *unpacked* directories
(`html/d3/`, `html/bootstrap-3.3.7/`, `html/js/trends.js`,
`html/js/histogram.js`) are gitignored, but their **sources are committed to
this repo** (`html/angularjs/angular-1.3.9.zip`, `html/angularjs/ui-utils-0.2.3.zip`,
`html/angularjs/ui-bootstrap-tpls-0.14.3.min.js`, `html/bootstrap-3.3.7-dist.zip`,
`html/d3-3.5.17.zip`) and unpacked by `html/Makefile.in` at build time. This
fork is directly responsible for shipping this EOL stack, not just linking to
something upstream manages elsewhere -- if anything this makes the case for
porting *stronger* (real maintenance burden we own), not weaker.

**Consequence for removal**: since `histogram.html` still depends on this same
shared vendor stack and was not ported, **none of the vendor files were
removed** -- doing so would break histogram, which is still the primary path
for that report. This is a hard blocker, not just caution: the vendor removal
described in the survey can only happen once histogram is also ported.

## What was built

- `webui/src/api.ts`: added `archivejson.cgi` client functions
  (`fetchStateChangeList`, `fetchAvailability`) and types (`StateChangeEntry`,
  `HostAvailability`, `ServiceAvailability`, etc.), plus `trendsPngUrl()` for
  the export link. Field shapes and CGI parameter names confirmed directly
  against `cgi/archivejson.c` (`json_archive_statechangelist`,
  `json_archive_host_availability`/`json_archive_service_availability`,
  `svm_au_states`/`valid_object_types`/`valid_availability_object_types` in
  `cgi/archiveutils.c`), not guessed from the Angular code alone.
- `webui/src/trends.ts` (new): host/service picker (datalist-based
  autocomplete off `fetchHostNames`/`fetchServices`, already used elsewhere in
  `webui/`), a time-range picker, hard/soft state toggle, backtracked-archives
  setting, a hand-rolled inline-SVG horizontal timeline (colored segments per
  state-change interval, native `<title>` tooltips, a simple 5-tick axis), a
  legend, an availability breakdown table (time + percent per state), and a
  link to the original `trends.cgi` PNG export (untouched C/gd rendering,
  kept for embedding/printing per the survey's recommendation).
- `webui/src/main.ts`: added `#trends` route.
- `webui/src/nav.ts`: "Trends" link now points at `#trends` instead of the
  static `trends.html`; the `(Legacy)` link to `trends.cgi` (image export) is
  unchanged.
- `html/stylesheets/nagios-app.css`: minimal styling for the legend/form.

## Deliberate simplifications vs. the original (see file-level comment in trends.ts too)

- **No zoom/pan.** The original used `d3.behavior.zoom` on the timeline. The
  time-range picker (presets + custom start/end) already lets you narrow the
  window and regenerate, which covers the same need without a new
  dependency or meaningfully more code.
- **No custom popup box.** Native SVG `<title>` tooltips on each timeline
  segment (state, duration, plugin output) cover "what was this and for how
  long" with zero JS event wiring, at the cost of a less polished hover UI.
- **A fixed set of time-range presets** (Last 4 Hours / 24 Hours / 7 Days /
  31 Days / This Month / Last Month / Custom) rather than the original's
  full canned-timeperiod list (today/yesterday/this week/last week/this
  year/last year/...). Covers the common cases; not a 1:1 port of
  `nagiosTimeService`'s calculations.
- **No D3 at all** -- the timeline is hand-rolled inline SVG (a linear time
  scale computed by hand, `<rect>` per segment). This was the single
  question flagged by the survey as worth checking before assuming "no D3":
  having read `trends-graph.js` in full, the actual visualization is a
  single-row horizontal bar timeline, well within what plain SVG handles
  cleanly -- no charting primitive was needed.

Everything else (host/service selection, hard/soft states, backtrack
setting, the state-change timeline itself, the availability breakdown, and
the PNG export link) has real functional parity with the original.

## Verification

- `cd webui && npm run typecheck && npm run build`: both clean.
  `html/js/nagios-app.js` grew from ~33kb to 42.9kb.
- **Empirically verified rendering**, not just type-checked: built a local
  mock server (serving the built `html/` output plus synthetic
  `archivejson.cgi`/`objectjson.cgi`/`statusjson.cgi` responses shaped to
  match the real API) and drove headless Chrome via the Chrome DevTools
  Protocol (same technique the browser-memory track used earlier this
  session: `google-chrome --headless=new --remote-debugging-port=...` + a
  hand-rolled CDP client, no npm dependency). Confirmed, for both report
  types:
  - **Host report**: form fill + submit works; timeline renders 12 SVG
    `<rect>` segments from synthetic state-change data; availability table
    renders 3 populated rows (Up/Down/Indeterminate, matching the mock's
    nonzero fields) with percentages; legend shows 4 host states; PNG
    export link correctly built
    (`trends.cgi?host=testhost&t1=...&t2=...`); zero console
    errors/exceptions.
  - **Service report**: switching the report-type select correctly toggles
    the service picker's visibility; timeline renders 12 segments; legend
    shows OK/Warning/Unknown/Critical/Indeterminate; availability table
    shows 5 rows with percentages summing to ~100%
    (87.98+5.87+2.93+1.47+1.76); PNG export link correctly includes
    `service=testsvc`.

This is real evidence the view actually works end-to-end against
API-shaped data, not just "compiles."

## Not verified / follow-up

- No live Nagios backend exists in this sandbox, so this is verified against
  synthetic data shaped to match the real JSON API, not the real CGI
  responses themselves (same caveat the browser-memory track noted for its
  own testing).
- Histogram is not ported. Doing so would follow the same pattern
  (`webui/src/histogram.ts`, reusing `fetchAvailability`/similar archivejson
  queries) but needs its own read of `histogram-form.js`/`histogram-graph.js`
  /`histogram-events.js` (446+184+85 lines) to get its distinct
  report-type/visualization shape right -- not attempted here.
- Only after histogram is ported can the vendor blobs
  (`html/angularjs/`, `html/bootstrap-3.3.7-dist.zip`, `html/d3-3.5.17.zip`,
  and their unpacked `.gitignore`'d directories) actually be removed, along
  with `html/Makefile.in`'s unzip/install rules for them.
- The real-user visual polish (this is functional but plain -- no styling
  effort beyond matching the existing SPA's table/form conventions) wasn't a
  goal of this pass; worth a look if this becomes the primary path.

## Files changed

`webui/src/api.ts`, `webui/src/main.ts`, `webui/src/nav.ts`,
`webui/src/trends.ts` (new), `html/stylesheets/nagios-app.css`,
`html/js/nagios-app.js` (built output).
