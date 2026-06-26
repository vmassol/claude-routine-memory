# SonarQube fix — learnings

Accumulated, generic learnings for the "fix one SonarQube issue in xwiki-platform" routine.
Append new, genuinely-useful items; don't duplicate.

## Picking a target rule (find phase = most token-expensive)

- Get the rule distribution cheaply first (no issue bodies):
  `.../api/issues/search?...&issueStatuses=OPEN&severities=BLOCKER,CRITICAL&facets=rules&ps=1`
- **`java:S5361` (replaceAll → replace) is the best default mechanical target.** As of 2026-06 there
  were ~317 open BLOCKER/CRITICAL instances. Fix = change `.replaceAll("literal", x)` to
  `.replace("literal", x)` ONLY when the first arg is a string literal with no regex metacharacters
  (`. * + ? [ ] ( ) { } | \ ^ $`). `String.replace(CharSequence,CharSequence)` also replaces all
  occurrences, so it is semantically identical. One-line, low-risk, isolated.
- Snapshot of the BLOCKER/CRITICAL distribution (2026-06, for fast next-time targeting):
  javascript:S3504 ~1800, java:S5361 ~317, java:S3776 ~229 (avoid: cognitive complexity),
  java:S3252 ~135 (avoid: often API/back-compat), java:S1192 ~129 (define a constant — usually OK),
  java:S1186 ~107 (avoid: empty methods), java:S2093 ~18 (try-with-resources — OK).
- Component key format is `projectKey:path/within/repo`; strip the `projectKey:` prefix and read the
  file locally at `/home/user/xwiki-platform/<path>`. Never fetch file contents over a remote API.

## Checking for existing agent PRs (do this once, up front)

- `gh` is NOT available in this environment. Use the GitHub MCP tool
  `search_pull_requests` with `is:pr is:open label:llm-agent` — returns a small JSON.
- Do NOT use `list_pull_requests` to find them: it returns ~660KB (overflows the tool-result limit
  and gets offloaded to a file) and the page payload has no `labels` field anyway.

## Building / verifying (xwiki-platform)

- A single leaf module builds with just `-pl <path> -Plegacy,snapshot` (no `-am` needed) IF its
  deps are already in `~/.m2` — they were on a fresh clone here.
- Fast compile+checkstyle verify: `mvn clean install -B -ntp -pl <path> -Plegacy,snapshot
  -Dxwiki.revapi.skip=true -Dxwiki.surefire.captureconsole.skip=true -DskipTests`.
  `xwiki-platform-legacy-oldcore` took ~6.5 min this way. Checkstyle + Spoon DO run in `install`.
- **Background-build gotcha:** piping the build through `| tail -40` means NO incremental output —
  `tail` buffers everything until the pipe closes, so a Monitor grepping the output file only fires
  at the very end. To watch progress, write the FULL output to a file (no `tail`) and grep that, or
  use `tee`.

## Process / conventions

- Commit + PR title format for Sonar fixes (no JIRA): `[Misc] <desc; mention SonarCloud>`. Add the
  `llm-agent` label to the PR (via `issue_write` update, `labels:["llm-agent"]`). Base branch is
  `master`.
- Override commit author when required: `git config user.email ...` AND pass
  `--author="Name <email>"` to `git commit` (sets both author and committer).
- Mark the issue accepted after the PR: `do_transition` with `transition=accept` →
  status `ACCEPTED`, resolution `WONTFIX`. Add a linking comment first via `add_comment`.

## Token-cost insight

- Exact per-phase token usage is recoverable from the session transcript jsonl
  (`/root/.claude/projects/<id>.jsonl` + `subagents/agent-*.jsonl`); dedupe by `message.id` (each
  message appears ~3×). Bucket by timestamp into find/fix/post-fix.
- The dominant raw token component is cumulative **cache-read** (cheap, ~10% rate). The expensive
  components (fresh input + cache creation + output) stay small when file reads are narrow and
  triage is delegated to an `Explore` subagent.
- **Waiting on a long build is what inflates cost**: every wakeup/poll turn re-reads the full cached
  context. Prefer fewer, longer wakeups (or one Monitor) over frequent polling while a 5–10 min
  build runs.
- **MISTAKE made 2026-06: armed a build Monitor but then kept manually `Read`-ing `build.log`
  anyway.** Each manual Read is a full turn that re-reads the whole cached context (~50K+ each), and
  these dominated the run: fix-phase total was ~1,059K tokens, almost entirely cache-read from
  ~10 needless log peeks during a ~4-min build (find phase, by contrast, was only ~263K). Rule:
  **once a Monitor is armed for the build, STOP — output a one-line status and end the turn; do not
  Read the log, do not re-grep it.** The Monitor event (and the background-task completion
  notification) will wake you. Polling it yourself is the single biggest avoidable cost.
- The stop-hook ("uncommitted changes") will fire while you're correctly waiting on the build. That
  is expected — do NOT commit before the build verifies. Just check the build result and proceed.

## Per-rule fix datapoints

- `java:S5361` again, smooth: `textContent.replaceAll(" ", "&nbsp;")` → `.replace(" ", "&nbsp;")`
  (office-importer, 2026-06). When converting, check BOTH args: first arg must have no regex
  metacharacters AND the replacement arg must have no `$`/`\` (regex-replacement specials). Both
  plain here → trivially safe.
- For a mechanical rule like S5361 you can skip the `Explore` subagent: reading ~15 lines of ONE
  candidate inline kept the find phase cheap (~263K). The subagent matters when you must reject
  several candidates; for a one-snippet pick, inline `Read` with offset/limit is fine.

## Fast small-module build targets (prefer over legacy-oldcore's ~6.5 min)

- `xwiki-platform-office-importer` builds (clean install, skipTests, checkstyle+Spoon) in ~4 min.
  When the chosen issue's component path lets you pick the module, favor a small leaf module over
  oldcore/legacy-oldcore to shorten the build-wait (which is the cost bottleneck — see above).
