# SonarQube fix — learnings

Generic, accumulated learnings for the "fix one SonarQube issue in xwiki-platform" routine.
Merge & trim — keep this compact; don't just append.

## Picking a target rule (find phase)

- Get the rule distribution cheaply first (no issue bodies), restricting to the mechanical allowlist:
  `.../api/issues/search?...&issueStatuses=OPEN&severities=BLOCKER,CRITICAL&rules=java:S5361,java:S1481,java:S1854,java:S2093,java:S2147&facets=rules&ps=1`
- **`java:S2093` (try-with-resources) is the current prime target** (all CRITICAL, ~11 open as of
  2026-06-30 after this run's batch of 5; drains slowly). S5361/S1481/S1854/S2147 are EXHAUSTED.
  Convert `Resource r = new ...(); try { ... } [catch] finally { r.close()/IOUtils.closeQuietly(r) }`
  → `try (Resource r = new ...()) { ... } [catch] }` — drop the finally.
- **CRITICAL caveat: ~half the S2093 hits are NOT real resource closes.** Sonar fires on any
  try/finally it *thinks* could be try-with-resources, including push/pop and state-restoration
  patterns that are NOT `AutoCloseable` and CANNOT be cleanly converted. SKIP these: `boolean pushed`
  + `renderingContext.pop()`; `scriptContext.removeAttribute/setWriter`; `xcontext.setWikiReference`;
  semaphore acquire/release; a `DataInputStream` wrapper whose finally does `stream.reset()` (closing
  it would close the caller's stream); a resource created mid-body (StringWriter/ZipOutputStream)
  whose finally doesn't close it. Verify the finally actually CLOSES an AutoCloseable declared just
  before `try`. In one 16-issue pull only ~5 were convertible.
- **Compile-safety check before converting:** the implicit `close()` throws `IOException`. If the
  surrounding catch only catches a *narrower* type (e.g. `FileNotFoundException`) and the method does
  NOT declare `throws IOException`, the converted code won't compile. Either the method already
  declares/catches `IOException` (fine), or add a `catch (IOException)` that wraps it sensibly (did
  this for `XWikiConfig(String)` — wrapped as config FORMATERROR). Check throws clause + catch types.
- **Removing `IOUtils.closeQuietly` may orphan the `IOUtils` import** → checkstyle failure. `grep -n
  IOUtils <file>` first; if it was the only use, delete the import line too.
- Done: `XarPackage.read` (2026-06-30); batch of 5 on 2026-06-30 — `Packager`, `XWikiExecutor`,
  `Importer`, `XWikiConfig`, `ZipExplorerPlugin`. S2093 issues are each in a DIFFERENT module (no
  single-module cluster), so a 5-fix PR needs a multi-module reactor build (~8-9 min: oldcore ~5 min
  cold + packager-plugin ~2.5 min + 3 leaf modules <20s each). Feed the triage subagent ALL candidate
  file:line pairs in ONE message (forgot 7 the first time and had to do a 2nd round).
- **`java:S5361` (replaceAll → replace)** — kept for reference / the batch override. Convert when the
  first arg reduces to a literal (no regex metachars `. * + ? [ ] ( ) { } | \ ^ $`, OR an escaped
  metachar that is one literal char: `"\\+"`→`"+"`, `"\\."`→`"."`) AND the replacement has no `$`/`\`
  (specials only in `replaceAll`'s replacement). Char classes, `\s+`, backrefs, `;jsessionid=.*?` are
  genuine regex — Sonar does NOT flag them; fix only the reported lines, not every `replaceAll`.
- **Counts shift fast as prior PRs land — always re-query.** 2026-06-29 open BLOCKER/CRIT:
  javascript:S3504 ~1800, java:S3776 ~229 (avoid: complexity), java:S3252 ~135 (avoid: API),
  java:S1192 ~129 (constant — usually OK), java:S1186 ~107 (avoid: empty methods),
  java:S2093 ~18 (try-with-resources — OK), java:S1948 ~46, java:S115 ~19 (naming — risky).
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
- **Don't add your own `> build.log` redirect to a backgrounded mvn.** The run-in-background tool
  already captures stdout to its own `tasks/<id>.output` file; a `>` redirect leaves that file nearly
  empty, so grepping it for `BUILD SUCCESS` finds nothing and looks like a failure. Either grep the
  file you redirected to, or (simpler) drop the redirect and grep the tool's `.output` file.

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
- **Recording learnings (memory repo, pushed to `main`):** the routine's xwiki-platform fix lives on
  a feature branch but learnings go to `main`. Do NOT edit learnings.md on the feature branch then
  `git stash`/`checkout main`/`stash pop` — main has diverged, the pop conflicts, and a careless
  `add`+`commit` will bake `<<<<<<<` markers into the commit. Instead: `git checkout main &&
  git pull origin main` FIRST, then edit learnings.md directly on the freshly-pulled main and commit.

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
