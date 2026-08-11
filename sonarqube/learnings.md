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
- **For a STRUCTURAL rule, locate sites by scanning for the code SHAPE and assert `found == issue
  count` per file.** For `S8714` that was "every `try {` block whose body contains `fail(`", printed
  with its full extent; 21 files matched 49-for-49, which proves up front that no site is missed and
  no line-number drift matters. Then feed each printed block verbatim into the `(file, old, new)`
  table. This generalises to any rule whose shape is greppable — it is the structural counterpart of
  the per-file match-count technique and it needs no snippet reads beyond the blocks themselves.
- **When a batch INTRODUCES a local declaration per site, two sites in the same method collide.**
  `T e = assertThrows(...)` twice in one test method is a duplicate-variable compile error; the
  compiler would catch it, but a 20-second post-apply scan (walk each `    {` … `    }` method body,
  flag a repeated `^        <Type> <name> =`) finds it before the 20-minute build. It fired on two of
  21 files. Fix by declaring once and assigning after, keeping a distinct name for the odd type out.
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
- **A fix that NARROWS the set of checked exceptions a call can throw breaks the enclosing
  `catch`.** Java rejects a `catch` for a checked exception the `try` can no longer throw, so the
  compile fails. It bit S4719 three times in one batch (`getBytes(String)`/`IOUtils.toString(…,
  String)` → the `Charset` overload kills `UnsupportedEncodingException`). Before applying such a
  fix, look at the enclosing `try`: if that exception is the ONLY thing the body throws, the fix
  includes deleting the dead `try`/`catch` and dedenting the body — which is a good change (the
  removed instructions are uncovered, so coverage goes UP), just not a one-liner. If the `catch`
  also covers something else (`catch (IOException)` around a `getOutputStream()`), it stays. The
  same shape applies to any rule that swaps a String-typed API for a typed one.
- **An "unnecessary" cast on an argument of an OVERLOADED method is not a mechanical fix** — removing
  it can silently re-dispatch to a different overload and still compile. Check the callee's overload
  set before trusting S1905.
- **A fix must not introduce a NEW Sonar issue — check the shape you are creating, not just the one you
  are removing.** A removal that orphans a constant or an import creates `S1068`/`S1128`, so delete those
  in the same edit; a declaration for a generic type should carry its type arguments and let the factory
  infer (`MultivaluedMap<String, String> x = mock();`) rather than being written raw. And **wrapping code in a lambda
  re-scopes every throwing expression inside it**: an `assertThrows` around `print(new URL("…"))` holds
  two checked-exception throwers and lands an `S5783` BUG on *new code*, invisible until the next scan.
  So after a batch merges, re-query the project once with `&sinceLeakPeriod=true&types=BUG` — a green
  build does not prove a clean sweep, only that nothing broke.
- **"Would this create a new issue X?" is answered by the project's PROFILE, not the rule catalogue.**
  `api/qualityprofiles/search?organization=xwiki&project=<key>` → the `java` profile key (XWiki's is
  *XWiki Java*, ~609 active rules) → `api/rules/search?organization=xwiki&qprofile=<key>&activation=true&rule_key=java:SXXXX`
  (`total` 1 = active, 0 = not enabled, so it can never fire). Worth knowing: **`S3740` (raw types) and
  `S6212` (type inference / `var`) are NOT enabled** for XWiki, so neither a raw declaration nor a
  class-literal factory call can be "trading one issue for another" — argue those on merit instead.
  A cross-check from the other direction: a rule with 0 open issues project-wide on a shape that occurs
  thousands of times is not enabled. `api/rules/show?organization=xwiki&key=java:SXXXX` gives the rule's
  own compliant example, which is the strongest possible answer to a reviewer asking "shouldn't the fix
  look like Y instead?".
- **Classify sites by the ANNOTATION above the flagged line — and by `private` — before batching a
  signature-level rule.** For `S1130` this turned a 294-issue pool into a provably-safe subset in one
  pass with no snippet reads: walk up from the flagged line over `@…`/comment/blank lines and bucket
  the site as `@Test`/`@BeforeEach`/`@AfterEach`/`@BeforeComponent` (safe — never overridden, never
  called cross-module), `@Override` (needs a parent check) or un-annotated. **An un-annotated site is
  only a drop when it is not `private`**: a `private` helper cannot be overridden by a `test-jar`
  consumer in another module, so it is as safe as an annotated test method (18 of 22 "non-annotated"
  sites were recovered this way). The same bucketing applies to any rule flagged on a method
  declaration. For a rule that fires on a *declaration of any kind* the cheapest split axis is
  **what is being declared**: `S117`'s 45 platform sites divided, with no source reading beyond the
  flagged line, into 28 local variables (pure mechanical, one PR) and 17 method parameters of public
  legacy APIs (visible in Javadoc/IDE, so a judgement call → the sibling PR). Deriving the safe/unsure
  split from the issue's own `textRange` line is what makes the two-PR override cheap.
- **A batch that RE-FLOWS a statement onto one logical line must refuse to touch comments.** Joining
  the statement's lines is the cleanest way to re-wrap after substituting a short variable for a long
  expression, but it silently destroys any `//` comment caught in the range: the comment swallows the
  rest of the line. Two guards, both needed — when walking back over continuation lines to find the
  statement start, **stop at a line beginning `//`, `*` or `/*`**, and assert the collected statement
  text contains no comment marker. Without the first guard a comment ending in a full stop reads as a
  continuation (`.` is in any sane continuation-character set) and is pulled in; the damage compiles
  as a syntax error, so it costs a whole build round rather than being caught by the assertions.
- **The flagged line is not always the STATEMENT start — for some rules it is the middle.** The skill
  warns that Sonar attributes a multi-line statement to its start line; the converse also happens
  (`S5778` is flagged on the line holding the lambda, while `T x =` sits on the line above). A fix
  that INSERTS a line before the flagged line then splits the assignment. Walk back over continuation
  lines (previous line ends `= ( , -> + && || ? :`) before deciding where the statement begins, and
  eyeball the dry-run diff for a removed line ending in `=`.
- **Sonar's message names the type FULLY QUALIFIED; the source almost always uses the simple name.**
  A batch that matches `message` tokens against source tokens must compare on `name.split('.')[-1]`
  on BOTH sides. Get this wrong and the edit is a **silent no-op that still produces a diff** — the
  script rebuilds the clause it was supposed to shrink, and the only visible symptom is an
  unexplained re-wrap plus a suddenly over-long line. The guard that catches it: assert
  `len(removed) == len(wanted)` per site and print a `!! mismatch` line, then re-run from
  `git checkout -- .`.
- **XWiki's brace style puts `{` on the NEXT line, so a declaration regex anchored on `…) {` matches
  ZERO sites.** A batch keyed to `throws\s+([^{;]+?)\s*\{\s*$` skipped all 155 S1130 sites and the
  skip list read like a data problem. Make the trailing `{` optional (`(\{)?\s*$`) and re-emit it only
  when it was there. Corollary for any signature-level rule: the flagged line usually ENDS the
  declaration, and the body brace is the line after.
- **A RENAME batch must never rewrite a `this.<name>` access.** When the flagged parameter shadows a
  field of the same name (very common in oldcore setters — `this.engine_context = engine_context;`),
  a plain word-boundary substitution over the method scope renames the field reference too and the
  file stops compiling. Use a `(?<![\w$])(?<!this\.)name(?![\w$])` pattern so the field keeps its own
  name; the resulting `this.engine_context = engineContext;` is correct and leaves the (separately
  flagged) field alone. The converse guard matters as much: assert the NEW name occurs **zero** times
  in the rename scope, which is what rules out a renamed local silently capturing a reference that
  used to resolve to a field.
- **A line-length guard must compare against the SET of original lines, not zip by index.** As soon as
  one edit re-wraps a statement onto two lines, every later line pairs with the wrong original and the
  guard fires on pre-existing over-long lines. `for line in new: if line not in set(old_lines): assert
  len(line) <= 120` is index-independent and still catches exactly the lines you wrote.
- **Check the flagged line's own git history before DELETING anything — a rationale is not always a
  comment, sometimes it is a commit message.** The universal drop conditions say to respect a comment on
  the flagged code; that is not enough. A run deleted a `.toString()` from a log call whose `toString()`
  had been *added on purpose* four days earlier by `64ba541` (XWIKI-24665) — **the direct parent of the
  branch, printed in the `git log -1` output while the branch was being set up** and read straight past.
  The repo had already oscillated on those lines (#1871 stripped them, #1872/#1884 restored them), so the
  sweep re-fought a settled decision. Guard, cheap enough for any deleting rule: loop
  `git log -1 --format='%h %s' -- <file>` over `git diff --name-only`, and when a subject is recent and
  JIRA-numbered, `git show <sha> -- <file>` to check it did not introduce the shape you are removing.
- **Consult `dropped-issues.md` for EVERY rule you shortlist, before reading ANY source — one grep of
  all the shortlisted keys, not one per rule you commit to.** This has now cost source reads twice:
  `S6035`/`S2093` in one run, `S1118`/`S3415` (rendering) in another, all four already recorded with
  the exact reason. The check is one call for the whole shortlist; skipping it is what makes the
  skip-index worthless.
- **Re-derive the fixed-issue COUNT from the applied edit table at PR time — never carry a planning-phase
  tally forward.** The count drifts every time a site is added or dropped during triage, and the stale
  number reaches the PR body, the commit message and the accept list at once. A run shipped "21" when the
  script had applied 28 (a `java:S125` file added after the tally was written). Compute it from the same
  structure that drives the edits — group the edit list by rule and print the histogram — and build the
  accept list from those same keys, so the PR body and the SonarCloud transitions cannot disagree.
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
  half of them** — a 70-key run left 35 still OPEN with no error output — whereas a **0.3s sleep after
  EVERY POST** (comment and transition alike) landed 66/66 with nothing left for the retry pass. So
  throttle from the start and still run the confirm pass (**168/168 landed on the first pass, zero
  stragglers**, but budget ~30 min of wall clock for 168 keys through this container's proxy — launch
  it BEFORE the memory write-up, not after, and it comfortably covers the whole write-up plus the OKF PR): loop `issues/search?issues=<keys>` → re-POST
  `accept` for anything not yet ACCEPTED → repeat until zero. `do_transition`'s response does NOT
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

- **Datapoint for a very wide platform reactor**: **49 modules** (incl. oldcore and web-templates),
  test-only signature edits, **25:10** with **3134 tests** green; the commons single-module leg
  (`extension-api`) alongside it was **3:13** / 223 tests. A 49-module `-pl` list is entirely
  practical — building one reactor for 58 thin-spread files beat every alternative.
- **Datapoint for a wide test-only sweep**: commons 4 modules **3:33** (523 tests) + rendering 2
  modules **1:11** (440) + platform **12** modules incl. oldcore **9:59** (2156) = **~15 min** for the
  whole three-repo chain, 3119 tests green. A 12-module platform reactor with oldcore in it is cheap —
  do not drop thin-spread sites to avoid it.
- **Datapoints for a three-repo chain** (warm `~/.m2`, all green, tests on): whole commons repo
  (130 modules) **17:46**; commons 12-module `-pl` **5:37** (1012 tests); rendering 2-4 modules
  **0:44**-**2:10**; platform **32**-module `-pl` reactor **13:03** (2181 tests), a 26-module one
  **18:25**, 12 modules incl. oldcore **6:24**, 3 leaf modules **2:23**. A wide platform reactor is
  NOT expensive: prefer it over dropping thin-spread sites.
- **When a commons batch touches >~25 modules, build the WHOLE repo** (`cd /home/user/xwiki-commons
  && mvn install -Plegacy,quality`) instead of a long `-pl` list: 35 touched modules ran 16:36 with
  2365 tests green, and it sidesteps the `-pl`-subset risk around `xwiki-commons-tool-*` modules that
  supply Checkstyle/Spoon rules as *plugin* dependencies to later modules. Rendering (2 modules) and
  platform (13 modules) stay on `-pl`: 1:26 and 9:35. Whole three-repo chain ≈ 28 min.
- **After a COMPILE error, rebuild only the failed module plus the ones the reactor skipped behind
  it.** The modules that already reported `Tests run:` in the failed run are green for sources you
  have not touched since, so re-verifying them is pure wall clock (a 3-module re-run replaced an
  11-module one). Confirm the claim first with `git diff` between the two applications of the batch —
  if the second application changed only the failing file, the earlier greens still hold.
- **A repo-wide cleanup wave landed in the last day or two is the likeliest cause of a build failure
  you did not cause.** Before assuming your batch broke something, check the failing file's history
  (`git log -1 --format='%h %ad %s' --date=short -- <file>`) and confirm your diff does not touch it
  (`git diff origin/master..HEAD -- <file>` / compare the failing line numbers with your hunk headers).
  Two independent failures in one run both traced back to the same "logging best practices" commit
  from the previous day: a `MultipleStringLiterals` Checkstyle error in an untouched `src/main` file,
  and a test in *another module* asserting a log message that the wave had reworded. **A test-only
  batch cannot cause a Checkstyle error in `src/main` or an assertion failure in a test you did not
  edit** — that reasoning is enough to keep the fixes rather than dropping the module.
- **`-fae` does NOT save the modules DOWNSTREAM of a failure — plan the drop before the build.** A
  single `jacoco:check` failure on a small early module (`eventstream-api`, 0.73 → 0.72) skipped 17
  of 32 platform modules including oldcore, costing a full second reactor. When a batch removes
  **covered** instructions from a module that contributes only one or two fixes, drop that module
  from the change up front rather than discovering it at minute 5.
- **A module whose `src/main` is essentially a helper library has no meaningful test suite of its
  own — add its CONSUMER to the reactor.** `xwiki-rendering-test` ran 2 tests; adding
  `xwiki-rendering-integration-tests` ran 1532 and is what actually exercises `TestDataParser`.
  Grep for the changed class across `*/src/test` to find the consumer.
- **`checkstyle:check` runs AFTER the tests, so a pre-existing `src/main` violation still gives you
  full test verification.** A test-only oldcore batch failed the module on two `DeclarationOrder`
  violations in an untouched `RegisterAction.java` while its 1140+45 tests ran green — that is enough
  to ship a test-only change. Note the failing module also SKIPS everything downstream of it in the
  reactor even with `-fae`, so re-run the skipped modules as their own `-pl` list (they resolve the
  failed module as a remote SNAPSHOT and build fine).
- **`git diff origin/master..HEAD` is NOT proof of what your commit touches** — the local
  `origin/master` ref lags (it was 1 commit behind the real HEAD here, so the diff pulled in unrelated
  files and made an untouched file look like mine). Worse: **these clones are SHALLOW** (~94 commits),
  so once a concurrent session fetches a newer `master` the two histories share no visible ancestor and
  even the three-dot `origin/master...HEAD` form invents a merge base — a follow-up batch edit keyed to
  it matched 82 sites in 50 files instead of 33 in 22, including pre-existing code. **Key every
  follow-up edit to your own commit** (`git show <sha> -U0`) and assert the site count. Before merging
  master you must first `git fetch --deepen=250 origin master` so a real merge-base exists (`.git` is
  only ~77 MB, so this is cheap); `git merge-base HEAD origin/master` returning your base is the
  green light.
- **Derive test counts by pairing `[INFO] Building <module>` with that module's `Tests run:` summary,
  never by slicing the log into line ranges.** A chained three-repo log sliced at the `###### REPO ######`
  marker over-counted platform 3× (2951 instead of 949 — the slice still held commons and rendering),
  and that wrong figure reached a PR description before it was caught. Same regex both times, so the
  cross-check that catches it is per-module pairing, not a second sum.
- **Do not repair unrelated master breakage inside a Sonar cleanup PR.** Ship the batch, and state the
  breakage precisely in *Clarifications*: the offending file and line, the commit that introduced it,
  the evidence that your diff is elsewhere, and an offer to rebase once master is green. Fixing it
  would muddle the review and can conflict with whoever is already repairing it.
- **Checkstyle METRIC rules are a drop condition, and the 120-column rule is not the only one.** Any fix
  that ADDS a statement or LENGTHENS a boolean expression can be rejected by a metric the line-length
  guard cannot see: **`BooleanExpressionComplexity` (max 3 operators)** and **`ExecutableStatementCount`
  (max 30 per method)** both fired in one run. They run in `checkstyle:check` *after* the tests, so each
  one costs a whole build round. Pre-check them in the apply script, where it is nearly free: for a
  merge-two-branches fix count the `&&`/`||` in the merged condition and refuse >3; for a hoist-an-
  expression fix count the statements already in the target method. When one fires, **drop the site** —
  splitting the method or re-nesting the condition is a refactor, not a Sonar cleanup — and say so in the
  PR's *Clarifications*. Useful side effect: a metric rejection is the codebase stating that the merged
  form is NOT more readable, which is a far better answer than a reviewer's opinion.
- **`-Dcheckstyle.skip=true` does NOT disable XWiki's Checkstyle gate** — the plugin configuration
  wins over the user property and `checkstyle:check` still fails the build. Useful consequence: you
  cannot skip past a pre-existing violation. The tests still run *before* Checkstyle, so a failed run
  of this kind still gives you the full `Tests run:` figures — grep them; they are the real
  verification for a test-only batch. **`-Denforcer.skip=true` is overridden the same way** — but the
  enforcer runs *before* compilation, so an enforcer failure yields NO verification at all (the module
  dies in under a second). There is no way to skip past it; fix the dependency graph instead.
- **An Enforcer `RequireUpperBoundDeps` failure on a stale branch is not yours — merge `master`.** The
  rule reads POMs only, so no source edit can cause it. It appears when the branch base predates a
  dependency-version bump on master while the freshly published sibling SNAPSHOT (built from today's
  master) already requires the newer version (seen as `jsqlparser:4.6 (managed) <-- 5.3` in
  `refactoring-api`, on a base two days old). Confirm with `git show --name-only HEAD | grep pom.xml`
  (empty ⇒ not yours), then merge current `master` into the branch — that is a base update, not
  "repairing master breakage in a cleanup PR", and it keeps every gate enabled. Prefer **merge over
  rebase** when the PR already has review comments: a force-push marks anchored threads outdated.
- **Use `-fae` (fail-at-end) when one module is expected to fail for a pre-existing reason.** Without
  it a single early failure marks every later module `SKIPPED` and you learn nothing about the rest of
  the reactor; with it every module runs and you can check that the only failures are the known ones.
- **A "module is red on master" drop goes STALE — re-probe it before writing off its pool.**
  `mvn package revapi:check -Pquality -DskipTests -pl <module>` costs ~2 min and settles it.
  (`xwiki-commons-extension-api`'s recorded revapi failure was fixed upstream; that one probe
  re-opened ~100 issues.) Same idea for a compile question you can answer in seconds: write a 10-line
  file and run plain `javac` on it rather than guessing (that is how the `doPrivileged` lambda
  ambiguity was settled). **The same throwaway file settles EQUIVALENCE questions, not just compile
  ones** — for a text-block conversion (`S6126`), put the old concatenation and the new text block in one
  `main` and print `old.equals(new)`. Incidental-indent stripping, trailing-whitespace removal and
  `\r\n` are exactly the traps that compile cleanly and silently change the string, and three
  conversions were confirmed byte-identical this way in under a minute.
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
  It bites `git push` too, and there the symptom is misleading: a chained
  `cd repoA && git push -u origin branchA` newline `git push -u origin branchB` runs the second push
  **in repoA** and fails with `error: src refspec branchB does not match any`, which reads like a
  missing branch rather than a wrong directory. **Write a
  multi-repo chain to a SCRIPT FILE and run
  `bash build.sh`, never as an inline heredoc.** In a chain, `cd repoA && mvn …` newline `mvn …` leaves
  the SECOND mvn in repoA — and re-typing a long heredoc to fix one `cd` reproduces the bug verbatim
  (it happened three times in a row). A file you can `Edit` makes the per-`mvn` `cd` visible and fixable.
  **The script file is not the fix by itself — the CHECK is.** It happened again *inside* a script
  file: `cd repoA && mvn …` on line 2, a bare `mvn` on line 4. Before running any multi-repo script,
  grep it: every `mvn` line must begin with its own `cd /home/user/<repo> &&`. The failure is silent
  and total (`Could not find the selected project in the reactor` + a Develocity Groovy NPE from the
  *other* repo's `.mvn/`), and it costs a whole build round.
- **Chain the multi-repo builds with plain newlines, not `&&`** — a failure in repo A must not prevent
  repos B and C from building, since each ships its own PR.
- **Read the OKF from the SOURCE REPO, not the session cache, before deciding any fix.** This is the
  single most expensive mistake this routine has made: the cached plugin was **1.0.4** while master was
  **1.0.16**, and `okf/conventions/logging.md` — added four days earlier and the authoritative answer
  for `java:S2629` — **did not exist in the cache at all**. The fix was shipped, merged into a PR body
  arguing the exact reasoning that file pre-emptively rejects, and had to be withdrawn. `git -C
  /home/user/xwiki-dev-llm pull` costs one call; do it in the find phase, not only when writing the
  OKF PR.
- **Rule knowledge is NOT all under `okf/sonarqube/`.** The skill routes to `index.md` + one family
  file, so a rule owned by a *convention* file is invisible from that entry point — `S2629` was in
  `okf/conventions/logging.md` and referenced from nowhere in the Sonar corpus. Before fixing a rule
  that is about a **convention** rather than a code shape (logging, naming, comments, deprecation),
  grep the whole `okf/` tree for the rule key, not just `okf/sonarqube/`.
- **When the fix's justification is a claim about a LIBRARY's behaviour, check what XWiki wraps around
  that library.** "SLF4J calls `toString()` itself, lazily" is true of SLF4J and false of XWiki: a job
  captures the `LogEvent` with its raw `Object[]` and XStream-serializes it into the job log. The
  giveaway was in plain sight and was not read — the class was `AbstractInstallPlanJob`, the field was
  a job's `this.logger`, and a sibling call in the same file already wrote the eager String on purpose.
  Generalise: *whose* code consumes what you are changing, not just what the JDK/library contract says.
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
- **This container's stop hook demands `noreply@anthropic.com` and conflicts with the author
  override.** It fires at the end of EVERY turn once a commit exists, asking you to
  `--reset-author`. Do NOT obey it — the routine's `vincent@massol.net` override wins and the
  resulting "Unverified" badge is the accepted cost. Acknowledge it in one line and move on; it
  will keep firing.
- **Author override, and how to satisfy the stop hook at the same time.** The routine's override email
  differs from the git userEmail context — use the override. Set the AUTHOR with
  `git commit --author="Name <email>"` plus a `Co-Authored-By: Name <email>` trailer, and set
  `git config user.email noreply@anthropic.com` so the **committer** is the address the
  `stop-hook-git-check.sh` hook requires (it flags any commit whose committer is not
  `noreply@anthropic.com` as "Unverified"). Author ≠ committer satisfies both; verify with
  `git log -1 --format='%an <%ae> / %cn <%ce>'`. Do **not** follow the hook's suggested
  `--amend --reset-author`: that overwrites the author and breaks the override. And do not rewrite an
  already-pushed commit that carries an anchored review comment just for the badge — squash-merge
  replaces the committer anyway.
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
- **An OKF PR must be branched off a FRESH plugin master, kept small, and pushed the same turn.**
  `xwiki-dev-llm` takes several commits a day and every shipped change bumps the version in four
  manifests, so a branch cut at the start of a long run conflicts on exactly those lines and gets closed
  rather than merged ("Not applying, because code conflicts and it'll be recreated by the routine later
  on if still needed"). Fetch master right before writing the OKF edit, do the version bump last, and
  don't let the branch sit while a build runs. When one is closed for conflicts, do **not** reopen or
  re-push it — condense what it held here and let a later run re-author it on a clean master.
- **Owed to the OKF, batch 2** (closed PR `xwiki/xwiki-dev-llm#53` — closed for a conflict with
  *"not critical … if it's useful, a new PR will be created"*, so re-author it on a fresh master when a
  run next touches these rules; do NOT re-push that branch). Two universal drop conditions and five
  rules, all validated by a real sweep:
  - **Checkstyle METRIC rules as a drop condition** — `BooleanExpressionComplexity` (3 operators) and
    `ExecutableStatementCount` (30 statements/method) reject mechanically-correct S1871/S3358/S9016
    fixes *after* the tests run. Full text under *Building / verifying* above.
  - **A rationale can be a commit message, not a comment** — check the flagged line's git history before
    deleting. Full text under *Batch mode* above.
  - **S1871** merge two branches with an identical body: `||` short-circuits in the same order the chain
    evaluated in (so a second condition that would throw is still guarded); the discriminator is the
    merged expression's operator count vs the cap of 3, decidable before editing.
  - **S2589** the fixable subset is redundancy *created by the surrounding code* (a conjunct implied by
    the preceding branch of the same chain, a null check after an `instanceof` pattern binding); drop
    defensive checks, documented case tables, and guards repeated for symmetry.
  - **S3358** turn the OUTER condition into a guard clause and leave the inner ternary as the only one;
    drop when the ternary is an argument of a `this(…)`/`super(…)` delegating call (nothing may precede
    it).
  - **S3457** two shapes, neither free: the `toString()` shape needs `String.valueOf(x)` on a log call
    (cross-reference `conventions/logging.md`, which is the authority and which the sonarqube corpus
    still does not point at); the `%n` shape is a behaviour change.
  - **S3824** clean only when the guarded block is exactly one `put`; drop on an `else if`, and drop when
    the map is a `ConcurrentHashMap` whose mapping function calls another component (bin-lock hazard).
  - **S4973** belongs on the denylist — a real-bug rule needing a per-site semantic decision.
- **Owed to the OKF, batch 1** (closed PR `xwiki/xwiki-dev-llm#38` holds the full text; re-author on fresh master
  when a run next fixes these rules):
  - **S9016** extract nested mock creation — drive the edit off the issue `textRange`; insert the
    declaration at the statement start; name `<type>Mock` reserving names **per method** (file-wide
    reservation yields a stray `fooMock2` in a method with no `fooMock`); declare a **generic** mocked
    type parameterized and let the no-arg `mock()` infer it (a raw declaration leaves an unchecked
    warning — note `S3740` is NOT enabled for XWiki, so it is a merit argument, not a new issue); keep
    the class literal for a non-generic type (the rule's own compliant example does, and it is ~92% of
    platform test code); the type-inferred `mock()` form converts by hand when the stubbed getter's
    return type is already imported and is a drop when naming it needs a foreign import; never hoist a
    `mock()` out of a repeatedly-invoked lambda.
  - **S9016 follow-up — Vincent's style call, apply it from the start:** write the extracted local as
    `Foo fooMock = mock();` (type-inferred), not `mock(Foo.class)`. No rule requires it (`S3740`/`S6212`
    are not in the XWiki profile and the rule's own example keeps the literal), but it was asked for in
    review and applied across the whole batch, so treat it as the preferred form for extracted mocks and
    new test code. It is only valid where the target type is explicit; `mock(X.class, "name")` /
    `withSettings()` / `RETURNS_DEEP_STUBS` keep the literal.
  - **S4719** charset name → `StandardCharsets` — retyping a `private` charset constant to `Charset`
    clears every site in the file in one line, but only if no remaining use still needs a `String`;
    otherwise fix the call site and delete the constant/import the change orphans. Check for a
    `catch (UnsupportedEncodingException …)` that would become unreachable.
  - **S2589** condition always true — safe when the condition is the negation of the branch it sits in,
    or a guard repeated inside its own guarded block; **drop** a dead defensive null check in concurrent
    code (it costs nothing to keep and reads as intentional).
- **An OKF PR must re-derive the plugin version from the LIVE master right before pushing.**
  Concurrent sessions bump `1.0.N` too, so a bump computed from the commit you branched off is stale
  by the time you push, and the PR gets closed for it ("the versions are wrong") — the content is
  never even reviewed. Re-read `marketplace.json` on master (`gh api repos/xwiki/xwiki-dev-llm/
  contents/.claude-plugin/marketplace.json --jq .content | base64 -d`) at push time, bump from THAT,
  and also re-check the rule map on master: a parallel run may have just documented the same rules.
- **What the OKF already covers moves fast.** Between one run's PR and the next, parallel sessions
  documented eight more rules and bumped the plugin three times (1.0.10 → 1.0.14). Re-read the rule map
  and the family file on master before writing anything and drop whatever landed meanwhile: a PR that
  re-documents an existing rule is noise, and a version bump computed from your branch point gets the
  PR closed unreviewed.
- **Recording learnings (memory repo → `main`).** The xwiki-platform fix lives on a feature branch but
  learnings go to this repo's `main`. Do NOT edit on the feature branch then stash/checkout/pop (main
  has diverged; the pop bakes `<<<<<<<` markers into the commit). Instead `git checkout main && git pull
  origin main` FIRST, then edit and commit directly on main. Route the learning per the WRITE protocol
  at the top — durable rule-correctness facts go to the OKF as a PR, not here.
