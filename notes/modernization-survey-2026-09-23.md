# Modernization / rewrite candidate survey — nagioscore (al2023-4.4.14, kuriyama12)

Scope: architecture/design-era mismatches worth reworking, given this deployment's actual
usage. Ordered by recommended priority (highest value-for-effort first). Each item: current
state, why it's dated, proposed direction, effort, upstream-divergence classification, and
recommendation.

Stance follows the user's own principle: don't preserve what this deployment doesn't use, but
respect upstream where a change would only make sense as a private fork.

---

## 1. Port `trends.html`/`histogram.html` off AngularJS 1.3.9 + D3 + Bootstrap 3.3.7 onto the TS SPA

**Recommendation: DO IT.** Best value-for-effort item in this whole survey.

**Current state.** `html/trends.html` and `html/histogram.html` (both still linked from
`webui/src/nav.ts`, with `trends.cgi`/`histogram.cgi` demoted to `legacyHref` fallbacks) load a
committed-but-vendored stack: AngularJS 1.3.9 (EOL since ~2016, zero security patches since),
D3, and Bootstrap 3.3.7 — all pulled from `.gitignore`'d directories (`html/angularjs/`,
`html/d3/`, `html/bootstrap-3.3.7/`, `html/js/trends.js`, `html/js/histogram.js`), i.e. they
come from upstream's own `make-tarball`/dist pipeline, not our repo. The only *committed*
custom code is `html/js/trends-form.js` and `html/js/trends-graph.js` (small Angular
controllers).

**Why this is the good kind of yak to shave.** Nagios upstream already did half the
modernization for us, years ago: `trends.cgi`/`histogram.cgi`'s C code now only emits
`Content-Type: image/png` graph images (`cgi/trends.c:1264`, `cgi/histogram.c:1057`) plus a
legacy `text/html` fallback branch (`cgi/trends.c:1217`, `cgi/histogram.c:1014`) that's very
likely dead code now that `nav.ts` routes to the static `.html` page by default. All the actual
*data* — host/service lists, historical event data — already flows through
`objectjson.cgi`/`archivejson.cgi` (`trends-form.js:122-138`, `trends-graph.js:343,476`), the
exact same JSON API our own `webui/` already consumes and that we just hardened
(`json_escape.c`). This is not "rewrite a CGI's C logic" — it's "delete a 10-year-old,
unmaintained, `.gitignore`'d vendor blob and replace its thin JS shell with TypeScript that
calls APIs we already own and trust." The graph-image generation itself (gd-based PNG
rendering in the C CGI) stays untouched.

**Proposed direction.** Add `webui/src/trends.ts`/`histogram.ts` (or a shared `report.ts`)
implementing the same host/service picker + `archivejson.cgi` query + client-side rendering
(SVG/canvas instead of D3, or a minimal D3 replacement if the trend-bar visualization is
non-trivial — worth reading `trends-graph.js` closely before committing to "no D3 at all").
Keep `trends.cgi`/`histogram.cgi`'s PNG-image branch as-is (still useful for
export/embedding/print). Delete the vendored Angular/D3/Bootstrap pull from `make-tarball`'s
dependency list only if nothing else needs it (see item 9 — nothing else does, once this is
done).

**Effort:** M (3-5 days). The hard part is re-implementing `trends-graph.js`'s bar-chart timeline
rendering (visual state-over-time bars), not the data fetching, which is a solved pattern in
this codebase already.

**Upstream divergence:** local-fork-only (this is UI presentation, not shippable as a generic
upstream patch since upstream still ships the Angular version for everyone else). Acceptable
per the user's stated principle — this is exactly "something I use, worth improving for
myself," not a case where diverging from upstream costs us anything on the maintenance side
(we're not touching C code upstream also ships).

---

## 2. Keep retiring legacy C-HTML-rendering CGIs behind the SPA — but note the SPA has NOT superseded most of them yet

**Recommendation: CONTINUE, prioritized by what's actually linked from nav.**

**Current state — this needed correction from my initial framing.** Checking
`webui/src/nav.ts` directly: the SPA has only replaced the "all hosts" / "all services" status
table views and Ack/Downtime actions (Phases A/B/C-1/C-3 from earlier this session). Nav still
routes, unmodified, to the full legacy C-HTML CGI set for *everything else*: `tac.cgi`
(Tactical Overview), `statusmap.cgi` (Map), `status.cgi?style=summary/grid/hostdetail`
(Summary/Grid/Problems/Hostgroup views — several distinct rendering styles never touched),
`outages.cgi`, `avail.cgi`, `history.cgi`, `summary.cgi`, `notifications.cgi`, `showlog.cgi`,
`extinfo.cgi` (5 sub-views: comments/downtime/process-info/perfdata/scheduling-queue),
`config.cgi`. That's ~13 more C-CGI-rendered pages, all still actively reachable from the main
nav, all still generating raw HTML via C `printf`-style string building — the exact code shape
that produced the JSON-escaping heap overflow we fixed this session.

**Why this matters beyond "it's old code."** `git log` on this repo shows a long, real history
of exactly this bug class recurring in exactly this code: `acd2365e` "JSON Query XSS Fix",
`0b682205` "Custom Variable Injection", `02aba584`/`e5ed38e5` CSRF fixes, `01494211` "stored XSS
in html/main.php", `8deeca7c` "vulnerability in trends/histogram/maps pages", `aa64b862`/
`bdeb57a1` "Superglobal REMOTE_USER susceptible to XSS" (twice), `0329033d` CVE-2018-18245 XSS
in Alert Summary, `c399086d` CVE-2018-13441 null pointer deref, `8a288ad7` CVE-2017-12847,
`ef690014` CVE-2016-10089, `ff22fd0d` CVE-2016-9566 (root privilege escalation). This is not
speculative "old code smells bad" — it's a codebase with a demonstrated, recurring pattern of
shipping real vulnerabilities in this exact architecture, largely because C string-building
into HTML has none of the structural protection a template engine or the JSON+JS-render model
gives you for free.

**Proposed direction.** Don't do this as one big-bang rewrite (too large, too risky, matches
neither "small verified batches" workflow this session has used nor a sane review size).
Instead, prioritize by actual daily use in this deployment (ask the user which of these they
personally open regularly — my guess ordering, cheapest-and-most-used first): `tac.cgi`
(dashboard, likely opened constantly) and `status.cgi` grid/summary/hostdetail styles (declined
once already this session as Phase C-2 — worth reopening now that the JSON API is proven out
across 3 CGIs) are probably the highest-traffic pages; `extinfo.cgi`'s comments/downtime/
process-info views are used whenever anyone acks something (which the SPA already half-owns via
Phase C-3); `showlog.cgi`/`history.cgi`/`summary.cgi`/`outages.cgi`/`avail.cgi` are more
"occasionally, for a report" — lower priority; `config.cgi` (renders the live object config) is
probably rarely opened outside debugging — lowest priority, maybe not worth ever porting.
`statusmap.cgi` is a special case (see item 3).

**Effort:** L overall (each view is roughly Phase-A/C-1-sized, i.e. what this session has
already been doing incrementally — call it 2-4 days per view, ~10 views left if all are done).
Correctly scoped as "keep doing what we've been doing, in priority order," not a new kind of
project.

**Upstream divergence:** local-fork-only for the SPA side; any *bugs* found in the legacy C CGI
along the way (like the json_escape fix) should still go upstream per the user's stated
principle, independent of whether we also route around them locally.

---

## 3. `statusmap.cgi` — separate track, smaller decision

**Recommendation: LOW PRIORITY, revisit only if network-topology visualization is something
this deployment's users actually use.**

Unlike `map.php` (removed this session — an AngularJS+D3 wrapper) or `trends.html`/
`histogram.html` (item 1), `statusmap.cgi` renders its own HTML/PNG directly in C
(`cgi/statusmap.c:328,374`) with no Angular/D3 dependency — it's a self-contained legacy CGI,
architecturally in the same bucket as item 2's list rather than item 1's "delete a vendor blob"
opportunity. Whether it's worth porting depends entirely on whether anyone actually uses the
network map view for this deployment's topology (single-server monitoring setups often don't
bother with visual topology maps). Recommend asking the user before spending effort here; if
the answer is "never open it," it may belong on a future "remove" list instead of a "port" list
(same posture as map.php's removal, if truly unused — worth explicitly confirming rather than
assuming from this analysis alone).

---

## 4. NEB / Event Broker mechanism (`base/broker.c`, `base/nebmods.c`) — LEAVE ALONE

**Recommendation: SKIP. Zero local benefit, real upstream feature, don't touch.**

Confirmed via `sample-config/nagios.cfg.in:222-225`: `broker_module` is present only as a
commented-out example — no module is configured or loaded in this deployment (consistent with
this session's earlier removal of the unused `module/`/`worker/` *sample* code, which never
shipped anyway). The NEB *mechanism itself*, however, is real, actively-used-by-others upstream
infrastructure (it's how things like `--enable-event-broker`'s core hooks and third-party
monitoring integrations like `mod_gearman`, Icinga's NEB-based sync, etc. work in the broader
Nagios ecosystem) — not dead sample code like `module/helloworld.c` was. There's no local value
in trimming a mechanism you don't call, and doing so would be pure upstream divergence with
zero payoff. This is the correct category to just leave fully alone, matching the same
reasoning the user already gave for `debian/`/`startup/openrc-init.in` in this session
("あまり upstream 使ってるかもしれないものだと乖離したくない").

---

## 5. `cgiauth.c` authorization/authentication — narrower opportunity than it first looks

**Recommendation: NO ACTION on the auth *mechanism*; this was a wrong initial framing on my
part, corrected below. Only the *authorization-mapping* logic is genuinely Nagios's to own, and
it's not obviously dated.**

`cgi/cgiauth.c:33,59-64` shows identity is **already externalized**: authentication comes from
either `REMOTE_USER` (whatever the webserver's auth stack decided — htpasswd today, but
*already* pluggable to LDAP/SSO/OIDC/mod_auth_* at the Apache layer with zero Nagios code
changes) or `SSL_CLIENT_S_DN_CN` (mTLS client-cert identity). So "modernize the auth model" is
largely **already solved** at the infrastructure layer — if a more modern login flow is wanted,
it's an Apache/httpd config change, not a nagioscore code change. What `cgiauth.c` actually
does — map an already-authenticated username string to per-object permissions via `cgi.cfg`'s
`authorized_for_*` directives — is inherent business logic Nagios has to have *somewhere*
regardless of identity provider, and 605 lines of straightforward permission-checking isn't
obviously "dated" by any concrete standard (it's not persisting secrets, not doing crypto, not
parsing anything complex). One thing worth flagging to the security track rather than this one:
the historical `aa64b862`/`bdeb57a1` "REMOTE_USER susceptible to XSS" commits mean this file's
output path (not its logic) has a proven vulnerability history — worth a quick look to confirm
those fixes are actually present in this tree's current `cgiauth.c`, but that's a bug-hunt, not
a modernization item.

---

## 6. Object config format (flat `.cfg` + `xodtemplate.c` template resolution) — LEAVE ALONE

**Recommendation: SKIP. Correctly too central and too upstream-coupled to touch.**

`xdata/xodtemplate.c` is 10,598 lines — by far the largest single file in the tree — implementing
config parsing, `use`-template inheritance resolution, and object linking. This is about as
core to "being Nagios" as anything can be: every third-party config-management tool
(Ansible/Puppet/Chef modules, homegrown generators, this org's own config if any exists)
targets this exact flat-file format. A local schema/format replacement here isn't "modernize
our fork" so much as "leave the entire Nagios config ecosystem behind" — there is no proposal
at any reasonable effort level that pays for itself. If startup/reload latency from template
resolution turns out to matter in practice (see the check-engine track's analysis), the
right fix is algorithmic (faster lookups inside the existing format), not a format rewrite.

---

## 7. `status.dat`/`retention.dat` flat-file format — LEAVE ALONE (design angle; see check-engine
track for the performance angle)

**Recommendation: SKIP the format itself. This one surprised me during investigation — the
blast radius is much larger than "an external integration format."**

Initially assumed this was primarily an external-tool compatibility concern (NRPE, third-party
status parsers). It's not — it's this deployment's **own CGI layer's primary data source**:
`grep -l status.dat cgi/*.c` hits `cgiutils.c`, `archivejson.c`, `histogram.c`, `tac.c`,
`trends.c`, `statusjson.c`, `objectjson.c`, `extinfo.c`, `statusmap.c` — i.e. essentially every
CGI, *including the two JSON CGIs we just hardened this session*, reads `status.dat` directly
off disk rather than querying the daemon live over a socket/shared memory. Replacing this
format (e.g. with SQLite or an mmap'd binary layout) would mean touching the read path of every
CGI in the tree, not just the writer in `xdata/xsddefault.c` — a much bigger and riskier project
than it first appears, and one that immediately duplicates most of item 2's scope (since you'd
be forced to touch the same 13 legacy CGIs anyway). Not worth it as a dedicated project; if the
write-side cost (periodic full rewrite) turns out to be a real bottleneck for this deployment's
scale, that's the check-engine track's call to make, and any fix there should stay
format-compatible.

---

## 8. Build system (autotools) — minor simplification only, not a rewrite

**Recommendation: SKIP a full replacement. Already appropriately trimmed this session; further
autotools cleanup is low-value busywork, not a strategic item.**

`configure.ac` is already down to ~850 lines after this session's module/worker/Solaris-pkgmk
removal. A CMake/Meson rewrite would be real effort (this file encodes a lot of
platform-detection cruft — Solaris/AIX/HP-UX/Cygwin branches — that could theoretically be
deleted given this package only ever targets one platform via one spec file) for a payoff that's
purely "nicer build tooling," with real upstream-divergence cost (every future upstream
`configure.ac` change becomes a manual merge instead of a clean patch). Given this repo's
explicit design philosophy this session (commit generated `configure`/`config.h.in` rather than
require autotools at rpmbuild time, minimize divergence from upstream where there's no clear
payoff), this isn't worth pursuing. If further trimming happens, it should ride along
opportunistically with other work (e.g. if item 2's CGI removals eventually shrink
`AC_CONFIG_FILES`/build targets further), not as a standalone project.

---

## 9. JSON CGI → sidecar service (Go/Rust/Python replacing statusjson.cgi et al.) — DON'T

**Recommendation: SKIP. Real architectural option, but a worse trade than it sounds.**

Considered seriously: could `statusjson.cgi`/`objectjson.cgi`/`archivejson.cgi` be replaced by
a small non-C service (reading `status.dat`/`retention.dat`, or eventually talking to the
daemon over a proper API) behind the same webserver? Technically yes. But: (a) it throws away
the property this session has repeatedly relied on — "compiles into the RPM, zero extra runtime
dependencies beyond what's already required" — replacing it with "now also ship and supervise a
separate long-running process, in a different language, with its own dependency/update
lifecycle"; (b) the C JSON CGIs are now *actually fine* — we just found and fixed their one
real memory-safety bug this session, with a regression test, and the JSON-object-tree API
(`jsonutils.c`) itself is solid; (c) it's 100% local-fork-only with no upstream-contribution
value, unlike almost everything else in this list where bugfixes at least flow back; (d) it
doesn't obviously solve a problem — CGI-per-request process spawn overhead is real but
small relative to actual check/status workloads, and hasn't shown up as a concern anywhere in
this session's work. This is the kind of "rewrite for its own sake" the user's own stated
principle argues against — not something we personally would gain from, at real ongoing
maintenance cost.

---

## 10. Things checked and found not applicable

- **NERD (Nagios Event Radio Dispatcher, `base/nerd.c`):** off by default
  (`configure.ac:476-483`), not enabled by the packaging spec, so `nerd.o` never even compiles
  into this build. Dead source, live binary is clean. Not worth removing the source file either
  — it's real upstream code (unlike `module/helloworld.c`, which was a sample), and stripping
  unused-but-real upstream features has no local payoff (same reasoning as item 4).
- **Embedded Perl plugin execution:** searched for `p1.pl` and embedded-Perl config options —
  not present anywhere in this nagioscore tree at all. This appears to be an NRPE/plugin-side
  or historical feature that isn't part of nagioscore proper; nothing to modernize or remove
  here.
- **`gd` (graphics library) dependency for PNG generation** (`statusmap.cgi`/`trends.cgi`/
  `histogram.cgi`): still an actively maintained upstream library, not remotely dated. Not a
  modernization candidate on its own — only relevant as a dependency of item 1/2/3's CGIs.

---

## Summary table

| # | Item | Effort | Divergence | Recommendation |
|---|------|--------|------------|-----------------|
| 1 | Port trends/histogram off AngularJS 1.3.9+D3+Bootstrap 3.3.7 | M | local-only | **Do it** |
| 2 | Continue retiring legacy C-HTML CGIs behind SPA (13 views left) | L | local-only | **Continue, prioritized** |
| 3 | statusmap.cgi port | S-M | local-only | Ask user if used first |
| 4 | NEB/broker mechanism | - | - | Leave alone |
| 5 | cgiauth.c auth model | - | - | No action (already externalized) |
| 6 | Object config format | - | - | Leave alone |
| 7 | status.dat/retention.dat format | - | - | Leave alone |
| 8 | Autotools → CMake/Meson | - | - | Skip |
| 9 | JSON CGI → sidecar service | L | local-only | Skip |
