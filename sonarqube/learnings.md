# SonarQube fix — learnings (index + core playbook)

Generic, reusable playbook for the "fix one (or a batch of) SonarQube issues in xwiki-platform"
routine. Keep it compact and generic: record techniques and gotchas, NOT run history — never append
dated anecdotes or PR logs.

## How to use these learnings (READ + WRITE protocol)

This file is the **always-load core**: find-phase strategy + cross-cutting mechanics you need on
every run. Per-rule detail lives in separate files under `rules/` and is loaded **lazily**.

- **READ:** Always read THIS file first. Then, for each rule you actually decide to fix this run,
  ALSO read its detail file from the *Rule index* below. Do NOT read detail files for rules you are
  not fixing — that is where the token savings come from. `token-cost-report.md` is loaded only when
  asked to report token cost. **`dropped-issues.md`** is a skip-index of issue KEYS already analyzed
  and rejected (with reason) — consult it in the find phase and SKIP those keys instead of re-triaging;
  add every new analyzed-but-not-fixed key to it (see *Recording learnings*).
- **WRITE (recording learnings):** put a new learning in the SMALLEST file that owns it — a
  rule-specific gotcha → that rule's file under `rules/`; a cross-cutting technique, build/PR/process
  fact → the matching core section HERE. Merge and trim in place; do not append. Re-synthesize only
  the file(s) you touched, not the whole set. See *Process / conventions → Recording learnings*.

## Rule index (load the file only for the rule you're fixing)

Grouped easiest/safest first. Each entry: rule(s) → detail file. Pick from here in the find phase;
open the detail file only once committed to fixing that rule.

| Family | Rules | Detail file |
|---|---|---|
| Pure syntax/annotation (safest, zero dataflow) | S1128 unused import, S1197 array designator, S1116 empty statement, S1161 missing `@Override`, S1611 redundant lambda parens, S1124 modifier order, S3878 redundant varargs array, S1118 add private ctor | `rules/syntax.md` (S1118 also in `rules/dead-code.md`) |
| Pure simplification (no use-verification) | S1125 boolean literal, S1488 inline return, S1858 pointless `toString()`, S2864 iterate `entrySet()`, S1612 lambda→method ref, S1155 `size()>0`→`!isEmpty()`, S1126 if-else→single return | `rules/simplification.md` |
| Modernization (mid-size safe pools) | S1640 HashMap→EnumMap, S1604 anon class→lambda, S1643 String `+=` in loop→StringBuilder | `rules/S1640-enummap.md`, `rules/S1604-lambda.md`, `rules/S1643-stringbuilder.md` |
| Constant extraction | S1192 duplicated literal | `rules/S1192-duplicated-literal.md` |
| Unused-code removal (light dataflow) | S1068 field, S1481 local, S1854 dead store | `rules/unused-code.md` |
| Utility/dead-code | S1118 private ctor, S1144 unused private method, S1185 remove super-only override | `rules/dead-code.md` |
| Structural | S1066 merge nested `if` | `rules/S1066-nested-if.md` |
| Structural (deepest pool) | S6201 instanceof pattern matching | `rules/S6201-instanceof.md` |
| Other clean | S6204/S6211 `.toList()`, S2093 try-with-resources, S2119 reuse Random, S1143+S1163 finally-throws, S5361 replaceAll→replace, S2147 combine catch, S3626 redundant jump | `rules/other-clean.md` |
| Test-code | S5786 JUnit5 package-private, S5785 assertEquals/assertSame, S3415 swap expected/actual | `rules/test-code.md` |
| Text block (judgment/churn — own PR) | S6126 String concat → text block | `rules/S6126-text-block.md` |

**Denylist — skip these** (bad ROI / risky / not one-liners): `S3776` (cognitive complexity),
`S3252`/`S1845` (API/backward-compat), `S1186` (empty methods), `S115` (naming), `S2447` (null from
Boolean method — in XWiki *script services* null is a deliberate "check getLastError()" contract),
`S1214` (constants-in-interface, cross-module), `S1113` (finalize), `S1215` (`System.gc()` — the
enclosing method may be a deliberately-exposed API, e.g. `$xwiki.gc()`), `S2696` (static field from
instance method — usually lazy-init needing sync), `S2157` (add `clone()`), `S5845` (assert
dissimilar types — erasure can make the assertion correct), `S3415` (swap assert expected/actual —
often breaks order-dependent tests, see `rules/test-code.md`). Verify before "fixing" any of these.

## Picking a target rule (find phase)

- Get the rule distribution cheaply FIRST (no issue bodies): one `issues/search?...&issueStatuses=OPEN&facets=rules&ps=1`
  call returns the whole project rule distribution. For an exact per-rule count read the response
  `total` (query with `&rules=java:SXXXX&ps=1`), not a facet value.
- Pools shift every run (see General techniques). If a rule's remaining issues are all non-convertible
  residue, pivot (skill rule: "if a fix is hard, drop it and pick another").
- **Scope/mode overrides.** A run may target only *new-code* issues (add `&sinceLeakPeriod=true` to
  the search; the pool is smaller, ~100 project-wide, but the same rule families apply) and/or ask for
  *safe changes only*. In safe-only mode stick to purely-mechanical families (syntax/simplification/
  unused/`S1066`) and DROP judgment-heavy ones (logging reformatting `S2629`/`S3457`, `S6880` if→switch,
  `S1130` remove-`throws`, `S1845`); whenever you're unsure a change preserves behaviour, drop it.
- **The BLOCKER/CRITICAL mechanical pool is frequently exhausted.** BLOCKER/CRITICAL-first is the
  skill's guidance but not a hard gate — a clean MAJOR fix beats forcing a risky higher-severity one.
  There is a deep MAJOR-severity clean pool.
- **Check open agent PRs up front** (`search_pull_requests` with `is:pr is:open label:llm-agent
  repo:xwiki/xwiki-platform`; `list_pull_requests` is too big). A recent PR can drain a WHOLE rule
  family. But scope the off-limits check by **(rule + module)**, not rule alone: a per-module batch PR
  only claims the files it touched, so the same rule in OTHER (incl. sibling) modules is fair game.
  When your planned rule already has multiple open PRs, PIVOT to a zero-PR rule family rather than
  threading the gaps. When you must know exactly which modules a wildcard "various modules" PR claims,
  read its file list (`pull_request_read` `get_files`). **A same-FILE open PR is off-limits even for a
  DIFFERENT rule** — a concurrent edit to that file risks merge conflicts; drop that site and pick another.
- Rule families and where the deep pools are: see the *Rule index* above (families ordered
  easiest/safest first). Reliable go-to deep pools when small mechanical rules are drained: S6201,
  then S6204/S6211, then the S1118/S1144/S1185 dead-code trio, then S1066.
- **When the whole tiny pure-mechanical allowlist (S1118/S1128/S1192/S2093/S3626/S6201/S6204/S1068/…)
  is drained AND its residue is already in `dropped-issues.md`, pivot to the MODERNIZATION pool:**
  S1640 (EnumMap ~35), S1604 (lambda ~20), S1643 (StringBuilder ~9). These are undocumented-by-prior-runs,
  behaviour-neutral, and spread across many leaf modules (notifications/ratings/security/office/
  refactoring) — a S1640+S1604+S1643 batch clears 60+ sites in one reactor and easily hits a 30-fix
  target. See their rule files for the drop conditions. (A whole-project rule-distribution query where
  the classic allowlist totals <40 is the signal to pivot here.)

## General batch-fix techniques (apply to every rule)

Cross-cutting mechanics shared by all rules; each rule's detail file notes only its *deltas*.

- **Verify the checkout is at the scan commit BEFORE trusting line numbers.** The container is often
  reset to the *latest* master (not the scanned one) — compare the last-analysis date
  (`api/project_analyses/search?project=…&ps=1` → `analyses[0].date`) against `git log -1 --format=%ci`.
  When they match, line numbers don't drift and line-keyed editing is safe. When the checkout is AHEAD
  (e.g. 3 days), lines DO drift and a line-keyed `old` silently won't match — tell subagents to LOCATE
  each site by the code pattern / Sonar `message` (method/field/constant name), not the line number.
  Brace-delta cross-check (S1066/S6201/S3878) still validates the edits regardless of drift.
- **Apply a many-file mechanical batch in ONE assert-guarded script**, not dozens of `Edit` calls: for
  each `(file, old, new)` assert `content.count(old) == 1` FIRST (catches stale/drifted/ambiguous) and
  write NOTHING if any assertion fails. (`Edit`'s `replace_all` matches only the exact indentation you
  typed and SILENTLY leaves the same pattern at other depths — so prefer the script, and grep for the
  residual pattern after any batch replace.)
- **Delegate STRUCTURAL rules** (reindent/brace surgery — S1066/S6201) and **per-site dataflow rules**
  (S6204/S1068/…) to PARALLEL general-purpose subagents (NOT Explore — they must Edit) over DISJOINT
  files. **Never trust a subagent's self-reported "CONVERTED"** — it routinely reports (with a
  plausible rationale) editing a file it never touched, and a missed dataflow site still COMPILES so
  the build won't catch it. After any delegated batch: cross-check the EXACT expected file set against
  `git diff --name-only`, open every expected-but-absent file and apply the missed sites yourself, and
  re-grep the pre-fix pattern across changed files (only intentional keeps should remain).
- **Two issues on the SAME line** (e.g. two `new T[]{}` on one line) → do NOT stack two edits both keyed
  to the original full line: the second's stored "old" goes stale after the first applies and only one
  lands (or the assert aborts). COMBINE them into one edit covering both replacements.
- **Sonar attributes a multi-line statement's issue to the STATEMENT-START line, not the line the flagged
  expression is actually on** — the reported `line` can be a few lines above the token. When a
  line-number-keyed edit's `old` isn't found there, grep the file for the actual token to get the real line.
- **Line length is the #1 drop cause:** a rewritten line can breach 120 — first drop redundant parens,
  then pick a SHORTER in-scope name; a pre-existing long line with no slack is an unavoidable drop.
  Grep the diff for >120 after every batch.
- **Deleting code orphans its import** → remove the now-unused import too, but ONLY IF the type's
  SIMPLE NAME is absent from the FINAL content, matched with a WORD-BOUNDARY regex (`\bLogger\b`, else
  a plain substring sees `Logger` inside `LoggerFactory`).
- **Pools shift every run** as PRs land — clean rules get EXHAUSTED then slowly REGENERATE; always
  re-query, never assume a rule still has convertible issues.

## Batch mode (Vincent's "fix 20-50 / all of rule X" override)

- Mixed issue types in one PR are explicitly allowed — bundle a purely-additive rule (`S1161`
  @Override) into a removal/simplification batch for a clean multi-type PR.
- **Reach the 20-50 target by density-first module selection:** query the rule(s) with `&ps=500`,
  group by module (`component.split(':')[-1]` for the path — the projectKey has TWO colons, so
  `split(':',1)[1]` is WRONG and every file open fails). Then either: (a) one dense single module
  (oldcore often holds 30-90 of a rule = one cheap build); or (b) a WIDE reactor of many cheap leaf
  modules when the pool is thin-spread. A 30+-module reactor still builds green in ONE shot.
- **oldcore's dense mechanical count is a TRAP for the small clean rules** (S1118/S1144/S1185/S1192/
  S3626/S6204/S1068). Datapoint: a run found 26 oldcore "mechanical" issues across 9 rules and nearly
  ALL were DROPs — S1118 factories are `new XxxFactory()`-instantiated by a service, S1185/S1144 are
  plugin/`.hbm.xml`-mapped reflective, S1192 forward-ref/coincidental, S3626 sits in complex
  try/catch/finally, S6204 escapes via a public getter. oldcore is heavily scanned so its EASY hits are
  long-fixed; a high count there means residue, not cheap fixes, and building oldcore's large test suite
  for 1-3 survivors is poor ROI. Triage a few oldcore candidates with targeted greps (`new Foo(`, the
  class's `extends`, `.hbm.xml`, the constant's decl line) BEFORE committing to it; for the small clean
  rules PREFER leaf feature modules — but note the leaf residue is now ALSO drop-heavy for those rules
  (S6201 is the exception — oldcore S6201 is still ~98% clean, per its rule file). Datapoint: a run where
  the ENTIRE mechanical allowlist (S1118/S1144/S1185/S1192/S1066/S6201/S6204/S3878/S2093/S1128…) held
  only ~56 issues project-wide yielded just 4 clean fixes across leaf modules — S1118 leaf sites were all
  `Abstract*` bases with subclasses or public (non-`internal`) holder classes (revapi), all 9 S2093 leaf
  sites were state-restore false positives, S3878 empty-array self-calls risked recursion, and two S6204
  sites escaped via a public builder-model getter (`*AnalysisResult`). So when the allowlist total is
  small, expect a low clean yield even from leaf modules and don't force the 20-50 target. This tempers
  the dead-code.md "oldcore is FAIR GAME for S1118/S1144/S1185" framing: fair game to *query*, but expect
  heavy drops there now.
- **When a rule's small pool sits below 20 alone, MIX zero-PR pure-mechanical rules** (S1612 + S1125 +
  S2864 + S1155 + S1197 + S1128, all zero-dataflow single-line edits) across ~20 modules into one
  green reactor. Pivoting to zero-PR rules beats threading the gaps in a PR-saturated rule. Reserve
  mixed *dataflow*-rule batches for when unavoidable (each rule multiplies edit-error surface).
- This section covers batch *composition* (which rules/modules to bundle); the *mechanics* of applying
  a batch are in **General batch-fix techniques** above.
- **Collect issue keys by a substring of the full component PATH** (`.../xwiki-platform-chart-macro/...`),
  NOT a guessed short module name (silently returns 0). Build the accept list by KEY, not edit count
  (a triple-nest S1066 merge or a class-level S5786 flag resolves more keys than edited sites).
- **Accept all issues in a loop** (per issue: `add_comment` + `do_transition accept`). Each issue is
  ~2 curls ≈ 4s, so 20+ issues blow a 2-min timeout — run the accept loop as a BACKGROUND task, and/or
  make it idempotent (re-query which keys are still OPEN). `do_transition`'s response does NOT reliably
  contain an `issues` key (don't `['issues']` → KeyError; the transition still applied). Confirm
  separately with `issues/search?issues=<keys>` → all ACCEPTED; re-POST accept for any straggler.

## Find-phase cost

- Inline `sed`/`Read` (offset/limit) of ONE candidate region is cheaper than an Explore subagent for
  mechanical rules; use a subagent only when you must read & reject several candidates. Always trim
  `issues/search` JSON through `python3`/`jq` (keep key,rule,component,line,message) — some rules attach
  huge `flows`/`locations`; never dump raw responses into context.
- Component key = `groupId:artifactId:path` (TWO colons). Path = `component.split(':')[-1]`. Read
  locally at `/home/user/xwiki-platform/<path>`; never fetch file contents remotely.

## Building / verifying

- Checkstyle + Spoon run in `install` (that's what catches the Sonar issues). **Run the tests too —
  never `-DskipTests` (or `-DskipITs`).** Even a mechanical Sonar fix can change runtime behaviour, and
  the edited modules' unit tests are what reveal a regression the compiler can't. If a test fails
  because of your change, fix the change — or drop the issue and pick another; never skip tests to go
  green. Under the background-build discipline in **Cost control** this is a TIME cost, not a token cost.
- **Build ALL affected modules in ONE reactor with `-Plegacy,quality`:** `mvn clean install -B -ntp
  -pl m1,m2,... -Plegacy,quality`. Maven sorts by dependency order; `-Plegacy` is required to include
  `*-legacy-*` modules. **`-Pquality` is MANDATORY for a Sonar-fix verification** — the JaCoCo
  `jacoco:check` goal (enforces each module's pinned `xwiki.jacoco.instructionRatio`) runs ONLY under
  `-Pquality`, and so do Revapi + Enforcer. Removing covered instructions (e.g. S6204
  `.collect(Collectors.toList())`→`.toList()`, or any dead-code/simplification removal) can drop a
  module below its ratio → **JaCoCo fails in CI even when your local `install` was green**. This is
  invisible without `-Pquality`. So NEVER skip revapi/checkstyle/coverage in the verification build,
  and do NOT run bare `-Plegacy` — always `-Plegacy,quality`. (If JaCoCo then fails, the change
  genuinely lowered coverage: prefer a smaller/behaviour-neutral edit, or use the
  `xwiki-increase-test-coverage` skill — do NOT just lower the pinned ratio to go green.)
- **Do NOT use the `snapshot` profile** — it was dropped from the org build recipe (not needed in
  general). Transitive `X.Y.0-SNAPSHOT` deps resolve from the local `~/.m2` (siblings you already
  built, or a prior full build) and the standard XWiki repos. If a `-pl` build hits `Could not find
  artifact org.xwiki.platform:<sibling>:jar:X.Y.0-SNAPSHOT` (a resolution error, NOT your code), add
  `-am` (also-make) so Maven builds the missing local siblings from source — already-installed modules
  are reused, so only the failing sub-tree rebuilds. Prefer building the fix modules alone (no `-am`)
  once the upstream tree is already installed, to keep `-Pquality` fast. **A fully cold `~/.m2`
  (fresh-cloned container, 0 platform artifacts) does NOT need `-am`** — a ~30-module `-pl` reactor
  resolves every SNAPSHOT sibling as a downloaded jar from the remote repos and builds green; `-am`
  is only for a sibling that is genuinely unpublished. Datapoint: a 29-module `-pl` reactor WITH tests
  under `-Plegacy,quality` on a cold `.m2` ran ~19 min.
- **Test-file-only batch: drop a heavy dependency module from `-pl` and let a dependent pull it as a
  remote SNAPSHOT.** For a batch that ONLY edits `src/test` (e.g. S6126 text blocks), a module you'd
  otherwise build just for 1-2 sites but whose test suite is enormous (oldcore) is bad ROI. Leave it out
  of `-pl` entirely: a dependent you DO keep (e.g. `legacy-oldcore`) resolves oldcore as a downloaded
  SNAPSHOT jar from the remote repos (no `-am`, no oldcore test run), and you drop that module's 1-2
  conversions. Net: keep the cheap dense modules, skip the one slow suite. (`-Pquality`'s `jacoco:check`
  runs the FULL module suite — you cannot `-Dtest=` your way to a fast partial verify without failing
  the coverage gate, so excluding the module is the only lever.)
- **A wide reactor that fails on ONE module for a reason UNRELATED to your edit → drop that module,
  keep the rest.** The reactor summary marks every other module `SUCCESS` (they're independently
  verified — build order means a leaf failing last doesn't taint earlier ones). Common cause:
  another agent's already-landed S1118 private-ctor change left `xwiki-platform-legacy-oldcore` (or
  another module) failing revapi `java.method.visibilityReduced` on classes YOU never touched. Confirm
  it's not yours (`git diff --name-only` doesn't list the flagged classes), `git checkout --` your one
  file in that module, and ship the other modules' fixes — no rebuild needed. Don't try to fix the
  unrelated revapi failure (out of scope; skill rule "if hard, drop it").
- **Session plugin cache can be STALE vs the xwiki-dev-llm source.** The build recipe / profiles are
  authoritative in the plugin *repo* (`xwiki/skills/xwiki-build/SKILL.md`, `instructions/xwiki-org.md`),
  which may be several versions ahead of the cached plugin loaded this session. When a reviewer cites
  "latest plugin says X", re-read the source repo, not the cache — and correct this file.
- **Always run mvn from the repo root** (`cd /home/user/xwiki-platform && mvn ...`): the shell cwd can
  silently reset to `/home/user` between turns; a `-pl <relative>` build from the wrong cwd fails fast
  with "Could not find the selected project in the reactor" (a path error, not code — relaunch from root).
- Rough datapoints (build only; running the modules' unit tests adds substantially more time —
  oldcore's suite is large): small leaf modules ~10-45s warm; oldcore ~3.5 min warm / ~6.5 cold;
  feed-api ~5 min. Pick the smallest leaf modules; avoid Solr submodules and feed-api. Now that tests
  run, prefer a few dense modules over a wide reactor — fewer test suites to execute.
- Run the build in the **background**, letting the tool capture stdout to its own `tasks/<id>.output`.
  Do NOT add your own `> build.log` redirect, do NOT `nohup … &`, NO `| tail` (a `| tail -N` on the
  backgrounded `mvn` DISCARDS all but the last N lines from the captured file — you then can't grep
  `Tests run:`/failing-test names, only the final summary). The completion notification carries the
  exit code; grep the full `tasks/<id>.output` for `BUILD SUCCESS` afterwards to confirm.

## Cost control (the build wait dominates the bill — but in TIME, not tokens)

- The token bill is driven by how many TURNS happen while the build runs, NOT how long it takes.
  Running the tests (see Building / verifying) makes builds slower in wall-clock but costs ~the same
  tokens, PROVIDED you keep this discipline: once the build is running, STOP — one line, end the turn.
  Do NOT Read/grep the log while it runs; the completion notification wakes you (one wake-up turn
  re-reads the cached context once, whether the build took 3 min or 25). Don't arm a short
  `ScheduleWakeup` for a build; if you want a fallback use 1200s+.
- Tests add real tokens in exactly one case: a test FAILS and you must diagnose/fix/rebuild — which is
  the regression you WANTED to catch. On clean mechanical fixes this is rare.
- The stop-hook ("uncommitted changes") firing while you wait on the build is EXPECTED — do NOT commit
  before the build verifies.
- **A container restart can kill the background build mid-run.** Working-copy edits PERSIST
  (uncommitted) — don't re-do the fix; check `git status`/`git diff`, confirm the branch, re-launch the
  same `mvn` build. Don't panic-commit unverified.

## GitHub (use the GitHub MCP tools — `gh` porcelain does NOT work here)

- **Why not `gh`:** the `gh` binary may be installed (a setup script can `apt install gh`), but the
  high-level porcelain (`gh pr create/list/view`, `gh issue`) still FAILS in this environment for two
  independent reasons: (1) the session proxy BLOCKS GraphQL — `gh pr`/`gh issue` are GraphQL-backed and
  return `HTTP 403: This GraphQL query is not enabled for this session — only the pinned set of
  PR-review operations is served. Use REST via gh api repos/{owner}/{repo}/... instead`; (2) the git
  remote points to a local proxy (`127.0.0.1:.../git/...`), not github.com, so repo-context commands
  can't resolve the repo (`none of the git remotes ... point to a known GitHub host`). `gh auth status`
  also mis-reports the token invalid. Only `gh api repos/{owner}/{repo}/...` (REST, explicit repo) works
  — but there is no reason to shell out to it: use the **GitHub MCP tools** for all PR/issue operations.
  (The skill text says "open the PR with `gh`" — ignore that here; use `create_pull_request` etc.)
  **`gh api` REST is the fallback for the FEW operations with NO MCP tool** — notably LOCKING a PR
  conversation (Vincent's override, collaborators-only comments): `gh api --method PUT
  repos/{owner}/{repo}/issues/{pr}/lock -f lock_reason=resolved` (verify with the PR's `locked:true`).
  `GH_TOKEN` is set in-env, so `gh api` (REST, not the GraphQL-backed porcelain) authenticates fine.
- Check existing agent PRs once up front with `search_pull_requests` (`is:pr is:open label:llm-agent
  repo:xwiki/xwiki-platform`). Do NOT use `list_pull_requests` (~660KB). The `search_pull_requests`
  result can itself exceed the token limit → parse it from the saved file with python.
- Create the PR with `create_pull_request` (base `master`); add the label + assignee with `issue_write`
  (method=update, `labels:["llm-agent"]`, `assignees:[...]`). Include the SonarCloud issue link(s) in
  the body. (Vincent's override: assign the PR to him — GitHub user `vmassol`.)
- Creating the PR auto-subscribes the session to PR webhooks. XWiki CI is Jenkins and reports later, so
  `get_status` is `pending`/`total_count:0` right after creation — NOT a failure. Webhooks don't deliver
  CI-success / new-push / merge-conflict transitions; for long watches schedule a ~1h self check-in and
  re-arm silently until merged/closed. Reply to reviewers only when genuinely necessary.

## Process / conventions

- Commit + PR title (no JIRA): `[Misc] <desc; mention SonarCloud/SonarQube>`. Security issues: keep the
  description cryptic (public logs).
- **Multi-repo runs (Vincent's "also check xwiki-commons / xwiki-rendering") depend on the session's
  scope.** Each sibling repo needs BOTH a local clone AND that repo in the session's GitHub access
  scope. A session scoped to `xwiki/xwiki-platform` only (no commons/rendering clone, GitHub calls to
  them denied) can't touch them — do the platform work and report the scope limit; don't try to clone
  out-of-scope repos (the proxy blocks them). Each sibling repo also has its OWN designated feature
  branch + its own `SONARQUBE_PROJECT_KEY`.
- **Separate-PR override (safe vs unsure):** when asked to isolate risky fixes, ship the safe
  mechanical batch on the designated branch, and put a judgment-heavy family (e.g. S6126 text blocks)
  on a SIBLING branch (`<designated>-<rule>`) as its own PR, so a reviewer can merge the easy PR without
  the hard one blocking it. Both PRs still get the label/assignee/lock treatment.
- **Author override:** `git config user.email <email>` AND `git commit --author="Name <email>"` AND a
  `Co-Authored-By: Name <email>` trailer — verify with `git log -1 --format='%an <%ae>'`. (This
  routine's override email differs from the git userEmail context — use the override.)
- **Reset the designated feature branch to master FIRST — it persists across runs.** `git fetch origin
  master` then `git checkout -B <branch> origin/master` before editing, or the new PR bundles old
  already-merged commits. (A stale local `origin/master` hides this — always fetch before judging.)
  **The local `origin/master` ref can LAG even right after `git fetch origin master`** (served a stale
  tip). When the feature branch looks like it carries unmerged sonar commits "ahead of master", DON'T
  trust the local ref — cross-check the REAL master HEAD via GitHub MCP (`list_commits` `sha=master`,
  read `[0].sha`). If the branch tip EQUALS the real master HEAD, the prior run's PR was already MERGED
  (squash-merged, so the local commits look distinct) → `git checkout -B <branch> <realMasterSha>` and
  start fresh. If it's genuinely ahead, rebase the unmerged commits onto the real HEAD.
- **Recording learnings (memory repo → `main`).** The xwiki-platform fix lives on a feature branch but
  learnings go to the memory repo's `main`. Do NOT edit on the feature branch then stash/checkout/pop
  (main has diverged; the pop bakes `<<<<<<<` markers into the commit). Instead `git checkout main &&
  git pull origin main` FIRST, then edit and commit directly on main. Write into the SMALLEST owning
  file (see *How to use these learnings*): a rule gotcha → `rules/<that-rule>.md`; a cross-cutting
  fact → the matching section in THIS file. Merge and trim in place — never append dated anecdotes;
  re-synthesize only the file(s) you changed. If you add a NEW rule's detail file, add its row to the
  *Rule index* table above.
