# SonarQube fix — learnings

Generic, accumulated learnings for the "fix one SonarQube issue in xwiki-platform" routine.
Merge & trim — keep this compact; don't just append.

## Picking a target rule (find phase)

- Get the rule distribution cheaply first (no issue bodies), restricting to the mechanical allowlist:
  `.../api/issues/search?...&issueStatuses=OPEN&severities=BLOCKER,CRITICAL&rules=java:S5361,java:S1481,java:S1854,java:S2093,java:S2147&facets=rules&ps=1`
- **`java:S5361` (replaceAll → replace) is the best default mechanical target** (~315 open
  BLOCKER/CRITICAL as of 2026-06). Convert `.replaceAll(pat, rep)` to `.replace(lit, rep)` when the
  pattern denotes a plain literal and the replacement has no regex-replacement specials:
  - First arg must reduce to a literal: either no regex metachars (`. * + ? [ ] ( ) { } | \ ^ $`),
    OR an escaped metachar that is just one literal char — e.g. `"\\/"` is literally `/`, so
    `replaceAll("\\/", ".")` → `replace("/", ".")`. Translate the pattern to its literal meaning.
  - Replacement arg must have no `$` or `\` (those are special only in the *replacement* of
    `replaceAll`). A `.` in the replacement is fine — `.` is special only in the *pattern*.
  - `String.replace(CharSequence,CharSequence)` also replaces ALL occurrences → semantically identical.
- 2026-06 BLOCKER/CRITICAL distribution (for fast targeting): javascript:S3504 ~1800, java:S5361 ~315,
  java:S3776 ~229 (avoid: cognitive complexity), java:S3252 ~135 (avoid: API/back-compat),
  java:S1192 ~129 (define a constant — usually OK), java:S1186 ~107 (avoid: empty methods),
  java:S2093 ~18 (try-with-resources — OK).
- Component key = `projectKey:path`; strip the prefix and read locally at
  `/home/user/xwiki-platform/<path>`. Never fetch file contents over a remote API.
- **One-PR-per-issue mode:** prefer a file with a SINGLE issue of the rule — a half-fix of an
  adjacent-lines file (e.g. AbstractQueryFilter L109+L110) looks sloppy and makes the accept-step
  ambiguous.
- **Batch mode (when told to fix N at once, e.g. "fix 50 S5361 in one PR"):** the opposite — find
  the file with the MOST issues of the rule and fix a contiguous run of N lines in it. S5361 clusters
  hard: a 200-issue pull spanned only 16 files, with `Util.noaccents()` alone holding 183 nearly
  identical `replaceAll("\uXXXX","X")` lines. Fixing 50 contiguous lines there = one module build +
  one trivial scripted edit. Verify safety in bulk with a tiny Python check (no regex metachar in the
  pattern, no `$`/`\` in the replacement) over the line range, then `s/.replaceAll(/.replace(/` on
  just those lines. Leaving the rest of the file's issues unfixed is fine — the PR count is what was
  asked for.
- **Whole-file mode ("fix ALL S5361 in this file"):** don't trust Sonar's per-issue line numbers —
  they come from the last scanned revision and drift from your working copy. Instead operate on the
  file directly: regex every `.replaceAll("PAT","REP")`, and for each decide by PAT, not by line:
  (a) `\uXXXX` or a plain literal (no `\`, no metachar) → keep PAT, just rename to `.replace`;
  (b) escaped single metachar like `"\\*"`/`"\\["` → rename AND rewrite PAT to its literal (`"*"`/`"["`);
  (c) anything with real regex semantics → SKIP. The count of OPEN S5361 in the file should equal the
  number you converted (e.g. 245 converted == 245 open) — a clean cross-check.
- **A backreference pattern like `replaceAll("\\1", …)` is NOT S5361** (it's genuine regex; Sonar
  won't flag it) — leave it. Bonus: `\1` with no capture group throws at runtime, so flag it as a
  latent bug rather than "fixing" it mechanically.
- For mechanical S5361, inline `Read` (offset/limit) of ONE candidate region is cheaper than an
  Explore subagent. Use the subagent only when you must read & reject several candidates.

## GitHub: `gh` is NOT available — use the GitHub MCP tools

- Check for existing agent PRs once, up front: `search_pull_requests` with
  `is:pr is:open label:llm-agent repo:xwiki/xwiki-platform` (small JSON). Do NOT use
  `list_pull_requests` (returns ~660KB and has no `labels` field anyway).
- Create the PR with `create_pull_request` (base `master`). Add the label with `issue_write`
  (method=update, `labels:["llm-agent"]`). The skill text says "open the PR with gh" — ignore that,
  gh is absent here.

## Building / verifying (xwiki-platform)

- Fast verify: `mvn clean install -B -ntp -pl <path> -Plegacy,snapshot -Dxwiki.revapi.skip=true
  -Dxwiki.surefire.captureconsole.skip=true -DskipTests`. Checkstyle + Spoon DO run in `install`.
- **Pick the smallest leaf module** containing the issue to shorten the build wait (the cost
  bottleneck). Datapoints: `xwiki-platform-notifications-notifiers-api` ~3 min,
  `xwiki-platform-office-importer` ~4 min, `xwiki-platform-(legacy-)oldcore` ~6.5 min.
- Run the build in the background (full output to a file, NO `| tail` — tail buffers and breaks
  incremental grep/Monitor). The background-task **completion notification wakes you** with the exit
  code.
- **Do NOT double-background.** Pass the bare `mvn …` command to the run-in-background tool — do NOT
  also wrap it in `nohup … &`. Doing both makes the tracked task capture only the launcher (returns
  exit 0 instantly) while the real build runs detached and unwatched; you then mistake "launcher done"
  for "build done". One layer of backgrounding only.

## Cost control (waiting on the build dominates the bill)

- The fix phase is the most expensive ONLY because it spans the build wait: every wait/poll/wakeup
  turn re-reads the full cached context (~50-65K each). Minimize those turns.
- **Do NOT also arm a short `ScheduleWakeup` (e.g. 300s) for a background build** — the completion
  notification already wakes you, so the wakeup is redundant, and 300s is the worst cache interval.
  If you want a safety fallback, use 1200s+.
- Once the build is running, STOP: output one line and end the turn. Do NOT Read/grep the log while
  it runs — each peek is a full cached-context re-read. The notification carries the exit code; one
  grep of the log afterwards confirms BUILD SUCCESS (don't double-grep both the task-output and the
  log — one is enough).
- The stop-hook ("uncommitted changes") fires while you correctly wait on the build. Expected — do
  NOT commit before the build verifies.
- Per-phase cost is recoverable from the transcript jsonl (`/root/.claude/projects/<id>.jsonl`):
  dedupe by `message.id`, bucket by timestamp (boundaries = the fix Edit, and the PR-creation call).
  Cumulative cache-read dwarfs everything else; fresh input + output stay tiny with narrow reads.

## Process / conventions

- Commit + PR title (no JIRA): `[Misc] <desc; mention SonarCloud/SonarQube>`. Override author:
  `git config user.email <email>` AND `git commit --author="Name <email>"`.
- Push with `git push -u origin <branch>`. Include the SonarCloud issue link in the PR body.
- After the PR: comment + accept the issue —
  `add_comment` then `do_transition` with `transition=accept` → status RESOLVED, resolution WONTFIX.
- **Bulk-accepting many issues:** each curl is ~1.2s, so 50 issues × (comment+transition) blows the
  2-min foreground Bash timeout. Run the accept loop as a **background** task (it wakes you on
  completion). After it returns, confirm with one `issues/search?issues=<comma-keys>` and check all
  are RESOLVED.
- **Token-cost report (when asked):** dedupe transcript by `message.id`, bucket by two boundary
  timestamps — the fix Edit and the `create_pull_request` call — summing input/output/cache_read/
  cache_write per phase. Phase 3 ("post") is necessarily a mid-phase snapshot since you're still in it.
- Creating the PR auto-subscribes the session to PR webhook activity; XWiki CI is Jenkins and reports
  later, so `get_status` is `pending`/`total_count:0` right after creation — not a failure.
- Security issues: keep the PR/commit description cryptic (public logs).
