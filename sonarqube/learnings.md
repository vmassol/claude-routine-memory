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
| Comment-only (safest of all, cannot change behaviour) | S7476 `///…` banner comment | `rules/syntax.md` |
| Pure syntax/annotation (safest, zero dataflow) | S1128 unused import, S1197 array designator, S1116 empty statement, S1161 missing `@Override`, S1611 redundant lambda parens, S1124 modifier order, S3878 redundant varargs array, S1118 add private ctor | `rules/syntax.md` (S1118 also in `rules/dead-code.md`) |
| Pure simplification (no use-verification) | S1125 boolean literal, S1488 inline return, S1858 pointless `toString()`, S2864 iterate `entrySet()`, S1612 lambda→method ref, S1602 useless lambda braces, S1155 `size()>0`→`!isEmpty()`, S1126 if-else→single return, S3706 `.stream().forEach()`, S2130 `valueOf`→`parseX` | `rules/simplification.md` |
| Empty-check (zero-dataflow, VERY safe) | S7158 `length()==0`→`isEmpty()` (String + StringBuilder/StringBuffer) | `rules/S7158-isempty.md` |
| Modernization (mid-size safe pools) | S1640 HashMap→EnumMap, S1604 anon class→lambda, S1643 String `+=` in loop→StringBuilder | `rules/S1640-enummap.md`, `rules/S1604-lambda.md`, `rules/S1643-stringbuilder.md` |
| Constant extraction | S1192 duplicated literal | `rules/S1192-duplicated-literal.md` |
| Unused-code removal (light dataflow) | S1068 field, S1481 local, S1854 dead store | `rules/unused-code.md` |
| Utility/dead-code | S1118 private ctor, S1144 unused private method, S1185 remove super-only override | `rules/dead-code.md` |
| Structural | S1066 merge nested `if` | `rules/S1066-nested-if.md` |
| Structural (deepest pool) | S6201 instanceof pattern matching | `rules/S6201-instanceof.md` |
| Other clean | S6204/S6211 `.toList()`, S2093 try-with-resources, S2119 reuse Random, S1143+S1163 finally-throws, S5361 replaceAll→replace, S2147 combine catch, S3626 redundant jump | `rules/other-clean.md` |
| Test-code | S5786 JUnit5 package-private, S5785 assertEquals/assertSame, S3415 swap expected/actual | `rules/test-code.md` |
| Test-code, fully scriptable (zero coverage risk) | S8924 static-import Mockito methods | `rules/S8924-static-import.md` |
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

- **When the ENTIRE classic allowlist AND the modernization pool are already in `dropped-issues.md`
  (every open key for S1118/S1185/S2093/S3626/S1192/S6204/S6201/S1640/S1643/S3878/S1068 matches a
  dropped key — a common steady state now), do NOT conclude "nothing safe left". PIVOT to the broad
  rule-distribution facet and scan for fresh *safe mechanical* rules not yet in the allowlist.** The
  proven pivot target was **S7158** (`length()==0`→`isEmpty()`, zero-dataflow) — now DRAINED in all
  three repos (114 fixed, 0 drops; see `rules/S7158-isempty.md`) but it regenerates, so re-query it.
  **The other lever once a rule is drained in platform is the SAME rule in commons/rendering** — the
  siblings are rarely swept, so a rule at 0 in platform often still has 30-40 sites in each of them
  (see *Process / conventions → Multi-repo mechanics*). When scanning the facet for more such rules, read the
  first issue's `message` to classify: SAFE mechanical candidates look like S7158/S1155/S1602 (useless
  lambda braces). **PROVEN second-generation pivot trio, all confirmed 0-drop and all present in every
  repo — go here FIRST when the classic allowlist is 100% dropped (the current platform steady state:
  its whole allowlist totalled 54 keys and every one already sat in `dropped-issues.md`):**
  `S7476` (`///` banner comments — comment-only, safest rule there is), `S3706` (`.stream().forEach()`)
  and `S2130` (`valueOf`→`parseX`). One sweep of the three cleared 48 platform + 28 commons + 0 drops.
  Cheap way to find the NEXT such rule: pull the broad facet, then batch one `ps=2` query per candidate
  rule and read just the `message` — one turn classifies ten rules. **`S8924` (static-import Mockito
  methods) is now VALIDATED** — test-only, zero coverage risk, fully scriptable, 27/27 in platform;
  see `rules/S8924-static-import.md`. It is the best platform fallback when the classic allowlist there
  is 100% dropped. Still-unswept candidates worth triaging: `S6397` (redundant regex character class,
  ~9 platform / 8 rendering). **Verdicts on the other two:** `S6485`
  (`new HashMap<>(n)`→`HashMap.newHashMap(n)`) is *safe and compiles* (repos are Java 21, `newHashMap`
  is Java 19+) but its 29 platform sites are spread over 14 modules incl. solr-api → poor build ROI;
  take it only as a rider on a reactor you're already building (oldcore holds 9). `S2065`
  (remove `transient`) — AVOID: the platform pool is job-status classes (`IndexerJob`,
  `PDFExportJobStatus`) that XWiki's job-status store serializes with XStream, so `transient` is load-bearing.
  AVOID from the broad list: S6355/S1123 (`@Deprecated since=` / add `@Deprecated` —
  needs the deprecating version, judgment), S5993 (constructor→protected — REDUCES visibility → revapi
  `visibilityReduced`), S5411 (boxed→primitive boolean — null-unbox behaviour change), S1168 (return
  empty vs null — behaviour change), S1172 (remove param — signature/override risk), S2143/S2160/S1141
  (java.time / equals / nested-try — refactors). S8714 (try/catch/fail→assertThrows) and S6068 (drop
  Mockito `eq()`) are test-only and structural — safe-ish but more effort, own PR if attempted.

## General batch-fix techniques (apply to every rule)

Cross-cutting mechanics shared by all rules; each rule's detail file notes only its *deltas*.

- **Verify the checkout is at the scan commit BEFORE trusting line numbers.** The container is often
  reset to the *latest* master (not the scanned one) — compare the last-analysis date
  (`api/project_analyses/search?project=…&ps=1` → `analyses[0].date`) against `git log -1 --format=%ci`.
  When they match, line numbers don't drift and line-keyed editing is safe. When the checkout is AHEAD
  (e.g. 3 days), lines DO drift and a line-keyed `old` silently won't match — tell subagents to LOCATE
  each site by the code pattern / Sonar `message` (method/field/constant name), not the line number.
  Brace-delta cross-check (S1066/S6201/S3878) still validates the edits regardless of drift.
- **For a regex-expressible rule, locate sites by per-file MATCH COUNT instead of line numbers** — the
  cheapest and most robust anti-drift technique, and it needs no `issue_snippets` call. Per flagged
  file: count pattern matches in the working copy and compare with that file's issue count from
  `issues/search`. Equal ⇒ transform every match (that IS the flagged set, whatever the drift).
  Unequal ⇒ inspect only those few files. Assert the equality per file and the total against the
  project `total`, and write nothing unless every file passes. Datapoint: 114 sites / 66 files across
  3 repos, only 2 files needed inspection (both a receiver shape the regex didn't cover — a fixable
  regex gap, not a bad site). Run it as a DRY RUN first, printing `- old` / `+ new` per site plus a
  `>120` line-length warning, then re-run with a `write` flag.
- **Apply a many-file mechanical batch in ONE assert-guarded script**, not dozens of `Edit` calls: for
  each `(file, old, new)` assert `content.count(old) == 1` FIRST (catches stale/drifted/ambiguous) and
  write NOTHING if any assertion fails. (`Edit`'s `replace_all` matches only the exact indentation you
  typed and SILENTLY leaves the same pattern at other depths — so prefer the script, and grep for the
  residual pattern after any batch replace.)
- **The assert-guarded script scales to STRUCTURAL rules too — prefer it over subagents.** A single
  `(file, old, new, nb_issues)` table where each `old` is a verbatim MULTI-LINE block asserted to occur
  exactly once handled 152 S6201/S8924 sites across 3 repos in one run with zero mis-edits, and it
  needs no post-hoc coverage cross-check because the assertions are the check. Workflow: ONE batched
  snippet dump (`±6` lines around every flagged line, grouped per module) → write the script → dry run
  → `write`. Track `nb_issues` per edit (one edit often resolves 2-7 issues) and assert the sum against
  the Sonar `total`. Reach for subagents only when the site count is far beyond what you can hold.
- If you DO delegate STRUCTURAL rules (reindent/brace surgery — S1066/S6201) or **per-site dataflow
  rules** (S6204/S1068/…) to PARALLEL general-purpose subagents (NOT Explore — they must Edit) over
  DISJOINT files: **never trust a subagent's self-reported "CONVERTED"** — it routinely reports (with a
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
  Grep the diff for >120 after every batch. **Assert the length on the CHANGED lines only** — a
  whole-file `all(len(l)<=120)` check aborts the script on PRE-EXISTING >120 lines that exist even in
  `src/main` (checkstyle excludes them); worse, if the assert fires mid-loop some files are already
  written. Validate every edit FIRST (count + new-line lengths), then write.
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
- **`while read` silently DROPS the last key of a file with no trailing newline** — `python3` writing
  `'\n'.join(keys)` produces exactly that, so a 106-key loop processes 105 and the miss looks like a
  transient API failure. End the file with a newline (`+'\n'`) or iterate in Python; either way
  re-verify the count afterwards.
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
  **The cheapest resolution is usually to DROP the offending MODULE, not to fix its coverage:**
  `git checkout -- <module>/`, remove it from `-pl`, note the exclusion in the PR body, re-run. Removing
  COVERED instructions always lowers the ratio (`(c-k)/(t-k) < c/t`), so any module pinned just above
  its threshold will fail — this is a property of the module, not a defect in your edit. Live case:
  `xwiki-commons-crypto-cipher`, 4 S6201 sites, 0.70 → 0.69.
- **A module failing mid-reactor SKIPS every module after it** — the ones before are genuinely verified
  (`SUCCESS` in the summary), the rest were never built. After dropping the failing module, re-run a
  reactor containing the SKIPPED ones; don't assume the first run covered them.
- **Chain the multi-repo builds with plain newlines, not `&&`** — a failure in repo A must not prevent
  repos B and C from building, since each ships its own PR. One failed repo then costs one short re-run
  instead of a full re-chain.
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
  **Write the multi-repo chain to a SCRIPT FILE and run `bash build.sh`, never as an inline heredoc.**
  In a chain, `cd repoA && mvn …` newline `mvn …` leaves the SECOND mvn in repoA — and re-typing a long
  heredoc to fix one `cd` reproduces the same bug verbatim (it happened three times in a row). A file
  you can `Edit` makes the per-`mvn` `cd` visible and fixable.
- **A pre-existing red module on master will fail YOUR reactor: confirm, drop the module, keep the rest.**
  Live example: `xwiki-commons-extension-api` fails `revapi` `java.annotation.removed` on
  `IndexedExtension#isCompatible` / `WrappingIndexedExtension#isCompatible` — fallout from the
  javax→JSpecify `@Nullable` migration on master, nothing to do with any sonar fix. Diagnose in one
  step: `git log -1 -- <the flagged class's file>` (a recent unrelated commit, and your `git diff
  --name-only` doesn't list it) ⇒ not yours. `git checkout --` your files in that module, drop it from
  `-pl`, and note the exclusion in the PR body. Modules the reactor SKIPPED after the failure still need
  a build — re-run them alone afterwards (they resolve the dropped module as a remote SNAPSHOT).
- Rough datapoints (build only; running the modules' unit tests adds substantially more time —
  oldcore's suite is large): small leaf modules ~10-45s warm; oldcore ~3.5 min warm / ~6.5 cold;
  feed-api ~5 min. Pick the smallest leaf modules; avoid Solr submodules and feed-api. Now that tests
  run, prefer a few dense modules over a wide reactor — fewer test suites to execute.
- **oldcore is NOT the build-ROI blocker it was thought to be.** Datapoint (cold `~/.m2`, no `-am`,
  `-Plegacy,quality` WITH tests): a 5-module platform reactor of oldcore + feed-api +
  filter-stream-xar + search-solr-api + tool-provision-plugin ran **7:25 total** (206 test suites,
  green). Compare: 16-module commons reactor 6:26, 7-module rendering reactor 2:11. So do NOT drop
  oldcore sites purely for "its test suite is huge" — a ~7 min background build is cheap in tokens.
  (Keep dropping oldcore *sites* on ROI only when the whole reactor exists for 1-2 fixes.)
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

- **Why not `gh` porcelain:** the `gh` binary may be installed, but `gh pr create/list/view` and
  `gh issue` FAIL here for two independent reasons: (1) the session proxy BLOCKS GraphQL (`HTTP 403:
  This GraphQL query is not enabled for this session`); (2) the git remote points to a local proxy
  (`127.0.0.1:.../git/...`), not github.com, so repo-context commands can't resolve the repo.
  `gh auth status` also mis-reports the token invalid. **`gh api` with a REPO-SCOPED REST path DOES
  work** (`GH_TOKEN` is in-env) and is the CHEAPEST channel for everything with a big response —
  cross-repo `gh api search/issues` is BLOCKED though (`sessions are bound to their configured
  repositories`), so scope every path as `repos/{owner}/{repo}/...`.
- **Prefer `gh api` REST over the GitHub MCP tools whenever the response or the request body is large**
  — MCP results land in context verbatim and `--jq` does not:
  - Open agent PRs per repo: `gh api "repos/xwiki/<repo>/pulls?state=open&per_page=100" --jq
    '.[]|select(.labels[]?.name=="llm-agent")|"\(.number) \(.title)"'`. `search_pull_requests` dumps
    every PR *body* (a batch PR body is huge); `list_pull_requests` is ~660KB. Both are last resorts.
  - **NEVER call `pull_request_read` `get_files` to learn which files a PR claims** — it returns the
    FULL PATCH of every file (tens of KB). Use `gh api "repos/…/pulls/N/files?per_page=100" --jq
    '.[].filename'`.
  - Create the PR from a FILE so a long body never passes through context:
    `gh api repos/{o}/{r}/pulls -X POST -f title=… -f head=<branch> -f base=master -F body=@pr.md
    --jq '.number,.html_url'`. Same for editing it (`-X PATCH -F body=@pr.md`).
  - Label + assignee in one call: `gh api repos/{o}/{r}/issues/{n} -X PATCH -f 'labels[]=llm-agent'
    -f 'assignees[]=vmassol'`. Lock (Vincent's override): `gh api --method PUT
    repos/{o}/{r}/issues/{n}/lock -f lock_reason=resolved`.
  (The skill text says "open the PR with `gh pr create`" — ignore that; use `gh api` REST as above.)
- **Vincent's PR lock blocks YOUR OWN comments too** (`403 issue is locked`), so a reviewer reply needs
  `DELETE …/issues/{n}/lock` → post the comment → `PUT …/issues/{n}/lock -f lock_reason=resolved` again.
  Do the unlock/re-lock in ONE command so the window stays short. Reviewers can still leave reviews on a
  locked PR, so expect to need this.
- **A reviewer saying "this feels wrong" about a MECHANICAL rule usually means the rule is a bad fit for
  that KIND of code, not that the transformation is broken.** Verify the mechanism (so the record is
  accurate — cite bytecode/source if you can), then judge whether the objection is about *intent
  clarity*; if it is, it stands even when the change is provably behaviour-preserving. Withdraw rather
  than argue: reply with the clarification and close the PR. Then see the next bullet for what to do
  with the issues — do NOT stop at closing them.
- **THE RIGHT WAY TO RETIRE A "WON'T FIX" ISSUE IS IN THE CODE, NOT IN SONARCLOUD.** Vincent's standing
  preference, and XWiki's documented practice (the SonarQube section of
  `https://dev.xwiki.org/xwiki/bin/view/Community/CodeStyle/JavaCodeStyle/`): add
  `@SuppressWarnings("java:SXXXX")` on the concerned class/method/field, **immediately preceded by a
  `//` comment stating WHY** — so the next developer reads the reason next to the code and isn't tempted
  to "fix" it again. Marking the issue *Accepted* in SonarCloud alone hides the reasoning from readers
  and is NOT sufficient on its own. Conventions, all verifiable by `grep -rn -B4 '@SuppressWarnings("java:S'`:
  the argument is the RULE KEY (`java:S5785`), the justifying `//` comment goes directly above the
  annotation (after `@Override`/`@Test` if present, i.e. the annotation is last before the declaration),
  and method-level scope is preferred over class-level so the rest of the file stays covered. Ideal
  template to copy: `RegexEntityReferenceTest` in `xwiki-platform-model-api` (class-level
  `@SuppressWarnings("java:S3415")` + a 5-line rationale). This route is VALIDATED, not speculative:
  `xwiki/xwiki-rendering#390` (6 method-level suppressions, no assertion touched) was MERGED as-is,
  right after the conversion PR for the same issues was closed on review. Workflow: revert the conversions, add
  suppression+comment, rebuild, ship as its own PR, and `do_transition transition=reopen` the issues with
  a comment saying the suppression is what will close them on the next analysis. Note the dev.xwiki.org
  page itself sits behind Cloudflare (403 to both WebFetch and curl) — derive the convention by grepping
  the repos instead.
- After such a reversal: post a CORRECTION comment on each Sonar issue (the original "Fixed by <PR>"
  comment is now wrong), and narrow the rule's entry in `rules/` to the code shape that provoked the
  objection — don't blanket-denylist a rule that is fine elsewhere. Only list keys in
  `dropped-issues.md` when there is genuinely no fix AND no suppression to add.
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
- **Multi-repo mechanics when all three ARE in scope** (the good case — it is the cheapest way to blow
  past a 30-fix target, since ONE rule usually has a pool in each repo):
  - `SONARQUBE_PROJECT_KEY` is only set for the session's primary repo. The siblings' keys are
    `org.xwiki.commons:xwiki-commons` and `org.xwiki.rendering:xwiki-rendering` (enumerate with
    `api/projects/search?organization=xwiki&ps=50`). Query each with `componentKeys=<key>`.
  - Do the whole find/apply phase for all three repos FIRST (cheap, no builds), then build them
    **SEQUENTIALLY in dependency order commons → rendering → platform**. They share one `~/.m2`, so
    concurrent `install`s race; and building commons first means rendering/platform verify against
    your *modified* commons jars rather than a downloaded SNAPSHOT. Chain all three in ONE background
    subshell with an explicit `cd /home/user/<repo> &&` before EACH `mvn` (a bare `cd /home/user`
    start makes the first `mvn` run where there is no pom) and an `echo ###### <REPO> ######` marker
    between them, then grep the markers + `BUILD SUCCESS`. Datapoint: 2 commons modules 3:31 +
    6 rendering modules 2:24 + 4 platform modules incl. oldcore 7:12 ≈ **13 min for 94 fixes**.
  - Datapoint (this shape is the reliable way to blow past a 30-fix target): **152 fixes in one session
    — commons 73 `S6201` (crypto tree), rendering 52 `S6201` (all of them), platform 27 `S8924`** — 3
    PRs, ~25 min of background builds with tests, 4 drops (one module, coverage). Zero open agent PRs in
    all three repos is the common starting state; check it, then go wide.
  - **The mechanical pool is usually drained in PLATFORM but untouched in the siblings** — platform's
    whole classic allowlist can total <60 (nearly all already in `dropped-issues.md`) while commons
    holds ~260 S6201 / 30 S3878 / 25 S1066 and rendering ~52 S6201 / 24 S1124 / 30 S5785. So on a
    multi-repo run, spend the effort in commons+rendering and treat platform as the small remainder;
    rendering in particular has a deep unclaimed NON-S6201 mechanical pool (S1124/S1488/S1611/S1602/
    S1197/S5361/S1192/S1066), most of it concentrated in the single `xwiki-rendering-wikimodel` module.
  - Ship each repo as its own commit + branch + PR (label + assignee + lock each). Cross-link the
    sibling PRs in a "Related" section so a reviewer sees it is one sweep.
  - Check open agent PRs per repo (`repo:xwiki/xwiki-commons`, …) — the siblings are usually at 0.
- **Holding the turn open during a build avoids the "uncommitted changes" stop-hook ping-pong.** Ending
  the turn while a background build runs trips that hook every time. Arming a `Monitor` with
  `until grep -qE "BUILD SUCCESS|BUILD FAILURE" <task output>; do sleep 15; done` keeps the turn alive
  for ~one extra turn's cost and reads the outcome in the same wake-up. Never commit unverified just to
  silence the hook.
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
