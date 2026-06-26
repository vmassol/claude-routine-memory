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
- **Prefer a file with a SINGLE issue of the rule.** If a file has the same rule on adjacent lines
  (e.g. AbstractQueryFilter L109+L110), the "one PR per issue" rule makes a half-fix look sloppy and
  the accept-step ambiguous — pick a single-issue file instead.
- For mechanical S5361, inline `Read` (offset/limit, ~15 lines) of ONE candidate is cheaper than an
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
- Security issues: keep the PR/commit description cryptic (public logs).
