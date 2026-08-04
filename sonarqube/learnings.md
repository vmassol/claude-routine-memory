# SonarQube fix — routine playbook

Cross-cutting playbook for the "fix one (or a batch of) SonarQube issues" routine. Keep it compact:
record techniques and observed state, NOT run history — never append dated anecdotes or PR logs.

## How to use these learnings (READ + WRITE protocol)

**Per-rule fix correctness is NOT in this repo.** It lives in the plugin OKF at
[`xwiki/okf/sonarqube/`](https://github.com/xwiki/xwiki-dev-llm/tree/master/xwiki/okf/sonarqube):
`index.md` (rule → family-file map, the rules never worth fixing, the universal drop conditions), then
one family file per group — `syntax-rules`, `simplification-rules`, `modernization-rules`,
`dead-code-rules`, `constant-and-resource-rules`, `test-code-rules` — plus `verification.md`. The
`xwiki-fix-sonarqube-issue` skill owns the *procedure* (assert-guarded batch script, match-count
location, subagent verification, the accept loop).

- **READ:** this file first, then `pool-state.md` when picking a rule, then the **OKF** family file
  for the rule you commit to. `dropped-issues.md` is a skip-index of issue KEYS already analyzed and
  rejected — consult it in the find phase and SKIP those keys instead of re-triaging; add every new
  analyzed-but-not-fixed key to it. `token-cost-report.md` is loaded only when asked.
- **WRITE:** a durable rule-correctness learning → **OKF PR** via `xwiki-knowledge`, not here. A pool
  observation → `pool-state.md`. A routine/build/GitHub/process fact → the matching section HERE.
  Merge and trim in place; do not append.

## Picking a target rule (find phase)

- Get the rule distribution cheaply FIRST (no issue bodies): one
  `issues/search?...&issueStatuses=OPEN&facets=rules&ps=1` call returns the whole project rule
  distribution. For an exact per-rule count read the response `total` (query with `&rules=java:SXXXX&ps=1`),
  not a facet value.
- **Pools shift every run** as PRs land — clean rules get exhausted then slowly regenerate. Always
  re-query; never assume a rule still has convertible issues. `pool-state.md` records where they were
  last seen, not where they are.
- If a rule's remaining issues are all non-convertible residue, pivot (skill rule: "if a fix is hard,
  drop it and pick another").
- **Scope/mode overrides.** A run may target only *new-code* issues (add `&sinceLeakPeriod=true`; the
  pool is smaller, ~100 project-wide, but the same families apply) and/or ask for *safe changes only*.
  In safe-only mode stick to the purely mechanical families (syntax / simplification / unused / S1066)
  and drop the judgment-heavy ones (`S2629`/`S3457` logging reformatting, `S6880` if→switch, `S1130`
  remove-`throws`, `S1845`); whenever you're unsure a change preserves behaviour, drop it.
- **The BLOCKER/CRITICAL mechanical pool is frequently exhausted.** BLOCKER/CRITICAL-first is the
  skill's guidance but not a hard gate — a clean MAJOR fix beats forcing a risky higher-severity one.
  There is a deep MAJOR-severity clean pool.
- **Check open agent PRs up front** — `gh api "repos/xwiki/<repo>/pulls?state=open&per_page=100" --jq
  '.[]|select(.labels[]?.name=="llm-agent")|"\(.number) \(.title)"'`. A recent PR can drain a WHOLE
  rule family. Scope the off-limits check by **(rule + module)**, not rule alone: a per-module batch PR
  only claims the files it touched, so the same rule in OTHER (incl. sibling) modules is fair game.
  When your planned rule already has multiple open PRs, PIVOT to a zero-PR family rather than threading
  the gaps. **A same-FILE open PR is off-limits even for a DIFFERENT rule** — a concurrent edit risks
  merge conflicts; drop that site. To learn exactly which modules a wildcard "various modules" PR
  claims, read its file list (`gh api "repos/…/pulls/N/files?per_page=100" --jq '.[].filename'`).
- **Finding the NEXT unswept rule** when the known families are all drained or dropped: pull the broad
  rule-distribution facet, then batch one `ps=2` query per candidate rule and read just the `message` —
  one turn classifies ten rules. Safe mechanical candidates read like S7158 / S1155 / S1602 (one-line,
  zero-dataflow). Then check the candidate against the OKF denylist before committing to it.

## Batch mode ("fix 20-50 / all of rule X" override)

- Mixed issue types in one PR are explicitly allowed — bundling a purely additive rule (`S1161`
  `@Override`) into a removal/simplification batch gives a clean multi-type PR.
- **Reach the target by density-first module selection:** query the rule(s) with `&ps=500` and group by
  module (`component.split(':')[-1]` for the path — the projectKey has TWO colons, so `split(':',1)[1]`
  is WRONG and every file open fails). Then either (a) one dense single module (oldcore often holds
  30-90 of a rule = one cheap build), or (b) a WIDE reactor of many cheap leaf modules when the pool is
  thin-spread. A 30+-module reactor still builds green in ONE shot.
- **When a rule's pool sits below 20 alone, MIX zero-PR pure-mechanical rules** (S1612 + S1125 + S2864
  + S1155 + S1197 + S1128 — all zero-dataflow single-line edits) across ~20 modules into one green
  reactor. Pivoting to zero-PR rules beats threading the gaps in a PR-saturated rule. Reserve mixed
  *dataflow*-rule batches for when unavoidable (each rule multiplies the edit-error surface).
- **Split a large batch into a TABLE-DRIVEN script for the repetitive rule and an exact-string
  script for the long tail.** A 99-site commons PR was three artefacts: a table of
  `(file, flagged line, pattern-variable name)` for the 47 scriptable S6201 sites, a list of
  `(file, old_text, new_text)` triples with `assert count == 1` for the 27 one-off edits across nine
  other rules, and four hand-edits for the shapes with nested scopes or re-wrapped conditions. The
  `count == 1` assertion is what makes the second form safe — a stale or ambiguous snippet fails
  loudly instead of editing the wrong place.
- **Re-run the whole script from `git checkout -- .` after EVERY fix to it.** Three successive bugs
  (a `} else if` brace-count, a `\b` that will not match after `]`, and a paren-greedy cast pattern)
  were each found by reading the compact diff `git diff -U0 | grep '^[+-]'` — a full-file diff is
  too big to scan and hides exactly this kind of damage.
- **A scripted transform beats hand-editing above ~20 sites, but only with these guards** (learned
  converting 53 S6201 sites in one pass; the same guards apply to any regex batch edit):
  - **Assert length ≤120 on the MODIFIED lines only.** A whole-file `all(len(l)<=120)` assert fails on
    pre-existing over-long lines in Sonar-heavy files (which are often Checkstyle-excluded) and
    silently blocks the whole file.
  - **Scope name-collision checks to the enclosing region + the class's field declarations, never the
    whole file** — the license header ("the NOTICE **file**") vetoes `file`, and a name already used in
    *another method* is still free here. Reserve a mangled fallback name (`xValue`, `theX`) for real
    collisions, then normalise the names in a second pass so the reviewer sees idiomatic ones.
  - **When the first pass reveals a script bug, `git checkout -- .` and re-run from clean.** Chasing
    line numbers that earlier deletions have shifted is where the real errors get introduced.
  - **Print an APPLIED/SKIPPED table with a reason per skipped site.** Two of my "safe" skip reasons
    were script bugs, not real drops — a skip list you can read is how you find them (the `} else if`
    scope bug alone hid 12 of 53 sites).
  - **Review the generated `git diff` before building.** One bad regex (a cast pattern matching a
    method-call paren, turning `equals((String) o)` into `equalsstring`) is obvious in the diff and
    invisible in the summary counts.
- **Drive the edit off the issue's `textRange`, not a regex over the flagged line.** `issues/search`
  returns `textRange: {startLine, endLine, startOffset, endOffset}`, which pinpoints the exact
  expression Sonar flagged. Slicing `line[startOffset:endOffset]` and asserting the slice looks like
  what you expect (`expr.startswith('mock(')`) is both simpler and far more robust than pattern-matching
  the line — a line with two `mock(` calls, or an FQN, or a nested call just works. Keep the FQN when
  you re-emit the type (`org.hibernate.query.Query queryMock = mock(org.hibernate.query.Query.class)`):
  a file that spells a type out usually does so because the simple name is already imported from
  another package.
- **When a batch invents variable names, do TWO passes: reserve names top-down, apply edits
  bottom-up.** Editing bottom-up is required (earlier edits must not shift later line numbers), but if
  you also *name* bottom-up the numbered suffixes come out reversed (the first site in the file gets
  `fooMock3`). Reserve in source order into a `taken` set first, then apply in reverse. Prefer one
  consistent scheme (`<type>Mock`, `<type>Mock2`, …) over "plain name, suffix only on collision" —
  the latter makes one file read three different ways.
- **A fix must not introduce a NEW Sonar issue — check the shape you are creating, not just the one you
  are removing.** Recurring traps when a fix *adds* code: a declaration for a generic type written raw
  (`MultivaluedMap x = mock(MultivaluedMap.class)`) trades the fixed issue for `S3740` "raw types" — write
  the type arguments and let the factory infer (`MultivaluedMap<String, String> x = mock();`); and a
  removal that orphans a constant or an import creates `S1068`/`S1128`, so delete those in the same edit.
- **Collect issue keys by a substring of the full component PATH** (`.../xwiki-platform-chart-macro/...`),
  NOT a guessed short module name (silently returns 0). Build the accept list by KEY, not edit count.
- **Don't force the target when the allowlist total is small** — expect a low clean yield even from
  leaf modules, and ship what is genuinely clean.
- **`while read` silently DROPS the last key of a file with no trailing newline** — `python3` writing
  `'\n'.join(keys)` produces exactly that, so a 106-key loop processes 105 and the miss looks like a
  transient API failure. End the file with a newline or iterate in Python; either way re-verify the
  count afterwards.
- **Accept all issues in a loop** (per issue: `add_comment` + `do_transition accept`). Each issue is
  ~2 curls ≈ 4s, so 20+ issues blow a 2-min timeout — run the accept loop as a BACKGROUND task, and/or
  make it idempotent (re-query which keys are still OPEN). **An unthrottled loop silently loses about
  half of them** — a 70-key run left 35 still OPEN with no error output — so the confirm-and-retry pass
  is mandatory, not a safety net: loop `issues/search?issues=<keys>` → re-POST `accept` for anything not
  yet ACCEPTED → repeat until zero, with a ~0.3s sleep between POSTs. `do_transition`'s response does NOT
  reliably contain an `issues` key (don't index it → KeyError; the transition still applied), and a
  `Transition from state RESOLVED does not exist: accept` error on the retry is benign — that issue is
  already closed.

## Find-phase cost

- Inline `sed`/`Read` (offset/limit) of ONE candidate region is cheaper than an Explore subagent for
  mechanical rules; use a subagent only when you must read & reject several candidates. Always trim
  `issues/search` JSON through `python3`/`jq` (keep key,rule,component,line,message) — some rules attach
  huge `flows`/`locations`; never dump raw responses into context.
- Component key = `groupId:artifactId:path` (TWO colons). Path = `component.split(':')[-1]`. Read
  locally at `/home/user/xwiki-platform/<path>`; never fetch file contents remotely.

## Building / verifying (this container)

The *rules* — never `-DskipTests`, `-Plegacy,quality` mandatory, why removing covered instructions
lowers a JaCoCo ratio, how to tell your reactor failure from a pre-existing one — are in the OKF
(`okf/sonarqube/verification.md` and the `xwiki-build` skill). What follows is container-specific.

- **Datapoints for a three-repo chain** (warm `~/.m2`, all green, tests on): whole commons repo
  (130 modules) **17:46**, rendering 3 modules **1:59**, platform **26**-module `-pl` reactor **18:25**
  — ≈38 min end to end for one background run. A 26-module platform reactor is NOT expensive: prefer it
  over dropping thin-spread sites.
- **When a commons batch touches >~25 modules, build the WHOLE repo** (`cd /home/user/xwiki-commons
  && mvn install -Plegacy,quality`) instead of a long `-pl` list: 35 touched modules ran 16:36 with
  2365 tests green, and it sidesteps the `-pl`-subset risk around `xwiki-commons-tool-*` modules that
  supply Checkstyle/Spoon rules as *plugin* dependencies to later modules. Rendering (2 modules) and
  platform (13 modules) stay on `-pl`: 1:26 and 9:35. Whole three-repo chain ≈ 28 min.
- **A repo-wide cleanup wave landed in the last day or two is the likeliest cause of a build failure
  you did not cause.** Before assuming your batch broke something, check the failing file's history
  (`git log -1 --format='%h %ad %s' --date=short -- <file>`) and confirm your diff does not touch it
  (`git diff origin/master..HEAD -- <file>` / compare the failing line numbers with your hunk headers).
  Two independent failures in one run both traced back to the same "logging best practices" commit
  from the previous day: a `MultipleStringLiterals` Checkstyle error in an untouched `src/main` file,
  and a test in *another module* asserting a log message that the wave had reworded. **A test-only
  batch cannot cause a Checkstyle error in `src/main` or an assertion failure in a test you did not
  edit** — that reasoning is enough to keep the fixes rather than dropping the module.
- **Do not repair unrelated master breakage inside a Sonar cleanup PR.** Ship the batch, and state the
  breakage precisely in *Clarifications*: the offending file and line, the commit that introduced it,
  the evidence that your diff is elsewhere, and an offer to rebase once master is green. Fixing it
  would muddle the review and can conflict with whoever is already repairing it.
- **`-Dcheckstyle.skip=true` does NOT disable XWiki's Checkstyle gate** — the plugin configuration
  wins over the user property and `checkstyle:check` still fails the build. Useful consequence: you
  cannot skip past a pre-existing violation. The tests still run *before* Checkstyle, so a failed run
  of this kind still gives you the full `Tests run:` figures — grep them; they are the real
  verification for a test-only batch.
- **Use `-fae` (fail-at-end) when one module is expected to fail for a pre-existing reason.** Without
  it a single early failure marks every later module `SKIPPED` and you learn nothing about the rest of
  the reactor; with it every module runs and you can check that the only failures are the known ones.
- **A "module is red on master" drop goes STALE — re-probe it before writing off its pool.**
  `mvn package revapi:check -Pquality -DskipTests -pl <module>` costs ~2 min and settles it.
  (`xwiki-commons-extension-api`'s recorded revapi failure was fixed upstream; that one probe
  re-opened ~100 issues.) Same idea for a compile question you can answer in seconds: write a 10-line
  file and run plain `javac` on it rather than guessing (that is how the `doPrivileged` lambda
  ambiguity was settled).
- **Do NOT use the `snapshot` profile** — it was dropped from the org build recipe. Transitive
  `X.Y.0-SNAPSHOT` deps resolve from the local `~/.m2` and the standard XWiki repos. A fully cold
  `~/.m2` does NOT need `-am` — a ~30-module `-pl` reactor resolves every SNAPSHOT sibling as a
  downloaded jar and builds green.
- **Test-file-only batch: drop a heavy dependency module from `-pl`** and let a dependent pull it as a
  remote SNAPSHOT. For a batch that ONLY edits `src/test`, a module you'd otherwise build for 1-2 sites
  but whose test suite is enormous (oldcore) is bad ROI — leave it out of `-pl` entirely and drop that
  module's conversions. (`-Pquality`'s `jacoco:check` runs the FULL module suite, so you cannot
  `-Dtest=` your way to a fast partial verify; excluding the module is the only lever. The exception is
  a batch where ONLY test code changed — then `-Dtest=<flagged classes> -DfailIfNoTests=false` runs
  exactly the tests that can break.)
- **Always `cd` explicitly before EVERY per-repo command, mvn or not** (`cd /home/user/xwiki-platform
  && mvn …`): the shell cwd can silently reset to `/home/user` between turns, and within one call it
  persists into the next command. This bites `git commit` in a three-repo chain exactly as it bites
  `mvn` — a second `git add -A && git commit` with no `cd` runs in the FIRST repo and reports "nothing
  to commit", which reads like "already committed" — so verify each one with `git -C <repo> log -1`
  (and `git -C <repo> status --porcelain`), never by reading the commit command's own output. **Write a
  multi-repo chain to a SCRIPT FILE and run
  `bash build.sh`, never as an inline heredoc.** In a chain, `cd repoA && mvn …` newline `mvn …` leaves
  the SECOND mvn in repoA — and re-typing a long heredoc to fix one `cd` reproduces the bug verbatim
  (it happened three times in a row). A file you can `Edit` makes the per-`mvn` `cd` visible and fixable.
- **Chain the multi-repo builds with plain newlines, not `&&`** — a failure in repo A must not prevent
  repos B and C from building, since each ships its own PR.
- **Session plugin cache can be STALE vs the xwiki-dev-llm source.** The build recipe, profiles and the
  OKF are authoritative in the plugin *repo*, which may be several versions ahead of the cached plugin
  loaded this session. When a reviewer cites "the latest plugin says X", re-read the source repo.
- Run the build in the **background**, letting the tool capture stdout to its own `tasks/<id>.output`.
  Do NOT add your own `> build.log` redirect, do NOT `nohup … &`, NO `| tail` (a `| tail -N` on the
  backgrounded `mvn` DISCARDS all but the last N lines from the captured file — you then can't grep
  `Tests run:` or failing-test names). The completion notification carries the exit code; grep the full
  `tasks/<id>.output` for `BUILD SUCCESS` afterwards.

## Cost control (the build wait dominates the bill — but in TIME, not tokens)

- The token bill is driven by how many TURNS happen while the build runs, NOT how long it takes.
  Running the tests costs ~the same tokens PROVIDED you keep this discipline: once the build is
  running, STOP — one line, end the turn. Do NOT Read/grep the log while it runs; the completion
  notification wakes you (one wake-up turn re-reads the cached context once, whether the build took
  3 min or 25). Don't arm a short `ScheduleWakeup` for a build; if you want a fallback use 1200s+.
- Tests add real tokens in exactly one case: a test FAILS and you must diagnose/fix/rebuild — which is
  the regression you WANTED to catch. On clean mechanical fixes this is rare.
- **Holding the turn open during a build avoids the "uncommitted changes" stop-hook ping-pong.** Ending
  the turn while a background build runs trips that hook every time. Arming a `Monitor` with
  `until grep -qE "BUILD SUCCESS|BUILD FAILURE" <task output>; do sleep 15; done` keeps the turn alive
  for ~one extra turn's cost and reads the outcome in the same wake-up. Never commit unverified just to
  silence the hook.
- **A cold `~/.m2` spends 10+ minutes downloading the plugin tree before the first `Compiling`
  line.** A build log that shows only `Downloading from xwiki-releases:` is healthy, not stuck —
  don't re-launch it. The whole three-repo chain (4 commons + 3 rendering + 7 platform modules) ran
  green from cold in one pass.
- **Commit locally (do NOT push) while the verification build runs.** That silences the
  uncommitted-changes stop hook without shipping anything unverified: if the build goes red you
  amend, and the push + PR only happen after `BUILD SUCCESS`. This is strictly better than either
  ending the turn dirty or pushing early.
- **A container restart can kill the background build mid-run.** Working-copy edits PERSIST
  (uncommitted) — don't re-do the fix; check `git status`/`git diff`, confirm the branch, re-launch the
  same `mvn` build. Don't panic-commit unverified.

## GitHub (this container — `gh` porcelain does NOT work here)

- **Why not `gh` porcelain:** the `gh` binary may be installed, but `gh pr create/list/view` and
  `gh issue` FAIL here for two independent reasons: (1) the session proxy BLOCKS GraphQL (`HTTP 403:
  This GraphQL query is not enabled for this session`); (2) the git remote points to a local proxy
  (`127.0.0.1:.../git/...`), not github.com, so repo-context commands can't resolve the repo.
  `gh auth status` also mis-reports the token invalid. **`gh api` with a REPO-SCOPED REST path DOES
  work** (`GH_TOKEN` is in-env) and is the CHEAPEST channel for everything with a big response —
  cross-repo `gh api search/issues` is BLOCKED though (`sessions are bound to their configured
  repositories`), so scope every path as `repos/{owner}/{repo}/...`.
  (The skill says "open the PR with `gh pr create`" — that is correct for a normal dev machine; here,
  use `gh api` REST as below.)
- **Prefer `gh api` REST over the GitHub MCP tools whenever the response or request body is large** —
  MCP results land in context verbatim and `--jq` does not:
  - Open agent PRs per repo: see *Picking a target rule*. `search_pull_requests` dumps every PR *body*
    (a batch PR body is huge); `list_pull_requests` is ~660KB. Both are last resorts.
  - **NEVER call `pull_request_read` `get_files`** — it returns the FULL PATCH of every file (tens of
    KB). Use `gh api "repos/…/pulls/N/files?per_page=100" --jq '.[].filename'`.
  - Create the PR from a FILE so a long body never passes through context:
    `gh api repos/{o}/{r}/pulls -X POST -f title=… -f head=<branch> -f base=master -F body=@pr.md
    --jq '.number,.html_url'`. Same for editing it (`-X PATCH -F body=@pr.md`).
  - Label + assignee in one call: `gh api repos/{o}/{r}/issues/{n} -X PATCH -f 'labels[]=llm-agent'
    -f 'assignees[]=vmassol'`. Lock (Vincent's override): `gh api --method PUT
    repos/{o}/{r}/issues/{n}/lock -f lock_reason=resolved`.
- **Vincent's PR lock blocks YOUR OWN comments too** (`403 issue is locked`), so a reviewer reply needs
  `DELETE …/issues/{n}/lock` → post the comment → `PUT …/issues/{n}/lock -f lock_reason=resolved` again.
  Do the unlock/re-lock in ONE command so the window stays short. Reviewers can still leave reviews on a
  locked PR, so expect to need this.
- **All three repos ship `.github/pull_request_template.md`** — mirror its headings (`# Jira URL`,
  `# Changes` / `## Description` / `## Clarifications`, `# Screenshots & Video`, `# Executed Tests`,
  `# Expected merging strategy`). For a Sonar sweep: "None — this is a `[Misc]` SonarQube cleanup
  commit" under Jira URL, the per-rule table under Description, the *dropped* sites and why under
  Clarifications, the exact `mvn` line + test counts under Executed Tests, and squash / no backport.
- Creating the PR auto-subscribes the session to PR webhooks. XWiki CI is Jenkins and reports later, so
  `get_status` is `pending`/`total_count:0` right after creation — NOT a failure. Webhooks don't deliver
  CI-success / new-push / merge-conflict transitions; for long watches schedule a ~1h self check-in and
  re-arm silently until merged/closed. Reply to reviewers only when genuinely necessary.
- **Handling a reviewer objection** (verify the mechanism, judge whether the objection is about intent
  clarity, withdraw rather than argue, then ship the `@SuppressWarnings` + rationale version as its own
  PR and reopen the issues) is in the `xwiki-fix-sonarqube-issue` skill. Note the dev.xwiki.org
  JavaCodeStyle page sits behind Cloudflare (403 to both WebFetch and curl) — derive the convention by
  grepping the repos, or read `okf/conventions/code-style.md`.

## Process / conventions

- Commit + PR title (no JIRA): `[Misc] <desc; mention SonarCloud/SonarQube>`. Security issues: keep the
  description cryptic (public logs).
- **Multi-repo runs ("also check xwiki-commons / xwiki-rendering") depend on the session's scope.**
  Each sibling repo needs BOTH a local clone AND that repo in the session's GitHub access scope. A
  session scoped to `xwiki/xwiki-platform` only can't touch them — do the platform work and report the
  scope limit; don't try to clone out-of-scope repos (the proxy blocks them). Each sibling repo also has
  its OWN designated feature branch and its own `SONARQUBE_PROJECT_KEY`.
- **Multi-repo mechanics when all three ARE in scope** (the cheapest way to blow past a 30-fix target,
  since ONE rule usually has a pool in each repo):
  - `SONARQUBE_PROJECT_KEY` is only set for the session's primary repo. The siblings' keys are
    `org.xwiki.commons:xwiki-commons` and `org.xwiki.rendering:xwiki-rendering` (enumerate with
    `api/projects/search?organization=xwiki&ps=50`). Query each with `componentKeys=<key>`.
  - Do the whole find/apply phase for all three repos FIRST (cheap, no builds), then build them
    **SEQUENTIALLY in dependency order commons → rendering → platform**. They share one `~/.m2`, so
    concurrent `install`s race; and building commons first means rendering/platform verify against your
    *modified* commons jars rather than a downloaded SNAPSHOT. Chain all three in ONE background
    subshell with an explicit `cd /home/user/<repo> &&` before EACH `mvn` and an
    `echo ###### <REPO> ######` marker between them, then grep the markers + `BUILD SUCCESS`.
  - Ship each repo as its own commit + branch + PR (label + assignee + lock each). Cross-link the
    sibling PRs in a "Related" section so a reviewer sees it is one sweep.
  - Check open agent PRs per repo — the siblings are usually at 0.
- **Separate-PR override (safe vs unsure):** when asked to isolate risky fixes, ship the safe mechanical
  batch on the designated branch, and put a judgment-heavy family (e.g. S6126 text blocks, S8714
  assertThrows) on a SIBLING branch (`<designated>-<rule>`) as its own PR, so a reviewer can merge the
  easy PR without the hard one blocking it. Both PRs still get the label/assignee/lock treatment.
  - **Verify the pair in ONE reactor, then split by file.** Apply both batches to the same working
    tree, build once, and only then separate them: commit batch A (`git add <its files>`), commit
    batch B (`git add -A`), then `git checkout -B <sibling> <masterSha> && git cherry-pick <commitB>`
    and `git checkout <designated> && git reset --hard <commitA>`. Legitimate because the two file
    sets are disjoint — say so in both PR bodies ("the reactor also contained the sibling branch's
    changes"). Halves the build bill versus verifying each branch separately.
  - `git cherry-pick` has **no `-q` flag** (it prints usage and the branch silently stays at base).
    Redirect its output instead, and if a commit goes dangling, recover the SHA from `git reflog
    --format='%H %gs'`.
- **Author override:** `git config user.email <email>` AND `git commit --author="Name <email>"` AND a
  `Co-Authored-By: Name <email>` trailer — verify with `git log -1 --format='%an <%ae>'`. (This
  routine's override email differs from the git userEmail context — use the override.)
- **Reset the designated feature branch to master FIRST — it persists across runs.** `git fetch origin
  master` then `git checkout -B <branch> origin/master` before editing, or the new PR bundles old
  already-merged commits. **The local `origin/master` ref can LAG even right after `git fetch origin
  master`.** `git ls-remote --heads origin` is the cheapest ground truth (one call, no API scope
  worries); the GitHub API (`list_commits` `sha=master`, read `[0].sha`) also works. **Check BEFORE
  the `-B`, not after** — resetting to a stale `origin/master` silently deleted a whole tracked
  directory from the working copy (the entire `okf/sonarqube/` corpus), and it only looked like "the
  files were never committed". Recover from `git reflog`, then re-point at the real HEAD.
  If the branch tip EQUALS the real master HEAD, the prior run's PR was
  already MERGED (squash-merged, so the local commits look distinct) → `git checkout -B <branch>
  <realMasterSha>` and start fresh. If it's genuinely ahead, rebase the unmerged commits onto the real
  HEAD.
- **Recording learnings (memory repo → `main`).** The xwiki-platform fix lives on a feature branch but
  learnings go to this repo's `main`. Do NOT edit on the feature branch then stash/checkout/pop (main
  has diverged; the pop bakes `<<<<<<<` markers into the commit). Instead `git checkout main && git pull
  origin main` FIRST, then edit and commit directly on main. Route the learning per the WRITE protocol
  at the top — durable rule-correctness facts go to the OKF as a PR, not here.
