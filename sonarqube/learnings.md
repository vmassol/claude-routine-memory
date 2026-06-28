# SonarQube fix — learnings

Generic, accumulated learnings for the "fix one SonarQube issue in xwiki-platform" routine.
Merge & trim — keep this compact; don't just append.

## Picking a target rule (find phase)

- Get the rule distribution cheaply first (no issue bodies), restricting to the mechanical allowlist:
  `.../api/issues/search?...&issueStatuses=OPEN&severities=BLOCKER,CRITICAL&rules=java:S5361,java:S1481,java:S1854,java:S2093,java:S2147&facets=rules&ps=1`
- **`java:S5361` (replaceAll → replace) is the best mechanical target.** Convert
  `.replaceAll(pat, rep)` → `.replace(lit, rep)` when the pattern denotes a plain literal and the
  replacement has no regex-replacement specials:
  - First arg must reduce to a literal: either no regex metachars (`. * + ? [ ] ( ) { } | \ ^ $`),
    OR an escaped metachar that is just one literal char — `"\\+"`→`"+"`, `"\\/"`→`"/"`,
    `"\\."`→`"."`. Rewrite the pattern to its literal meaning when you change the method name.
  - Replacement arg must have no `$` or `\` (special only in `replaceAll`'s replacement). A `.` in
    the replacement is fine. `String.replace(CharSequence,CharSequence)` also replaces ALL → identical.
  - Char classes (`[Æ...]`), `\s+`, backreferences, `;jsessionid=.*?...` are genuine regex —
    Sonar does NOT flag them; leave them. A file can have both flagged literals and unflagged regex
    on different lines (e.g. `XWiki.java`: only the literal `"-"` flagged, the `noaccents` char-class
    table untouched). Fix exactly the reported lines, not every `replaceAll` in the file.
- **Counts shift fast as prior PRs land.** 2026-06: S5361 down to ~19 open BLOCKER/CRITICAL (was
  ~315 earlier in the month). Always re-query, never trust a stale count. Other 2026-06 BLOCKER/CRIT:
  javascript:S3504 ~1800, java:S3776 ~229 (avoid: complexity), java:S3252 ~135 (avoid: API),
  java:S1192 ~129 (constant — usually OK), java:S1186 ~107 (avoid: empty methods),
  java:S2093 ~18 (try-with-resources — OK).
- Component key = `projectKey:path`; strip prefix, read locally at `/home/user/xwiki-platform/<path>`.
  Never fetch file contents over a remote API.
- **One-PR-per-issue mode:** prefer a file with a SINGLE issue of the rule (clean accept-step).
- **Don't trust Sonar's per-issue line numbers blindly** — they can drift from your working copy.
  Cross-check: `grep -n "\.replaceAll(" <file>` and match by the literal pattern, not the line. The
  count of OPEN issues in a file should equal the number you convert (clean cross-check).

## Find-phase cost

- Inline `Read` (offset/limit) of ONE candidate region is cheaper than an Explore subagent for
  mechanical rules. Use a subagent only when you must read & reject several candidates.
- Always trim `issues/search` JSON through `python3`/`jq` (keep key,rule,component,line,message) —
  some rules attach huge `flows`/`locations`. Never dump raw responses into context.

## Batch / "fix ALL of rule X" mode (e.g. Vincent's S5361 override)

- One `issues/search?rules=java:S5361&ps=100` gives every open issue + component + line. Group by
  file/module.
- **Apply scattered single-line fixes with a per-line Python script**, not sed (sed escaping of
  `\\+`/quotes is error-prone). For each `(relpath, 1-based line, old_substr, new_substr)`: assert
  `old_substr in lines[idx]` and fail loudly on mismatch before writing. This caught nothing wrong
  this run (19/19) precisely because grep gave working-copy line numbers. Delete the helper script
  before committing.
- **Build ALL affected modules in ONE reactor invocation**, not one-by-one:
  `mvn clean install -B -ntp -pl m1,m2,m3,... -Plegacy,snapshot -Dxwiki.revapi.skip=true
  -Dxwiki.surefire.captureconsole.skip=true -DskipTests`. Maven sorts them in dependency order;
  missing transitive deps resolve from the remote snapshot repo (`-Psnapshot`). 19 fixes across 5
  modules (oldcore + legacy-oldcore + query-manager + feed-api + notifications-notifiers-default)
  built in ONE ~8-min wait. **`-Plegacy` is required** to include `*-legacy-oldcore`.
- Accept all issues in one loop (each issue: add_comment + do_transition accept). 19×2 curls ≈ 45s
  (OK in foreground). For 50+ run it as a **background** task. Confirm after with one
  `issues/search?issues=<comma-keys>` → all RESOLVED/WONTFIX.

## Building / verifying

- Checkstyle + Spoon run in `install` (that's what catches issues); `-DskipTests` is fine.
- Pick the smallest leaf module(s). Datapoints: notifications-notifiers-api ~3 min, oldcore ~3.5 min
  in a warm reactor (~6.5 min cold), feed-api/legacy-oldcore <1 min each.
- Run the build in the **background**, full output to a file (NO `| tail` — it breaks incremental
  grep). The completion notification wakes you with the exit code. **One layer of backgrounding only**
  — pass the bare `mvn …` to the run-in-background tool; do NOT also wrap in `nohup … &`.

## Cost control (the build wait dominates the bill)

- The fix phase is most expensive because it spans the build wait: every wait/poll/wakeup turn
  re-reads the full cached context (~50-65K each). Minimize those turns.
- Once the build is running, STOP: one line, end the turn. Do NOT Read/grep the log while it runs.
  The notification carries the exit code; ONE grep of the log afterwards confirms BUILD SUCCESS.
- Do NOT arm a short `ScheduleWakeup` (300s) for a background build — the completion notification
  already wakes you. If you want a fallback, use 1200s+.
- The stop-hook ("uncommitted changes") fires while you correctly wait on the build — expected, do
  NOT commit before the build verifies.

## GitHub: `gh` is NOT available — use the GitHub MCP tools

- Check existing agent PRs once up front: `search_pull_requests` with
  `is:pr is:open label:llm-agent repo:xwiki/xwiki-platform`. Do NOT use `list_pull_requests` (~660KB).
- Create PR with `create_pull_request` (base `master`). Add label with `issue_write`
  (method=update, `labels:["llm-agent"]`). Ignore the skill's "open with gh".
- Creating the PR auto-subscribes the session to PR webhook activity. XWiki CI is Jenkins and reports
  later, so `get_status` is `pending`/`total_count:0` right after creation — NOT a failure. Webhooks
  don't deliver CI success/new pushes/merge-conflict transitions; for long watches schedule a
  ~1h self check-in (`send_later` if present, else `CronCreate` one-shot) and re-arm silently.

## Process / conventions

- Commit + PR title (no JIRA): `[Misc] <desc; mention SonarCloud/SonarQube>`.
- **Author override:** `git config user.email <email>` AND `git commit --author="Name <email>"`,
  AND a `Co-Authored-By: Name <email>` trailer — verify with `git log -1 --format='%an <%ae>'`.
  (This routine's override email differs from the git userEmail context — use the override.)
- Push with `git push -u origin <branch>`. Include the SonarCloud issue link(s) in the PR body.
- Security issues: keep PR/commit description cryptic (public logs).

## Token-cost report (when asked)

- Parse the transcript jsonl at `/root/.claude/projects/<id>.jsonl`. Dedupe assistant turns by
  `message.id`; sum input/output/cache_read/cache_write per phase.
- **Find phase boundaries by tool_use NAME, not string-match on content.** Matching the literal
  string `"create_pull_request"` in message content gives a FALSE early boundary (the name appears in
  early schema-loading text). Iterate `content[]` blocks, take `type=="tool_use"` and match
  `b["name"]=="mcp__github__create_pull_request"` (post boundary) and the Write/Bash of the fix
  (find→fix boundary). Verify boundary timestamps are monotonic before trusting them.
- Buckets: (1) find = up to the first fix edit; (2) fix = edit + build + commit + push; (3) post = PR
  + label + accept + report (necessarily a mid-phase snapshot). Cache-read dwarfs everything;
  fresh-input/output stay tiny with narrow reads. Edit the PR **body** to add the report (don't
  comment) if the routine asks for it.
