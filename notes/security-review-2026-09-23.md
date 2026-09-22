# Security review: nagioscore CGI/base (2026-09-23)

Scope: `/claude/nagios-core/nagioscore` @ `al2023-4.4.14-kuriyama12`. Areas audited: `cgi/getcgi.c`, `cgi/cmd.c`, `cgi/cgiauth.c`, `cgi/statusjson.c`/`objectjson.c`/`archivejson.c`, `base/checks.c`/`base/commands.c`, plus a general grep sweep of `cgi/*.c`/`base/*.c` for classic C bug patterns. This is a read-only audit — no fixes implemented, no git state touched. Quality bar: every finding below was traced from an untrusted source (HTTP request) all the way to its sink, not just flagged from pattern-matching; several initially-alarming leads turned out to be correctly guarded on closer inspection, and are recorded under "Swept clean" specifically so the leads aren't re-walked.

## Findings, most severe first

### 1. MEDIUM — `sanitize_cgi_input()` is dead code (defense-in-depth layer silently disabled)

**File:** `cgi/getcgi.c:18-42`, called unconditionally from `getcgivars()` at line 324 (the single shared CGI-input entry point used by every CGI in the package).

```c
void sanitize_cgi_input(char **cgivars) {
	char *strptr;
	int x, y, i;
	int keep;

	/* don't strip for now... */
	return;

	for(strptr = cgivars[i = 0]; strptr != NULL; strptr = cgivars[++i]) {
		...
		/* remove potentially nasty characters */
		if(strptr[x] == ';' || strptr[x] == '|' || strptr[x] == '&' || strptr[x] == '<' || strptr[x] == '>')
			keep = 0;
```

The function's very first statement is an unconditional `return;`, making everything below it (which would strip `;|&<>` from every decoded CGI variable, application-wide) unreachable. Confirmed via `git log` that this predates this session by years (last touched 2021-03-12, "Bug fixes, status format change, misc cleanup"; traces to "Initial import of Nagios code") — this is inherited upstream behavior, not something introduced locally.

**Why this matters:** the presence of this function suggests upstream once intended a blanket sanitization safety net across all CGI input, but it was deliberately disabled (probably because stripping those characters unconditionally breaks legitimate free-text fields like comments). Its being dead doesn't by itself create a new hole — every individual sink is responsible for its own escaping — but it means **there is no defense-in-depth layer**: a missed escape in any single CGI is directly exploitable, with nothing else standing in the way. This raises the stakes of getting every individual sink right (see the `cmd.cgi` trace in "Swept clean" below for one sink that IS correct).

**Confidence:** CONFIRMED (direct code reading, unambiguous).

**Suggested fix direction:** Either delete the dead function entirely (it does nothing, so removing it is a pure no-op) or actually re-enable a *narrower*, correctness-preserving version of it if there's an appetite for defense-in-depth (e.g., strip control characters only, leave `;|&<>` which have legitimate uses in comments/output text and are already handled at each real sink). Given every sink checked in this pass turned out to already handle its own escaping correctly, deletion is the lower-risk option; re-enabling anything risks behavior changes to fields that currently accept these characters (e.g. comment text containing `&`).

### 2. LOW — `json_object_add_member()` leaks the old allocation on `realloc()` failure

**File:** `cgi/jsonutils.c:292-297`

```c
obj->members = realloc(obj->members,
		((obj->member_count + 1) * sizeof(json_object_member *)));
if(NULL == obj->members) {
	obj->member_count = 0;
	return NULL;
	}
```

Same antipattern as the one already fixed in `json_escape_string()` this session (kuriyama11): assigning `realloc()`'s return value directly over the only pointer to the original allocation. On OOM, the original `obj->members` buffer is leaked (not a use-after-free or corruption — `member_count` is correctly reset to 0 so nothing dereferences the lost pointer — just a leak). Practically low-severity: reaching this requires an actual allocation failure, at which point the process is generally in trouble anyway. Not exploitable for memory corruption.

**Confidence:** CONFIRMED (code reading; same bug class already fixed once this session, so the pattern-match is exact).

**Suggested fix:** same shape as the `json_escape_string()` fix — realloc into a temporary, only overwrite `obj->members` on success, `free()` the old pointer explicitly on failure before returning NULL.

## Swept clean (verified safe, don't re-walk these)

- **`cmd.cgi` command-pipe injection via embedded newlines** — this was the most promising lead and took the most effort to close. `host`, `hostgroup`, `service`, `servicegroup`, `com_author`, `com_data` are all set via `strdup()` of the raw, unescaped POST value (only `strip_html_brackets()` applied, which strips `<`/`>` only) — no per-field check for `;` or `\n` at assignment time (`cgi/cmd.c` ~380-460). `commit_command()` (`cgi/cmd.c:1882`) rejects `;` in `host_name`/`service_desc`/`hostgroup_name`/`servicegroup_name`/`comment_author` but *not* `comment_data`; that gap is closed separately by `clean_comment_data()` (`cgi/cmd.c:2225`, replaces `;` with space), called unconditionally on both `comment_data` and `comment_author` in `commit_command_data()` (`cgi/cmd.c:1458,1464`) before any command-type dispatch — confirmed `commit_command_data()` is the single call site that leads to `commit_command()` (`cgi/cmd.c:229,1822`), so there's no path that skips it. None of this touches `\n` though. The actual newline defense is a **global, last-mile check on the fully-assembled command string** right before the pipe write: `write_command_to_file()` (`cgi/cmd.c:2177`) does `if(!cmd || !*cmd || strchr(cmd, '\n')) return ERROR;` with a comment that names this exact threat ("malicious users... bypass the access-restrictions"). Verified on the read side too: `command_input_handler()` in `base/commands.c:144` splits the raw command-pipe byte stream on `"\n"` via `iocache_use_delim()` and dispatches each line independently to `process_external_command1()` with no further per-line authorization (by design — the pipe itself is the trust boundary, which is exactly why the CGI-side newline check matters and is correctly load-bearing here). **Conclusion: newline injection into the command pipe is blocked**, just at a different (and more robust — catches it regardless of which field carries it) choke point than a naive per-field audit would expect.
- **`cmd.cgi` buffer overflow via `plugin_output`/`performance_data`/`start_time_string`/`end_time_string`** — a grep for non-literal `strcpy()` initially flagged `cgi/cmd.c:535,552,608,623` as copying raw CGI values with `strcpy` (no `n`-bounded variant). All four are actually safe: `plugin_output`/`performance_data` are `char[MAX_INPUT_BUFFER]` (1024 bytes, `cgi/cmd.c:71-72`) and are guarded by an explicit `if(strlen(variables[x]) >= MAX_INPUT_BUFFER - 1) { error = TRUE; break; }` immediately before the `strcpy` (`cgi/cmd.c:529,543`); `start_time_string`/`end_time_string` are `malloc(strlen(variables[x]) + 1)`-sized exactly before their `strcpy` (`cgi/cmd.c:598-599,613-614`). Correct in both cases, just via two different patterns.
- **JSON CGIs' format-string/escaping discipline** (`statusjson.c`/`objectjson.c`/`archivejson.c`) — zero `strcpy`/`strcat`/raw `sprintf` in any of the three files (all string building goes through `json_object_append_string`/`json_array_append_string`/`asprintf`). Spot-checked every `NULL`-escapes call site (`grep -n "append_string(.*NULL"`): all either have a compile-time-literal format string with no `%s` substitution of variable data (e.g. `"%d,%d,%d"` with numeric counters) or the variable being escaped as the format argument itself is correctly paired with `&percent_escapes` (e.g. `objectjson.c:3995`, servicegroup names). No bypass of the `percent_escapes`/`json_escape_string` machinery found.
- **`archivejson.c` log file path traversal** — `get_log_archive_to_use()` (`cgi/cgiutils.c:1465`) builds archive filenames via `snprintf(buffer, len, "%snagios-%02d-%02d-%d-%02d.log", log_archive_path, tm_mon+1, tm_mday, tm_year+1900, tm_hour)` — every substituted field is a numeric struct-tm component derived from a computed `time_t`, never a raw attacker string; there is no `%s` of request-controlled text anywhere in this path. The `archive==0` branch uses `log_file`, which is set only from `cgi.cfg`'s `log_file=` directive (`cgi/cgiutils.c:475-480`), not from any CGI parameter. No traversal vector.
- **`cgiauth.c` authorization logic** — matches upstream design; no logic-inversion bugs (nothing that defaults to "authorized" on an unhandled/error case). Two long-standing *upstream* quirks worth knowing but not fixing here: (a) `is_authorized_for_hostgroup`/`_servicegroup` (view access) return `TRUE` if the user can view *any single* member host/service, not all of them (explicit history in the comments: "CHANGED in 2.0 ... Reverted for 3.3.2"), while the *commands* variants correctly require every member to pass; (b) `is_authorized_for_hostgroup_commands`/`_servicegroup_commands` vacuously return `TRUE` for an empty group (loop body never executes). Neither is remotely triggerable as a meaningful privilege escalation (the empty-group case is a no-op; the view-access case is deliberate, documented upstream behavior).
- **`getcgi.c` POST body handling** — `Content-Length` is bounds-checked (`< 0` or `>= INT_MAX - 1` rejected, `cgi/getcgi.c:171`); `pairlist` growth arithmetic (`(paircount + 256) * sizeof(char *)`) can't overflow 64-bit `size_t` even at the theoretical max paircount, it would just fail allocation cleanly and exit. A pathological `&&&&&...` body could still cost real memory/CPU before hitting that wall — bounded in practice by whatever request-size limit the front-end webserver enforces, not by this code. Not flagging as an actionable bug; standard resource-exhaustion class present in any CGI of this vintage, not something introduced here, and reasonably mitigated at the webserver layer.
- **`PARANOID_CGI_INPUT`** (`cgi/getcgi.c:14`) — explicitly `#undef`'d in-file, so `unescape_cgi_input()`'s printable-ASCII-only filter (would have stripped control chars including `\n`) can never be compiled in even if someone passed `-DPARANOID_CGI_INPUT`. Noted for context (it's *why* control characters survive decoding at all, which is what made the command-injection lead above worth chasing down) but not a standalone actionable finding since nothing downstream currently relies on it being active.
- General grep sweep: `gets(` — zero hits. `system()`/`popen()` — 2 call sites total; `cgiutils.c:1894`'s `system(filename)` only runs admin-installed, filesystem-executable SSI header/footer scripts whose path is built entirely from server config (`physical_html_path`) and a fixed per-CGI name macro, never from a request parameter — requires the SSI directory to already contain an attacker-planted executable file, i.e. presupposes a worse compromise already happened; `utils.c:621`'s `popen()`/`my_system()` is the plugin-execution path, discussed below. No remaining `mbstowcs`/`wcstombs`/`mbrtowc` usage anywhere outside the code already fixed in `json_escape.c`. Did not exhaustively hand-review all ~150 `strcpy`/~145 `sprintf` call sites across the full `cgi/`/`base/` tree (mostly in the older non-JSON CGIs like `status.c`/`extinfo.c`/`showlog.c`/`history.c` and in `base/*.c`) — spot-checks (`showlog.c`'s icon-name `strcpy`s, all literal-macro sources) found nothing, but a full pass would benefit from static-analysis tooling (`cppcheck`/`clang-tidy`) rather than manual grep triage; flagging as a follow-up rather than claiming completeness.

## Out of scope for this pass (architectural, not a fixable "bug")

Plugin/notification/event-handler command execution (`lib/runcmd.c:372`, `argv[0] = "/bin/sh"`) shells out check_command/event_handler lines with Nagios macros substituted in, and macro substitution does **not** shell-escape values (e.g. `$HOSTOUTPUT$`/`$SERVICEOUTPUT$`, which can originate from a passive check result submitted via `cmd.cgi` if the submitting user is authorized for `PROCESS_*_CHECK_RESULT`). This is well-documented, decades-old upstream Nagios design — the whole point of `check_command`/`event_handler` config syntax is shell semantics, and changing it would be a major compatibility-breaking redesign, not a bug fix. Worth a mention in the modernization-survey track as a "know your risk model" item (admins should avoid `$*OUTPUT$`-style macros in shell-sensitive positions in event handlers), not something to patch here.
