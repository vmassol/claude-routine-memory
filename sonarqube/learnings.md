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

## Rule index (detail files under `sonarqube/rules/`)

Rule-specific gotchas that are NOT in the plugin OKF live in one file per rule here. Read only the
rows for the rules you commit to fixing this run.

| Rule | File | One-line reason it has a file |
|---|---|---|
| `javascript:S7773` | [rules/javascript-S7773.md](rules/javascript-S7773.md) | `parseInt`/`parseFloat` are free; `isNaN`/`isFinite` are NOT equivalent |
| `javascript:S6353` | [rules/javascript-S6353.md](rules/javascript-S6353.md) | `\D`/`\W` equivalence depends on the regex flags — prove it with `node` |
| `java:S6885` | [rules/java-S6885.md](rules/java-S6885.md) | `Math.clamp` throws when `min > max`; the nested form does not |
| `java:S1193` | [rules/java-S1193.md](rules/java-S1193.md) | the duplicated `throw` creates a fresh `S1192` |
| `java:S1872` | [rules/java-S1872.md](rules/java-S1872.md) | split verdict: `void.class` is clean, `isAssignableFrom` is a behaviour change |
| `java:S6916` | [rules/java-S6916.md](rules/java-S6916.md) | false positive on an enum `switch` — guards need pattern labels |
| `javascript:S7762` `javascript:S7768` | [rules/javascript-S7762.md](rules/javascript-S7762.md) | safe only when the receiver provably IS the node's `parentNode` |
| `javascript:S6637` | [rules/javascript-S6637.md](rules/javascript-S6637.md) | the flagged `}.bind(this)` text is never unique — and the sibling binds must stay |
| `java:S3415` | [rules/java-S3415.md](rules/java-S3415.md) | **not** a default drop (15/18): the free classifier is whether either operand is `null` |
| `javascript:S7781` | [rules/javascript-S7781.md](rules/javascript-S7781.md) | fires only on single-character regexes, and pairs with `S6397` — one edit clears two issues |
| `javascript:S6660` | [rules/javascript-S6660.md](rules/javascript-S6660.md) | the drop condition is diff size, and a comment between `else {` and `if` must move |
| `javascript:S6644` | [rules/javascript-S6644.md](rules/javascript-S6644.md) | `\|\|`, never `??` — the original test was truthiness |
| `java:S3457` | [rules/java-S3457.md](rules/java-S3457.md) | ONE rule key, five unrelated defects — triage it by `message` |
| `javascript:S6582` | [rules/javascript-S6582.md](rules/javascript-S6582.md) | `0?.f()` THROWS where `0 && 0.f()` short-circuits |
| `javascript:S7765` | [rules/javascript-S7765.md](rules/javascript-S7765.md) | `includes` differs from `indexOf` only for `NaN`; the receiver must really be an Array |
| `java:S1172` | [rules/java-S1172.md](rules/java-S1172.md) | OKF-denylisted, but only non-`private` is a signature change — the private subset is a 45-site three-repo pool; and `private` is NOT airtight, an AspectJ ITD can call it |
| `java:S6355` `java:S1123` | [rules/java-S6355.md](rules/java-S6355.md) | OKF-denylisted for "needs the deprecating version" — which the element's own `@deprecated` Javadoc tag already states in 464 of 768 sites |
| `java:S1186` | [rules/java-S1186.md](rules/java-S1186.md) | comment-only, and the comment is a property of the CLASS — 192 issues are ~20 class-level judgements |
| `java:S108` | [rules/java-S108.md](rules/java-S108.md) | comment-only too (the rule ignores a block containing a comment); 82 sites are ~35 sentences, and an empty `else`/`finally` is deleted instead |
| `java:S9142` | [rules/java-S9142.md](rules/java-S9142.md) | hoisting a regex out of a loop is javadoc-definitional, but `String#split` on ONE character compiles no Pattern — that shape is a false positive |
| `java:S9355` | [rules/java-S9355.md](rules/java-S9355.md) | `/*`→`/**` for a real Javadoc; DELETE the `(non-Javadoc)` boilerplate instead of promoting it |
| `java:S6213` | [rules/java-S6213.md](rules/java-S6213.md) | OKF-denylisted for the *method* half only — `"Rename this variable"` vs `"Rename this method"` splits the pool for free |
| `java:S5993` | [rules/java-S5993.md](rules/java-S5993.md) | OKF-denylisted as a Revapi visibility break — true only outside `internal` packages, which `revapi.json` excludes; and JLS §6.6.2.2 makes it a no-op for an *abstract* class |
| `java:S1130` | [rules/java-S1130.md](rules/java-S1130.md) | the recorded "permanent `src/main` residue" is a VISIBILITY split — 19 `private` sites were sitting in it |
| `java:S2177` | [rules/java-S2177.md](rules/java-S2177.md) | the pool is `private` methods, so the rename is free; drop when the method's counterpart is a `public` one that cannot follow |
| `java:S6880` | [rules/java-S6880.md](rules/java-S6880.md) | `switch` enforces a dominance rule the `if`-chain does not, so a compile error IS a dead-branch finding; `case null` is the only behaviour question; and a fifth of the pool is `==`-against-constants, not `instanceof` |
| `java:S2142` | [rules/java-S2142.md](rules/java-S2142.md) | one inserted line, and the pool splits for free on the catch clause's TYPE — a broad `catch (Exception)` is the drop |
| `java:S8786` | [rules/java-S8786.md](rules/java-S8786.md) | whole-rule drop: a flagged site ALREADY carries the possessive-quantifier fix the rule recommends, so the remediation does not clear the issue |
| `java:S9354` | [rules/java-S9354.md](rules/java-S9354.md) | free everywhere except where a TEST asserts the *magnitude* of `compareTo` — grep for it, the build round is the alternative |
| `java:S9357` | [rules/java-S9357.md](rules/java-S9357.md) | the 2026 restatement of `S1604`, so its drop index already applies; two more drops are a target parameter typed `Object` and an observed `getClass().getSimpleName()` |
| `java:S2386` | [rules/java-S2386.md](rules/java-S2386.md) | denylisted for the MESSAGE's fix (`protected` → Revapi); making the value immutable clears it with no declaration change |
| `java:S2198` | [rules/java-S2198.md](rules/java-S2198.md) | ask WHY the comparison is dead — a proper fix that RE-ADDS your deleted line means the line was evidence of a defect |
| `java:S1123` | [rules/java-S1123.md](rules/java-S1123.md) | splits by `message`; the annotation half's biggest bucket is sites carrying a COMMENTED-OUT `@Deprecated` under a `// TODO:` |
| `javascript:S3504` | [rules/javascript-S3504.md](rules/javascript-S3504.md) | the module-global `var XWiki = (function…)` must stay `var`; the rest co-locates with `S2814`/`S4138` |
| `css:S4666` `S4656` `S4657` `S4670` `S4651` `S8759` `S125` | [rules/css-rules.md](rules/css-rules.md) | the whole CSS facet: merging a duplicate selector is free UNLESS both blocks set the same property, and a `.css` file can be a Velocity template (S4670 FP) |
| `java:S3398` | [rules/java-S3398.md](rules/java-S3398.md) | the only rule whose fix is metric-exposed *by construction* — moving a method INTO a class raises that class's Checkstyle `ClassFanOutComplexity`; recover by moving the cheap ones only |
| `java:S4144` | [rules/java-S4144.md](rules/java-S4144.md) | a BUG detector half the time — the pair's NAMES classify it: same operation ⇒ extract, different operations ⇒ the identical body is the defect, report it |
| `javascript:S4138` `javascript:S1940` | [rules/javascript-S4138.md](rules/javascript-S4138.md) | not a trap: index-used-only-as-`collection[i]` plus a `Symbol.iterator` receiver check (jQuery ≥3 — read the pom). `S1940`'s `!(x >= 0)` → `x < 0` is an FP |

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
- **A recorded "N workable" count is PR-constrained and expires the moment those PRs merge — when the
  open-agent-PR list comes back EMPTY, re-derive every pool from scratch.** The rule that a past run
  wrote down as "12 workable of 90" was 61 workable on a day with zero open agent PRs, and its
  sibling went 12 → 32 the same way; two rules that `pool-state.md` described as nearly spent
  yielded 92 issues in one run. Concurrent PRs are the binding constraint in the WAR resources, so
  the *first* call of the run (`pulls?state=open` filtered by the `llm-agent` label) decides whether
  the recorded counts mean anything at all. An empty list is the strongest "re-query everything"
  signal there is — treat the recorded numbers as a lower bound, never as the pool.
- **Check open agent PRs up front** — `gh api "repos/xwiki/<repo>/pulls?state=open&per_page=100" --jq
  '.[]|select(.labels[]?.name=="llm-agent")|"\(.number) \(.title)"'`. A recent PR can drain a WHOLE
  rule family. Scope the off-limits check by **(rule + module)**, not rule alone: a per-module batch PR
  only claims the files it touched, so the same rule in OTHER (incl. sibling) modules is fair game.
  When your planned rule already has multiple open PRs, PIVOT to a zero-PR family rather than threading
  the gaps. **A same-FILE open PR is off-limits even for a DIFFERENT rule** — a concurrent edit risks
  merge conflicts; drop that site. To learn exactly which modules a wildcard "various modules" PR
  claims, read its file list (`gh api "repos/…/pulls/N/files?per_page=100" --jq '.[].filename'`).
- **Verify a DENYLIST reason against the rule's own definition before writing a repo off.** The OKF
  denylist is a one-line summary and it can be wrong: `S3252` was listed as "usually
  backward-compat-bearing public API" when the rule in fact only asks you to qualify a static member
  with the class that declares it — no declaration changes at all. One
  `api/rules/show?organization=xwiki&key=java:SXXXX` gives the rule's name and compliant example and
  settles it in a single call; that one re-opened 123 clean CRITICAL sites across three repos on a day
  when the entire classic allowlist read 61/19/6 and was almost all already dropped. Two entries have
  now been found wrong this way (`S1117`, `S3252`), and both were rules that *sound* like renames.
  Run the check when the allowlist is dry, not before — and record the correction as an OKF PR, since
  a wrong entry (unlike a missing nuance) is worth the version-bump conflict risk.
- **A denylist reason that is a property of only PART of a rule's pool splits it — and the split axis
  is almost always VISIBILITY.** Fourth denylist rescue after `S1117`/`S3252`/`S3415`, and the first
  found by reading the *denylist itself* rather than the allowlist: `java:S1172` is listed as "a real
  behaviour or signature change", which is true of a `public`/`protected`/package-private method and
  false of a `private` one, where no caller can exist outside the compilation unit and the compiler is
  therefore the whole verification. One bucketing pass over the modifier on each flagged declaration
  line (no snippet reads beyond that line) turned "132 platform / 25 commons / 5 rendering, all
  denylisted" into a **45-site three-repo pool, 41 of them shipped**. So when the allowlist is dry,
  re-read the DENYLIST for entries whose reason reads "API change" / "signature change" /
  "breaks callers" and ask *which subset of the pool that is actually true of* — `private`, `internal`
  package, and test-only sites are the standard escapes. **Outcome: all three PRs merged with no
  review comment at all**, which is now the fourth denylist rescue to land that way — the
  re-derivation keeps paying, and it is worth one turn every time the allowlist reads dry. Same lever as `S1130`'s
  annotation-and-`private` bucketing and `S117`'s locals-vs-parameters split.
  **When a denylist reason names a BUILD GATE, RUN THE GATE. Reading its config is the weaker move and
  it cost a whole extra run.** `java:S5993` ("make this abstract class's constructor `protected`") was
  denylisted as a Revapi `visibilityReduced` break. Run A read `revapi.json`, found it excludes
  `**.internal.**` and `**.test.**` in both `org.xwiki` and `com.xpn.xwiki`, shipped the **81
  `internal`** sites on that basis and wrote the other 163 down *here* as a permanent drop —
  a correct fix built on an unverified premise. Run B spent **2:01** instead: apply four
  non-`internal` sites in a small leaf module, `mvn package revapi:check -Pquality -DskipTests -pl
  <module>` → *"API checks completed without failures"*, and the "permanent drop" shipped as **162
  sites across all three repos**. The mechanism was in the same config file the first run had already
  opened: it reclassifies **all source-compatibility checks to EQUIVALENT severity**, and narrowing an
  abstract class's constructor is source-only. Generalise: a config file tells you what the gate
  *looks at*; only running it tells you what the gate *says*, and one small module with `-DskipTests`
  is two minutes. Do this for every denylist entry whose reason is "the build will reject it" —
  Revapi, Checkstyle metrics, Enforcer, JaCoCo — before believing it, and before believing your own
  earlier run that only read the config.
  **And a "reduces visibility" objection is sometimes refutable outright, not just scoped away** — for
  an *abstract* class, JLS §6.6.2.2 permits both `super(...)` and `new AbstractX(…){…}` on a
  `protected` constructor from any package, and plain `new AbstractX(…)` is already illegal, so no
  compilable caller can break. A two-package `javac` proving exactly that took one turn and is the
  strongest thing in the PR body. Generalises: when the denylist reason is a *claim about the
  language*, spend the throwaway `javac` before accepting it.
- **When a fix's correctness turns on a CONVENTION, read the convention's own page before inventing
  an open question — and never trust a recorded "that source is unreachable".** The `S6355` sweep hit
  a Javadoc tag naming two versions, reasoned that `@Deprecated(since)` takes one `String`, picked one
  and shipped the choice as the open question of a judgement PR. The Java Code Style page answers it
  outright — *"If the deprecation is done in several branches, the since parameter should use a
  comma-separated list of all versions"*, `@Deprecated(since = "15.5RC1,14.10.12")` — and the same
  section also mandates the two things review had already asked for (no version in the `@deprecated`
  tag; a WHY/WHAT sentence in it). Two failures compounded: the OKF's `versioning.md` covered the
  version *format* and the `@since` backport rule but nothing about `@Deprecated`, and a note in THIS
  file said the page 403s behind Cloudflare, so no run re-tested it (it returns 200). Both fixed
  (`xwiki-dev-llm#72`). Generalises: **a "judgement call" you are about to hand a reviewer is first a
  documentation lookup** — if the question is "which of these does XWiki want", the answer usually
  exists, and asking it in a PR body is how a run discovers that the OKF was incomplete *after*
  shipping rather than before.
- **When you LOOSEN a classifier regex mid-run, re-test it against the cases the strict version
  rejected — the rejects are the test set.** The `S6355` triage ran twice: a first pass whose version
  pattern was `\b(…)\b` and correctly rejected the malformed versions in the pool (`4.4MA`, `5.2M`,
  `14.0CR1`), and a second, tightened-in-other-ways pass that dropped the trailing boundary and so
  matched their numeric *prefix*. Three sites shipped with a truncated version in the annotation and
  the orphaned tail (`MA`, `M`, `CR1`) left behind in the Javadoc — compiles, passes Checkstyle,
  Revapi and 4738 tests, and is the first thing the reviewer commented on. Two guards, both cheap:
  keep the earlier regex's rejects as an explicit expectation when you rewrite it, and after applying,
  **audit each site for a capture that is a PREFIX of a longer token** in the original text (the next
  character is alphanumeric, or `.` + digit). That audit took one pass over the recorded rows, needed
  no source reads, and found all three — including the two the reviewer had not spotted, which is what
  turns one comment into one push instead of three rounds.
- **When a fix MOVES a fact into a machine-readable form, deleting the prose copy is part of the same
  change — expect to be asked for it.** The `S6355` sweep put the deprecating version into
  `@Deprecated(since = …)` and left the `@deprecated since 8.3RC1, use {@link X} instead` Javadoc tag
  untouched; Vincent's first instruction on the open PRs was to strip the version from the tag too
  ("once you've moved the since info to the annotation, also remove it from the javadoc"). The
  duplicate would have drifted, so the ask is right and it generalises: whenever a rule's fix relocates
  information (a version, a type, a name) into an annotation or a typed API, grep the neighbouring
  comment for the old copy in the same edit. Cost of getting it wrong: a second full reactor over the
  same 77 modules (~21 min warm) and a second commit on six open PRs.
- **A denylist reason can be about the MESSAGE'S suggested remediation rather than about the rule —
  and most rules have more than one compliant outcome.** Eleventh denylist rescue and a *new escape
  axis*, distinct from the visibility split and from "is the missing fact already written down".
  `java:S2386` was listed as *"make this member `protected`" → Revapi `java.field.visibilityReduced`*,
  which is exactly what its `message` says and is a real break. But the rule fires on a `public
  static` member whose **value** is mutable, so making the value immutable (`Arrays.asList(…)` →
  `List.of(…)`) clears it with **no declaration change at all** — 10 sites across platform and
  rendering, and it is one of the very few rules with a pool in all three repos on a day the
  mechanical allowlist is dry. Ask it of every denylist entry whose reason paraphrases the issue
  message: *is this an objection to the rule, or to one way of satisfying it?*
  **The confirmation step is a grep of the scanned source, not `api/rules/show`.** S2386's
  description shows only non-compliant examples, so the API could not settle whether the analyzer
  accepts the alternative form. What settled it in one call: **find a constant already written in the
  candidate form and confirm it carries no open issue** — `PasswordClass.SUPPORTED_ALGORITHMS
  = List.of(…)` (platform) and `ImageBlockParser.IMAGE_ALIGNMENT = Map.of(…)` (rendering), both
  `public static final` collections, both unflagged. That in-repo precedent is also the strongest
  sentence in the PR body, and it is the general answer to the `S8786` hazard ("does the remediation
  actually clear the issue?") whenever the rule doc is silent.
- **A denylist reason of the form "the fix needs information X" is a question about whether the code
  already WRITES X DOWN — and in XWiki it often does, two lines above the flagged line.** Fifth
  denylist rescue (after `S1117`, `S3252`, `S3415`, `S1172`) and the biggest pool any run has found:
  `java:S6355`/`S1123` were denylisted as "needs the deprecating version; see versioning", which
  reads like an unanswerable question. But the flagged `@Deprecated` sits directly under a Javadoc
  `@deprecated since 8.3RC1, use {@link …} instead` tag — **464 of the 768 open sites state their own
  version** and the fix copies it. So this is a *different escape axis from the visibility one*: not
  "which subset of the pool is the reason false for", but "is the missing input already documented on
  the element?". Ask it of every denylist entry whose reason is a missing *fact* (a version, a
  replacement API, an owner) rather than a missing *safety* argument. Related, and worth trying next:
  `S6355`'s own siblings `S1123` (the tag exists, the annotation is missing — the version is equally
  derivable) and `S1133`.
- **A rule whose REMEDIATION IS A COMMENT is a whole untouched generation — and its unit of judgement
  is the CLASS, not the site.** Sixth rescue of a never-attempted rule, and the first found by asking
  what the *fix* costs rather than what the rule's reason says: `java:S1186` ("methods should not be
  empty", CRITICAL, 192 open across all three repos) had been passed over by every run as "needs
  prose", the same objection that keeps `S1123`/`S1135`/`S1134` off the table. But prose is only
  expensive **per site**, and this pool is clustered — 192 issues in 78 files, the top 6 files holding
  96 of them, every one a deliberate no-op (stub, void/null object, default hook, deprecated no-op,
  component-manager constructor, Velocity factory constructor). One sentence per *class* covers every
  empty body in it, so the batch cost ~20 judgements, not 192, and **172 shipped**. Two consequences
  worth carrying: (a) when the allowlist and the denylist re-derivations are both spent, sort the
  facet for rules whose remediation is **comment-only** — they are as safe as `S7476` and they survive
  every cleanup wave, exactly like the naming rules do; (b) before writing off a "needs prose" rule,
  **group its issues by file first** — a flat count hides whether the prose is per-site or per-class.
  See [rules/java-S1186.md](rules/java-S1186.md). The drop condition on such a rule is not safety but
  **truthfulness**: drop the site when you would have to invent the reason (6 of 192 here), because a
  comment asserting intent the code does not support is worse than the open issue.
- **A "permanent RESIDUE" recorded by a past run is the same visibility split as a denylist entry —
  re-bucket it.** Ninth rescue of this shape, and the first where the wrong record was written by
  *this routine* rather than by the OKF: `pool-state.md` described platform's `java:S1130` remainder
  as "the permanent `src/main` residue" (54 of 58 keys "fresh" but written off), and one bucketing
  pass over the modifier gave **19 `private` `src/main` sites**, all of which shipped green.
  `src/main` was never the drop line for that rule — non-`private` is, exactly as for `S1172`,
  `S116`, `S117` and `S5993`. **Outcome: merged uncommented within hours** (platform #6236), which is
  now what every one of these rescues has done, so the re-derivation is insurance, not a gamble. So treat every recorded phrase of the form "the X residue" as a claim
  to re-derive, not a fact: the cost is one `issues/search` plus one line read per site, and the
  wording that hides a pool is always a *proxy* for the real condition (`src/main` for visibility,
  "needs prose" for per-class prose, "API change" for `private`).
- **Bucketing by the FLAGGED line's modifier is wrong for multi-line signatures, and it fails
  silently in the safe direction's favour... or against it.** Sonar attributes a `throws`/parameter
  issue to the line holding the token, which under XWiki's wrapping style is often a continuation
  line carrying no modifier at all (`        throws WikiManagerException, ComponentLookupException`,
  `        Map<K, V> parameters) throws E`). A `'private' in flaggedLine` test buckets those as
  package-private — it under-counted the `S1130` private pool and over-counted `S1172`'s.
  Walk back to the nearest line matching `\b(public|protected|private)\b` before classifying.
- **"Is this repo dry?" is ONE query per repo, not a per-rule sweep.** Pull EVERY open Java issue key
  (`issueStatuses=OPEN&languages=java&ps=500`, 3 pages covers ~1200) and grep the keys against
  `dropped-issues.md`; then read only the rule names of what survives and check them against the OKF
  denylist plus `pool-state.md`'s rejected list. Two turns settled commons (1155 issues) and rendering
  (430) as closed with zero source reads. Do this before budgeting any sibling-repo PR. **The cheaper
  variant, and the one to open a run with: one `issues/search` per repo carrying a ~78-rule mechanical
  allowlist (`&rules=…&ps=500`), then grep every returned key against `dropped-issues.md`** — three
  calls plus one grep gave a whole cross-repo plan (platform 69 fresh of 201, commons 8 of 59,
  rendering 0 of 22) and both siblings were answered without opening a single file.
- **When the classic allowlist reads zero, the volume is in a big NEVER-TRIAGED MAJOR rule, not in the
  allowlist's residue.** Sort the distribution facet for rules with 20+ issues that `pool-state.md`
  marks *untriaged*, and prefer the one whose fix is decidable from the flagged block alone: `S3824`
  (35) yielded 27 in one PR, more than the entire remaining allowlist across all three repos. The
  per-site triage there is ~10 lines of context each, which is affordable precisely because no
  whole-file read is needed.
- **When the whole JAVA facet is dry, the answer is `languages=js`, not another Java rule.** Platform
  carries ~880 open `javascript:` issues and no run had touched them; the Java allowlist that day
  returned 59 "fresh" keys of which 54 were the permanent `S1130` `src/main` residue. Two mechanics
  matter: (a) **`javascript:` rule keys START WITH THE STRING "java"**, so the obvious
  `rule.startswith("java")` filter for "show me the non-Java rules" silently hides the entire JS
  generation — split on the `:` (`rule.split(':')[0]`) or query `&languages=js&facets=rules`
  separately. (The `rules` facet also caps at **100 values**, so on a repo with more rules than that
  the tail is genuinely missing — check `len(values)` before trusting a facet as exhaustive.)
  (b) the pool is **almost entirely one module**,
  `xwiki-platform-web-war/src/main/webapp/resources/`, i.e. the hand-written Prototype-era WAR
  scripts. Commons and rendering have **no** JS at all.
- **Before touching any WAR JavaScript file, check its header for third-party provenance.** Several of
  the densest files are vendored scripts XWiki redistributes rather than maintains
  (`table/tablefilterNsort.js` — Guglielmi/de Valk/Eldenmalm, `widgets/validation/livevalidation_prototype.js`
  — LiveValidation 1.4 MIT); XWiki-owned files carry the standard LGPL "See the NOTICE file" banner.
  Excluding the vendored ones cost 22 issues out of 65 and is the right call — say so under
  *Clarifications*. **`git log` cannot answer this here**: the clones are shallow, so every file's
  history collapses to the same boundary commit.
  **Provenance is a BLOCK-level property, not only a file-level one — grep the file's INTERIOR, not
  just its first 20 lines.** `xwiki.js` carries the standard LGPL banner and is XWiki-owned, yet its
  `BrowserDetect` function (~lines 1055-1130) is a third-party snippet with its own header
  (`Version: 2.1.6`, an author, a CC-BY URL) — and it held **34 of the 61** non-vendored-file
  `S7765` sites, i.e. more than half the pool, invisible to a header-only check. One
  `grep -nE "^ \* (Author|License|Version|URL):"` per candidate file finds these. The right move is
  not to drop them: the transform was provably safe, so they went into the **judgement PR as their
  own commit** with the policy question stated ("does XWiki want to diverge an attributed snippet
  further from upstream?"), which keeps the mechanical half clean and still puts the 34 issues in
  front of a maintainer.
- **Triage the JS pool by RULE SEMANTICS first — several of the big ones are traps.** `S1848` "useless
  object instantiation" is a false positive on Prototype (`new Ajax.Request(…)` *is* the side effect);
  `S6582` optional chaining, `S7741` `typeof x === 'undefined'`, `S4138` `for`→`for-of`, `S7740`
  `var self = this` and `S7761` `.dataset` all change behaviour in at least one shape. The
  provably-safe subset found so far is `S7773` (partly) and `S6353` — see the Rule index.
- **A drop verdict phrased as a PRIOR ("usually unsafe", "default DROP") is not a drop CONDITION —
  re-derive it from the flagged line.** Third instance after `S1117` and `S3252`, and the cheapest yet:
  `java:S3415` was "default DROP" here and "usually unsafe — default to dropping it" in the OKF, yet a
  platform pass went **15/18** green first try. The condition the caution was really about (an
  `assertNotEquals(obj, null)` contract assertion, an asymmetric `equals()`) is decidable **from the
  one line the issue already points at** — does either operand read `null`? Whenever a recorded
  rejection reads as a probability rather than a test, spend one `ps=5` query and read the flagged
  lines: if the discriminator is visible there, the rule is a pool, not a drop. **Outcome datapoint:
  that PR merged with no review comment at all**, which is the same verdict `S1117`/`S3252` got — a
  rule rescued from a prior-shaped rejection has now merged uncommented three times running, so the
  re-derivation is cheap insurance rather than a gamble.
- **Intersect the `(file, line)` sets of your shortlisted rules — CO-LOCATED rules double the yield per
  edit.** `javascript:S7781` and `javascript:S6397` both fire on `temp.replace(/[Ĳ]/g,"IJ")`, so going
  straight to `replaceAll("Ĳ","IJ")` cleared **14 issues from 7 edits**. This is the opposite of the
  known "two issues on the same line will not both land" hazard: it is that hazard turned into a lever,
  and it is free — one set-intersection over the fresh list, no source reads. Also worth checking
  across languages/families, since the pairing is about the code shape, not the rule family.
- **A rule key that bundles several UNRELATED defects is triaged by `message`, not by rule.** When a
  rule's own name is generic (`java:S3457` "printf-style format strings should be used correctly"),
  one `issues/search` grouped by `message[:60]` splits the pool before any source read: 41 platform
  issues came out as five shapes with five different verdicts (18 `toString()`, 12 no-format-specifier,
  8 `%n`, 2 unused-argument, 1 concatenation → 7 fixable, 34 drops). Print the message histogram first;
  it *is* the triage, and it stops a run rejecting a whole rule because its biggest shape is a drop.
- **"Which open rules has no run ever LOOKED at?" is one grep, and it is the cheapest find-phase step
  there is.** Concatenate this repo's three files + `rules/*.md` + the plugin's `okf/sonarqube/*`,
  pull each repo's rule facet, and print the rules whose key does not appear in that text. Three
  calls and a regex; on a day when the whole mechanical allowlist read zero fresh keys in all three
  repos it surfaced **`java:S9142` and `java:S9355`** — never triaged, both mechanical, both present
  in two repos — and `css:S4666`. Run it BEFORE re-deriving denylist entries: an unseen rule needs no
  argument against a recorded rejection. (Match on the bare key with a word boundary, or `S108`
  matches inside `S1084`.)
  **Run it on the FULL rule set, not on one facet call — the 100-value cap silently truncates it and
  the truncation looks like a clean result.** Platform's Java facet holds **161** rules; a single
  `facets=rules` call returns the top 100, all of which are documented, so the diff printed *zero
  never-mentioned rules across all three repos* and read like "the catalogue is exhausted". Splitting
  the same query by severity (`severities=BLOCKER`, then `CRITICAL`, `MAJOR`, `MINOR`, `INFO`) and
  unioning the five facets returned **30 never-mentioned rules** — the whole low-count tail, where a
  never-triaged rule lives by construction. Split by `languages=` too (`java`/`js`/`css`/`xml` each
  get their own 100). Cost: five calls per repo instead of one. Do not trust `len(values) < 100` as
  proof of completeness either — check it against the response `total` split you actually queried.
  **And it REGENERATES, so re-run it every time — a note here saying it is "essentially spent" is
  about the day it was written, not about the rule catalogue.** SonarSource ships new rules
  continuously: two runs after that note, the same three-call diff surfaced a whole new **`S93xx`
  generation** none of which existed when the corpus was written — `S9354` (16), `S9357` (18),
  `S9358` (6), `S9365` (3) and `githubactions:S7637` (3), **43 issues, 34 of them shipped in one
  sweep across all three repos**. Two properties make a fresh rule generation the best pool there
  is: nothing in `dropped-issues.md` or the OKF denylist argues against it, and it is dense in
  **commons and rendering at the same time** — which is what a multi-repo run needs and what the
  Java allowlist has not delivered in a dozen "commons and rendering are closed" confirmations.
  So open every run with this diff, not with the allowlist.
- **A "new" rule is often a RESTATEMENT of an old one — carry the old rule's drop index across.**
  `java:S9357` ("Make this anonymous inner class a lambda") is `java:S1604` with a new key and a new
  description, and its four commons sites were the same `AccessController.doPrivileged` lines
  `dropped-issues.md` had already ruled permanently un-convertible under `S1604`. Recognising that
  cost one grep of the drop index **by the code shape** (`doPrivileged`, `URIClassLoader`) rather
  than by key, and it saved a build round on a lambda that provably does not compile. So when the
  never-mentioned diff surfaces a rule, read its *name and compliant example* against the recorded
  rule names before treating it as unexplored — and record the new keys next to the old ones.
- **A fix that narrows a return value's RANGE without changing its meaning can be contradicted by a
  test asserting the old value.** Sibling of the recorded "a logging fix can be contradicted by a
  test asserting the RAW log message", and the same lesson: the contract the codebase *tests* can be
  narrower than the contract it *documents*. `Integer.compare` returns −1/0/1 where `a - b` returned
  the difference, and `ResourceReferenceHandlerTest` asserted `assertEquals(300, h1.compareTo(h2))`
  — over-specified (only the sign is in the `Comparable` contract), so adapting the test is the right
  answer, but it is a judgement call on someone else's test and belongs in the **judgement PR**, not
  in the mechanical batch. The cheap pre-check is a grep of the flagged class's module tests for
  `assertEquals(<number>,` near a `compareTo`/`compare`; the expensive one is what happened here —
  the failing module sat early in the reactor and `-fae` does not save the modules downstream of it,
  so one site cost a 12-module recovery run.
- **For a comment-only rule, the comment's CONTENT is a policy question — settle it before writing 80
  sentences.** The `S108` sweep wrote a true, careful rationale into each empty `catch` and the review
  came back: in XWiki a `catch` that neither rethrows nor logs is *a bug to fix later*, so the wanted
  comment is a `// TODO:` asking for the fix, not an explanation that reads as blessing the smell.
  Answered with one push (78 sites, two TODO forms) and both PRs stayed on track — but the cheap
  guard is to ASK FIRST: when the codebase has no precedent for how that shape is commented (grep for
  it — here there was none), put the two candidate wordings in the PR body, or in a one-line question,
  instead of shipping one flavour at scale. The rule is now in `okf/conventions/code-comments.md`; the
  per-rule text is in [rules/java-S108.md](rules/java-S108.md).
- **Outcome of the four-PR sweep: ALL FOUR MERGED the same day, 119 issues.** `S9142` (platform,
  after one style round), `S9142`+`S9355` (rendering) and the `S6213` judgement PR (commons,
  uncommented — the seventh denylist rescue to land) merged within hours; the `S108` comment PR took
  **four review rounds** and then merged too. So the risk in a sweep is not the transform, it is the
  PROSE: a batch that writes 80 sentences into the codebase is the one that gets read line by line,
  while 25 mechanical edits merge on sight — but the prose PR does land, so do not avoid the rule:
  budget the rounds, and ship the mechanical rules as their own PR (which is what let three of these
  land while the fourth iterated). Each round here was a concrete instruction answered by one
  verified push, never an argument.
- **Two review rounds on the same sweep, both concrete, both answered by a push in the same session** —
  the recorded posture (an ask with correctness content → push; a general style principle → verify
  first) held: "add TODO comments" and "never the one-line `/** x */` Javadoc" are both instructions,
  not opinions, so neither deserved a debate. Note the second one also asked to convert the *file's
  existing* one-line comments, which is the same intra-file-consistency principle reviewers apply to
  transforms.
- **A comment-only rule has SIBLINGS, and the family is worth enumerating once.** `S1186` (empty
  *method*) paid 172; its sibling `S108` (empty *block*) then paid 82 the next run, same shape, same
  reviewer-facing story. What makes such a rule cheap is stated in the rule's OWN description under
  *Exceptions* — S108's reads *"ignores code blocks that contain comments"* — so
  `api/rules/show?organization=xwiki&key=java:SXXXX` decides whether a "needs prose" rule is
  comment-remediable in one call. Do that before writing off `S1123`/`S1134`/`S1135`-shaped rules,
  and note the corollary: these rules survive every cleanup wave, exactly like the naming rules.
- **Before committing to a never-triaged rule, check whether one of its own flagged sites ALREADY
  contains the remediation the rule recommends.** One line read per site, and a hit kills the rule
  outright rather than merely making it expensive: `java:S8786` (non-linear backtracking, 17/2/3 —
  the only rule with fresh keys in commons *and* rendering that day) recommends possessive
  quantifiers, and `TagPlugin:479` is `split("\\s*+,\\s*+")`, fully possessive **and still flagged**.
  A fix that does not clear the issue is worthless here, so the whole rule went to
  [rules/java-S8786.md](rules/java-S8786.md) as a drop. Sibling of the `S108` move (read the rule's
  own *Exceptions* section) — both are one API/grep call that decides a rule before any real triage.
- **When the Java facet AND the JavaScript facet are both claimed, the answer is `languages=css`.**
  Recorded as "the only never-opened language facet left" and now opened: platform 19 issues over 7
  CSS rules, **13 shipped / 5 dropped / 1 false positive**, commons and rendering 0. Three things
  make it the best pool available on such a day: it is decidable from ~10 lines plus a scan of what
  sits *between* the two blocks an issue names (no whole-file reads, no dataflow), it sits in
  `xwiki-platform-web-war` where the `closure-compiler:minify`/`yuicompressor` executions are real
  verification, and **no agent PR had ever touched a `.css` file**, so nothing was claimed. See
  [rules/css-rules.md](rules/css-rules.md). Corollary for the find phase: run the facet query per
  `languages=` (`java`/`js`/`css`/`xml`) rather than trusting one call — the same 100-value cap that
  hides the low-count Java tail also buries the whole CSS facet under the Java rules.
- **The "deferred, NOT dropped" entries of `dropped-issues.md` are a standing invitation that
  EXPIRES — re-query the keys before planning a batch around them.** The recorded lever ("after
  choosing the batch, re-read the index for *deferred* entries and intersect their modules with your
  `-pl` list") paid 14 sites once. This run it returned mostly **GONE**: all 7 of the
  `java:S7476`/`S3706` `search-solr-api` keys and the whole `javascript:S6637` `dashboard.js` set had
  been fixed or claimed by another PR in the meantime. One `issues/search?issues=<keys>` over the
  deferred list costs one call and tells you which invitations are still open — do it before
  budgeting a module into the reactor for them.
- **A run can find the never-mentioned-rule diff genuinely EMPTY, and that is information, not a
  failed query.** Five severity-split facet calls per repo returned, across all three repos, only
  count-1 rules plus `javabugs:S6322` (rendering 2). When that happens the catalogue really is swept
  and the next lever is a *language* facet (above) or a re-derivation, not another rule hunt.
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
- **An `assert count == 1` that fires on a duplicated block is a finding, not an obstacle.** The same
  lazy-init block often appears twice in one long legacy file while Sonar flags only one of them
  (`XWikiRightServiceImpl`'s `grouplistcache`). Extend that site's `old` with the preceding unique line
  (a comment, or the statement above) and convert BOTH occurrences — leaving one half of a file
  converted is what a reviewer objects to. Count only the flagged key as fixed and say so in the PR.
- **When the `textRange` covers only PART of the flagged expression, the token BEFORE it is the free
  classifier.** For `S3252` the range is just the member name; reading the qualifier that precedes it
  split a 175-issue pool, with zero source reads beyond one line per issue, into 123 pure
  qualifier swaps (mechanical, 0 drops) and 52 sites where the qualifier's simple name EQUALS the
  declaring class's — those are not qualifier changes at all but import swaps, and in XWiki they mean
  abandoning `org.xwiki.text.StringUtils`, so they are a permanent drop. Derive the safe/unsure split
  from the one line the issue already points at before reading anything else; it is the same lever as
  `S117`'s locals-vs-parameters and `S1130`'s annotation bucketing.
- **A batch that swaps a type qualifier must fix imports in BOTH directions, in the same edit.** Add
  the declaring class's import (skip it when the class is in the file's own package — the compiler is
  not the guard here, a redundant import is a Checkstyle error) and remove the old qualifier's import
  once the swap orphans it, matching its simple name with a word boundary over the WHOLE file so a
  `{@link X}` in Javadoc still counts as a use. Insert the new import into the group whose first
  package segment matches, keeping that group alphabetical — inserting merely "before the first
  greater import" walks it across a blank-line group boundary. One file needed five imports removed
  and two added; getting either direction wrong fails only in `checkstyle:check`, i.e. after the
  tests, i.e. a whole build round.
- **A PROBE edit left in the working tree silently shrinks the batch — collect the sites from a
  pristine tree, or re-derive after the probe.** The `S5993` run applied four sites by hand to test
  the gate, then built its site table from the working copy; the collection pass required the flagged
  line to start with `public`, so those four now read as "already fixed" and were skipped. They were
  then reverted by the branch reset, so the batch shipped **158 of 162** and the gap only showed up
  when the PR-body count was cross-checked against the facet. Recovering it cost a second commons
  reactor. Two guards: run the collection pass before any probe (or after `git checkout -- .`), and
  always reconcile the applied-edit count against the rule's own live `total` per repo before writing
  the PR body — the same cross-check the recorded "re-derive the fixed count at PR time" rule asks for.
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
- **Write the equivalence-PROOF program BEFORE classifying the sites — its output IS the classifier.**
  For `javascript:S6582` the intuition "`a && a.b` is equivalent to `a?.b` whenever the result is only
  tested for truthiness" is right for a property access and **wrong for a call**: `?.` short-circuits
  only on `null`/`undefined`, so `0?.f()` evaluates `(0).f()` and throws where `0 && 0.f()` yielded `0`.
  A 20-line `node` program run BEFORE the triage moved 19 of 31 sites out of "trivially safe" and into
  "receiver must be checked"; run after the edits it would only have documented what already shipped.
  Same lever as the throwaway `javac`/`java` file for a Java question — and the same payoff, since the
  output is also the strongest thing to put in the PR body.
- **A signature-shortening fix can MOVE the issue instead of clearing it — check whether the removed
  argument's producer is itself a parameter of a non-`private` method.** The recorded `S1172` guards
  cover the overload collision and the orphaned local; this is a fourth shape and it costs nothing to
  test. Three of five `private` platform sites failed it: `AttachInputSocket`/`AttachOutputSocket`'s
  package-private constructors are called only from `AttachNode.input(BitField)` /
  `output(BitField, Entry)`, both **`protected`**, so removing the constructor parameter orphans the
  caller's own parameter and trades the issue for a fresh `S1172` that can never be fixed; and
  `DocumentLocaleReader#readXMLElement`'s `filter` cascades through a private `readDocument(…)` onto the
  **public** `read(XMLStreamReader, Object filter, …)`. Walk one level up the call graph per site and
  drop when the chain ends on anything non-`private`. Same reflex for any rule whose fix narrows a
  signature (`S1130` included) — the recorded "fix the whole file" advice only helps when the cascade
  terminates inside the file.
- **A RAW-typed local erases the generics of everything you read through it, so a retyping fix that is
  correct against the declaration still fails to compile.** `BaseCollection#getFields()` is declared
  `private Map<String, Object>`, so `for (String key : this.getFields().keySet())` compiles — but the
  second loop in the same method reads through `BaseCollection oldCollection = (BaseCollection)
  oldObject;`, a **raw** type, which erases the member's type arguments and makes `keySet()` a raw
  `Set`. `java.lang.Object cannot be converted to java.lang.String`, at minute 2 of a 23-module
  reactor. The fix is to parameterize the local (`BaseCollection<?> oldCollection = (BaseCollection<?>)
  oldObject;`) rather than reverting the loop — the helpers it is passed to take the raw type, so a
  wildcard is assignable and nothing else changes. Expect this on any `S4838`/`S6201`-style retyping in
  oldcore, which is full of raw locals: **when the fix retypes a value read off a variable, check that
  variable's own declaration for a raw type, not just the method's return type.**
- **A batch that REMOVES an element from a parameter/argument list needs three guards the obvious
  script does not have.** (Learned removing 41 unused private-method parameters; applies to any rule
  that shortens a call.) (a) **The collision guard must use the RESULTING arity**: asserting that only
  one declaration of that name has `nparams` arguments correctly refuses a same-arity overload pair,
  but misses the case where the *shortened* signature equals an existing one — a private
  `display(A, B, C)` minus one argument became the public `display(A, C)` it delegates from, i.e. a
  compile error found only at minute 2 of the reactor. Scan for `nparams - 1` too. (b) **Removing an
  argument can orphan the local that produced it** — here an `instanceof T name` pattern binding, so
  Checkstyle `UnusedLocalVariable` failed the module *after* its tests passed. Post-apply scan, nearly
  free: for every bare identifier removed from a call line, count remaining word-boundary occurrences
  in the file and flag `<= 1`. (c) **Removing element 0 leaves the next one holding its whitespace**
  (`f( b, c)`), which no compiler and no line-length check complains about — strip it in the same
  edit. Splitting the list itself needs a scanner that tracks parens, brackets, **angle brackets**
  (`Map<String, Serializable>` is one element) and string literals; plain `split(',')` is wrong.
- **An "unnecessary" cast on an argument of an OVERLOADED method is not a mechanical fix** — removing
  it can silently re-dispatch to a different overload and still compile. Check the callee's overload
  set before trusting S1905.
- **A COMPILE ERROR from a batch is sometimes the run's best finding — read it before assuming the
  script is wrong.** A rule whose fix moves code into a construct with *stricter static checks* makes
  the compiler audit code the old construct never audited. `java:S6880` (`if`/`else if` chain →
  `switch`) is the clean example: `switch` rejects a `case` dominated by an earlier one, while the
  identical order in an `if`-chain is merely dead code, so *"this case label is dominated by a
  preceding case label"* means the original chain has an unreachable branch (here
  `ExtensionUtils#wrap` could never return a `WrappingIndexedExtension`). The right move is **revert
  that one site, leave the issue open, and report the dead branch in the PR body** — deleting or
  reordering the branch is a product decision, and the report is worth more than the issue. Same
  reflex for any conversion into a construct with exhaustiveness, definite-assignment or
  effectively-final rules.
- **Deleting an unused private TEST helper orphans imports (Checkstyle) but must NOT orphan the
  fields it used — those are often component registrations.** Removing
  `EditModeResolverTest#wysiwygEditor` (a `java:S1144` site) left `Editor`, `EditorDescriptor` and
  `SyntaxContent` imported and unused, which `checkstyle:check` fails on *after* the tests, i.e. a
  whole build round; delete them in the same edit (word-boundary count == 1 ⇒ only the import
  remains). But the same deletion also left `@MockComponent private EditorManager editorManager`
  with no reader, and that field must **stay**: in the XWiki test framework a `@MockComponent` field
  registers a mock in the component manager for the class under test, so it is load-bearing even
  when nothing in the test file references it. Same reflex for `@InjectMockComponents`,
  `@RegisterExtension` and Mockito's `@Mock`/`@Captor` — an annotated field is never dead code, and
  Sonar's `S1068` exempts them too.
- **A fix that changes a MODIFIER can move the field into a different rule's scope — the clearest case
  is `final`.** `java:S1165` asks for `static String plugName` → `static final String plugName`, and
  `static final` is precisely what makes a field a *constant*, so `java:S115` (and Checkstyle's
  `ConstantName`, which XWiki enables in `checkstyle.xml`) then demand `CONSTANT_CASE`. The fix has to
  carry the rename or it trades one issue for another; here the field was package-private and read only
  by three `super(...)` calls in its own file, so `PLUG_NAME` was free. Ask it of any fix that adds
  `final`, `static`, `transient` or a visibility keyword: *which rule's pool does this field/method now
  belong to?* (The same rule's other sites are a clean drop for a different reason — `XWikiException`
  has `public` setters, so `final` does not compile at all.)
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
- **A fix that narrows a SIGNATURE cascades through the call graph in the same file — fix the whole
  file, not the flagged sites.** Sonar only reports the frontier: narrowing those sites exposes the
  next ring, so the rule regenerates in the very files you just shipped. A platform S1130 PR narrowed
  4 private test helpers; the next scan flagged 11 more helpers in the same class, and narrowing those
  would have exposed the 13 `@Test` methods that call them — three PRs for one file. Strip the whole
  file in one pass and let the compiler reject anything that really can throw (assert the match count
  so a partial sweep fails loudly). Count only the flagged keys as "fixed"; the pre-emptive ones are a
  bonus worth one line in the PR body. Generalises to any rule that changes what a method can throw or
  return.
- **A "hides the field" rename is provably safe once you scope it to the BLOCK, not the method.** Java
  shadows the field from the local's declaration to the enclosing block's closing brace, so every bare
  occurrence in that computed range is the local and nothing outside it is — brace-match forward from
  the declaration and substitute only there. This is what turns `S1117` (long recorded as "rejected,
  a missed occurrence silently re-binds") into a 0-drop batch, including sites where the local and the
  field share a type and the compiler could not have caught a partial rename. Three assertions carry
  it: the flagged line really is a declaration, the NEW name occurs zero times in the range, and the
  name never appears inside a string literal in the range.
- **A RENAME batch must never rewrite a `this.<name>` access.** When the flagged parameter shadows a
  field of the same name (very common in oldcore setters — `this.engine_context = engine_context;`),
  a plain word-boundary substitution over the method scope renames the field reference too and the
  file stops compiling. Use a `(?<![\w$])(?<!this\.)name(?![\w$])` pattern so the field keeps its own
  name; the resulting `this.engine_context = engineContext;` is correct and leaves the (separately
  flagged) field alone. The converse guard matters as much: assert the NEW name occurs **zero** times
  in the rename scope, which is what rules out a renamed local silently capturing a reference that
  used to resolve to a field.
- **A block-scoped rename has TWO scope cases, and one of them is not brace-matching.** For a plain
  declaration statement the scope ends at the `}` that closes the enclosing block (brace-match forward
  from the declaration to depth −1). But when the declaration sits in a *header* — `for (T x : …)`,
  `catch (E x)`, a parameter list — brace-matching from the declaration first counts the body's own
  `{`, so it runs past the method and swallows unrelated code. For those, walk forward to the header's
  closing paren, then take the block that follows: the scope is the declaration through the end of
  that block (which correctly includes the rest of the `for` header). Detect the case from the flagged
  line's first token, not from the rule.
- **A pattern variable (`if (x instanceof T name)`) is flagged by the same rules and is NOT
  brace-scoped** — its scope is flow-sensitive (the `if` body, or the *negation* when the condition is
  inverted). No brace rule models it. Fix those sites as a one-off exact-string edit over the two or
  three lines that use the binding, and keep them out of the scripted table.
- **A batch that INSERTS a declaration above an anchor lands between that anchor's Javadoc and the
  anchor itself.** Anchoring a generated `private static final …` on an existing field declaration
  puts the new field *under* the old field's `/** … */`, silently re-attributing the documentation —
  it compiles, and the only symptom is in the diff. Guard: if the anchor's preceding non-blank line
  ends with `*/`, insert above the whole comment block (or re-emit the comment after your
  declaration). It fired on two of four insertions in one batch. Same family as the empty-block
  insert: **look at what is immediately ABOVE and BELOW your insertion point, not just at the line
  you matched**.
- **Every batch that INVENTS a name must check the new name against the class's own field
  declarations** — otherwise the rename can re-create the very rule you are clearing (or a sibling
  one) under a different name, silently, with a green build. One regex over `^\s{4}(modifiers…)\s(\w+)\s*[=;]`
  gives the field set per file; intersect it with the new names before writing anything.
- **A rename that LENGTHENS the identifier will push some pre-existing ≤120 line over the limit — do
  not solve that by picking a worse name.** Give the batch script a second, ordered `POST_EDITS` list
  of exact `(old_text, new_text)` pairs applied *after* the renames, each asserted to occur exactly
  once, and use it to re-wrap those few statements onto a continuation line. Two of 107 sites needed
  it, and one of them (a line already at exactly 120 columns) had no shorter name available at all.
  This "scope-rename pass + exact-string post-edit pass" shape generalises to any batch where the
  mechanical transform is right but a handful of sites need hand surgery.
- **A line-length guard must compare against the SET of original lines, not zip by index.** As soon as
  one edit re-wraps a statement onto two lines, every later line pairs with the wrong original and the
  guard fires on pre-existing over-long lines. `for line in new: if line not in set(old_lines): assert
  len(line) <= 120` is index-independent and still catches exactly the lines you wrote.
- **A merged Sonar sweep is a RELIABLE source of fresh issues in the very files it touched — and the
  `git log` provenance check is what finds them.** The known form of this ("S1130 regenerates in the
  files you just fixed") generalises to any rule whose *fix shape* creates another rule's shape: #6178's
  `S3824` `Map.get` → `computeIfAbsent` conversion left the assigned-to local unread (a fresh
  `S1854`+`S1481` pair on one line) and introduced a `name ->` lambda parameter that shadows a field
  (a fresh `S1117`). So run `git log -1 --format='%h %s' -- <file>` over the candidate files at triage
  time, not only before deleting: a recent `[Misc] Fix various SonarQube issues` parent is **positive**
  evidence — it means your fix *completes* that one rather than re-fighting it, which is worth one line
  in the PR body and pre-empts "didn't we just change this?". The negative case (a JIRA-numbered commit
  that deliberately introduced the shape) is the one below.
- **Check the flagged line's own git history before DELETING anything — a rationale is not always a
  comment, sometimes it is a commit message.** The universal drop conditions say to respect a comment on
  the flagged code; that is not enough. A run deleted a `.toString()` from a log call whose `toString()`
  had been *added on purpose* four days earlier by `64ba541` (XWIKI-24665) — **the direct parent of the
  branch, printed in the `git log -1` output while the branch was being set up** and read straight past.
  The repo had already oscillated on those lines (#1871 stripped them, #1872/#1884 restored them), so the
  sweep re-fought a settled decision. Guard, cheap enough for any deleting rule: loop
  `git log -1 --format='%h %s' -- <file>` over `git diff --name-only`, and when a subject is recent and
  JIRA-numbered, `git show <sha> -- <file>` to check it did not introduce the shape you are removing.
- **Cross-check the drop index with a SUBSTRING test per key, never a regex that guesses the key
  SHAPE — and this is the single most expensive mistake available in the find phase.** A run built its
  "is this key already dropped?" set by regexing `dropped-issues.md` for
  `\bAY[\w-]{10,}\b` plus a UUID pattern. SonarCloud keys in these three projects start with **`AV`,
  `AW`, `AX`, `AY`, `AZ` and `Aa`**, so the set held 179 keys out of the ~700 the file actually lists,
  every non-`AY` key read as "fresh", and the run re-triaged from scratch ~60 keys the index already
  explained (`S9357` ×2 with the identical verdicts written down, `S2093` ×11, `S1185`, `S1118`,
  `S3824`, `S1871`, `S1640`, `S1643`, `S2440`, `S6916`, `S2177`, `S1192`). Worse, it *applied* two
  commons fixes the index had rejected on behaviour grounds, built them green, and had to revert them
  in a second commit on an already-open PR. The check that cannot fail is `key in
  open(dropped).read()` per key (or one `grep -F -f keys.txt`) — the file is one blob of text and the
  keys are literals, so no pattern is needed at all. **Verify the set size against the file**: a count
  far below the number of `` ` ``-quoted keys in the file is the symptom.
- **A drop-index entry that says "deferred, NOT dropped — build ROI" is a standing invitation, and the
  moment to cash it is when your reactor is already wide.** 14 `java:S1186` singletons, one per module
  across 14 modules, had been written off because "shipping it would add a whole module to the reactor
  for one issue" — and all 14 shipped for free in a run whose mechanical batch already needed a
  23-module reactor. So after choosing the batch, re-read the index for *deferred* (not rejected)
  entries and intersect their modules with your `-pl` list; it is the cheapest yield in the run.
- **"A comment would have to invent the intent" is a claim to re-derive, and the answer is usually two
  members away.** For a comment-only rule the recorded drop condition is truthfulness, which makes it
  tempting to write a site off from the empty body alone. Two of three `java:S1186` platform "real
  drops" had their reason in plain sight: `ComputedFieldClass#displayHidden` is empty because the
  class Javadoc says "a field **without storage**" and its `fromString`/`newProperty` siblings both
  `return null` under the comment "There is no content in a computed field";
  `UserInstanceOutputFilterStream#endUser` is empty because `beginUser()` twenty lines above builds the
  document and calls `saveDocument`. Read the CLASS Javadoc and the sibling members before concluding a
  reason cannot be stated.
- **When a dead-code drop is justified by "but the code documents intent", the fix is to move the
  intent into a COMMENT.** `java:S2198` on `WikiPageUtil.isValidXmlNameStartChar(char ch, …)` had been
  dropped because deleting the unreachable `(ch >= 0x10000 && ch <= 0xEFFFF)` row drops the
  supplementary-plane row of the XML `NameStartChar` production the table transcribes. That is right
  about the information and wrong about where it belongs: a comparison that can never fire is not
  documentation. Shipping the deletion *plus* a three-line note (the range exists in the spec, a `char`
  cannot reach it, supporting it needs an `int` code point) answers the objection instead of deferring
  to it, and it converts a permanent drop into a fix. Ask this of every "the dead code is a record of
  something" rejection.
  **But first ask WHY the code is dead — if a proper fix would bring your deleted line BACK, the line
  is evidence of a defect and deleting it is worse than the open issue.** This is the same `S2198`
  site, and the follow-up settled it against the cleanup: Vincent took the PR's stated open question
  to the forum, the accepted answer was to change the signature to an `int` code point, and with an
  `int` that unreachable `(ch >= 0x10000 && ch <= 0xEFFFF)` becomes the row that actually implements
  the supplementary half of the XML `NameStartChar` production — so the fix re-adds the line, drops
  the comment, and `S2198` stops being raised on its own. One sentence would have caught it:
  *"if someone fixed this properly, would my deleted line come back?"* Symptoms to check are a
  neighbouring method that already takes the wider type (`isValidXmlChar(int ch)` did) and a deleted
  line that loses a row of a spec the code transcribes. See
  [rules/java-S2198.md](rules/java-S2198.md).
  Net for the run: the technique is still right, and its first application was the wrong site — but
  the *outcome* was the run's best: #425 was closed and replaced by XRENDERING-814 (#426), which adds
  the `int` overloads, walks by code point and moves the `char` signatures to a new
  `xwiki-rendering-legacy-wikimodel` module, fixing a 20-year-old bug the Sonar issue was pointing at.
  **So a closed PR is not always the review verdict `learnings.md` records it as** (the recorded rule
  — "a silent close IS the review verdict, record the rule as a permanent drop" — holds for a *silent*
  close; a close that names a superseding PR is the finding being upgraded). And the mechanism that
  produced it is worth copying: the PR body **stated the open question** rather than hiding it, the
  maintainer took that question to the forum, and the community answered it. State the open question.
- **Consult `dropped-issues.md` for EVERY rule you shortlist, before reading ANY source — one grep of
  all the shortlisted keys, not one per rule you commit to.** This has now cost source reads twice:
  `S6035`/`S2093` in one run, `S1118`/`S3415` (rendering) in another, all four already recorded with
  the exact reason. The check is one call for the whole shortlist; skipping it is what makes the
  skip-index worthless.
- **A `dropped-issues.md` entry can CONFLATE two shapes under one rule heading — read the stated
  reason against the site you are holding, not just the key.** Three of the six Java fixes in one run
  were keys already listed as dropped: the `S1872` heading said "`instanceof` is not equivalent",
  which is true of the two `OfficeServerLifecycleListener` sites but not of the third
  (`"void".equals(returnType.getName())`, a fixed-class comparison), and the `S1193` heading said
  "duplicates the `throw`", which is a *solvable* problem (extract the message into a constant), not a
  drop condition. When you re-open a key, edit the entry into a **Correction:** line rather than
  deleting it, so the next run sees the reasoning and not just a shorter list. Same failure mode as
  the OKF-denylist entries that turned out wrong (`S1117`, `S3252`) — a one-line summary loses the
  shape it was about.
- **Cross-check EVERY shortlisted key against `dropped-issues.md`, including rules you found via the
  facet rather than the allowlist.** The allowlist query does this automatically; a rule you pull
  separately (because the facet surfaced it) skips the check silently, which is exactly how those
  three already-dropped keys got re-triaged from scratch.
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
- **STOP RUNNING THE ACCEPT LOOP FOR ISSUES THE PR FIXES — the skill now forbids it, and that
  supersedes every bullet below.** `xwiki-fix-sonarqube-issue` (1.5.x) says it outright: *"Never
  transition an issue the PR fixes. SonarCloud closes it as FIXED on its own at the next branch
  analysis after the merge. Accepted means 'won't fix': on a fixed issue it buys nothing, and hides a
  real defect from the quality gate if the PR never lands."* So the whole throttled-loop apparatus
  below (30-50 min of wall clock, ~2 requests per key, the confirm pass, the reopen-after-a-closed-PR
  repair) now applies to **only** the issues the code deliberately keeps — a `falsepositive` or an
  `accept` with a stated reason — which in this routine is close to zero, because analyzed-but-unfixed
  issues go to `dropped-issues.md` instead. Skipping it also removes the "a closed PR makes your
  SonarCloud comments wrong" failure mode entirely. Keep the mechanics below for the rare deliberate
  transition; do not budget wall clock for a sweep any more.
- **An *Accepted* issue still flips to FIXED on its own once the fix lands** — the accept is interim
  bookkeeping, not a permanent verdict, so it never leaves a wrong record. Measured on this sweep: hours
  after the Java PR merged its 25 keys read `FIXED` while the 23 JS keys (PR merged minutes earlier) were
  still `ACCEPTED`, i.e. the flip happens on the next analysis, per language. So do not re-transition
  anything after a merge, and do not read a lingering `ACCEPTED` as a failed accept — one
  `issues/search?issues=<keys>` distinguishes them.
- **Accept all issues in a loop** (per issue: `add_comment` + `do_transition accept`). Each issue is
  ~2 curls ≈ 4s, so 20+ issues blow a 2-min timeout — run the accept loop as a BACKGROUND task, and/or
  make it idempotent (re-query which keys are still OPEN). **Budget the wall clock from a measured
  per-key rate, not from a past run's total** — the proxy's throughput varies by an order of magnitude
  between runs (168 keys in ~30 min once, but only 50 keys in ~50 min on another, i.e. ~60 s/key). At
  the slow rate the loop outlasts the whole write-up, so launch it FIRST and check progress with one
  `issueStatuses=ACCEPTED` count query rather than tailing its log. **An unthrottled loop silently loses about
  half of them** — a 70-key run left 35 still OPEN with no error output — whereas a **0.3s sleep after
  EVERY POST** (comment and transition alike) landed 66/66 with nothing left for the retry pass. So
  throttle from the start and still run the confirm pass (**464/464 landed on the first pass, zero
  stragglers, ~35 min for 464 keys ≈ 4.5 s/key**, and **119/119 first pass at a similar rate a run
  later** — so the 0.3 s throttle costs nothing and the loop
  comfortably covers the whole write-up; earlier: **168/168 first pass**, but budget ~30 min of wall clock for 168 keys through this container's proxy — launch
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

- **A PR is gated by the repo's own `Analyze` workflow, and its verdict is "the issues YOUR OWN LINES
  carry" — but only since 2026-08-28, because this routine got it fixed.** Worth reading as a worked
  example of what to do when a gate failure is provably an artifact.
  The workflow (added the day before) originally waited on the SonarCloud project quality gate, which
  on a PR counts **every issue SonarCloud reports in a changed file**, whatever line it sits on. So a
  pre-existing dataflow finding in a file you edit is re-reported as *new code* and fails the gate
  (`C Reliability Rating on New Code`), and the small-diff `≤3% duplication on new code` condition
  trips the same way. **Identify it by arithmetic, which is free**: master held `javabugs:S2259` at
  oldcore `XWiki.java:4755`; a PR that replaced 13 lines with 6 at line 1501 saw it as new at
  **4748** (−7) and its sibling, which inserted one line at 1114, saw it at **4756** (+1). Same rule,
  same message, same untouched method, two deltas — stronger than a re-run, and the analysis is
  deterministic, so don't spend one. The duplication twin: two listeners already shared a ~14-line
  verbatim block, master reported **0** duplications for the file (the run sat just under the
  ~100-token threshold), and inserting the identical line into *both* copies pushed it over — 2 lines
  against a 40-line diff is 5.0%. Measure it with
  `api/measures/component?...&pullRequest=N&metricKeys=new_lines,new_duplicated_lines` (values live
  under `periods[0].value`, **not** `value` — a plain `.value` read prints `None` and reads like a
  missing metric) plus `api/duplications/show?key=<projectKey>:<path>&pullRequest=N`, and prove the
  copy-paste pre-dates you by repeating the call **without** `pullRequest`.
  **What to do, and the outcome that validates it: do NOT mutilate the batch.** Dropping the sites
  that cause the line shift is whack-a-mole (any edit above the landmine line re-triggers it) and
  fixing the underlying finding inside a `[Misc]` cleanup is out of scope (`javabugs:S2259` is
  OKF-denylisted precisely because each site needs its own dataflow argument). Post **one comment per
  PR** with the arithmetic, the sibling-PR confirmation and a proposed patch for the real defect —
  and read **every** failed condition first: a first comment claiming "the same and only reason"
  missed the duplication one and needed a correction. Vincent then **fixed the workflow** rather than
  taking any of the three options offered (master `1a2cae6d`, *"[Misc] Fail a pull request on the
  SonarQube issues its own lines carry"*: it drops `sonar.qualitygate.wait` and keeps only issues
  whose line the PR actually wrote), rebased both branches onto it himself, and `Analyze` went green
  with the batches untouched — ~4.5 h from the comment. So a precise, arithmetic-backed artifact
  report is worth far more than a defensive edit.
  **Residual state to expect**: the `SonarCloud Code Analysis` check (the SonarCloud app reporting the
  *project* gate) can still be red for the moved-finding reason; it is no longer the repo's verdict —
  `Analyze` is. Don't act on the app check alone.
  **But the arithmetic proof is not always available, because a `javabugs:` finding can exist ONLY in
  the PR analysis.** The recorded proof (same rule, same message, two line deltas matching the diff)
  assumes the finding is on master at a shifted line. It sometimes is not: platform #6273 failed
  `new_reliability_rating` on two `javabugs:S2259` in `DownloadAction.java:428/432` while master —
  analyzed four hours earlier, and holding 127 `S2259` project-wide — had **zero** open issues of
  that rule in that file. So don't spend calls hunting for the master twin; when it is absent, the
  argument that stands is **distance plus mechanism**: name your actual hunks (here line 25, an import
  removal, and line 82, `Arrays.asList` → `List.of` in a static field initializer), show they are
  ~350 lines from the finding, and state that there is no data path from them to the property the rule
  is about (a method parameter's nullness). Add that `Quality / Analyze` passes on the same commit and
  that a re-run is pointless because the analysis is deterministic. Post it as one comment and leave
  the batch alone.
  **`@SuppressWarnings("javabugs:SXXXX")` IS honoured by the dataflow engine — verified, so a genuine
  `javabugs:` false positive is resolvable in code.** This was an open question (the `S8786` hazard:
  does the remediation actually clear the issue?) and the rule catalogue does not answer it. Two
  things settled it, in order: XWiki already carries `@SuppressWarnings("javabugs:S2190")` in
  `SolrCoreInitializer` and `AuthenticationFailureStrategy` (the in-repo-precedent grep again), and
  then the live confirmation — a `javabugs:S2259` annotated this way on `XWiki#checkDeletingDocument`
  **disappeared from the next PR analysis** while four other `S2259` in the same file were still
  reported. So for a `javabugs:` finding you can prove unreachable, the OKF's normal false-positive
  route (suppress + `//` reason) works; you are not forced into a null guard.
  **A null guard is often the WRONG remediation anyway — check what the method IS.** On
  `checkDeletingDocument` (a permission check) an `if (x == null) return;` would report "delete
  allowed" without having checked anything: strictly worse than the NPE. Reserve a guard for a method
  whose null return already means "nothing found" (`DownloadAction#getAttachment`, where returning
  `null` feeds an existing not-found path — and verify the value is dead afterwards, or the guard just
  relocates the NPE). State the choice in the PR and offer the alternative.
  **And the set of `javabugs:` findings a PR analysis surfaces for one file is NOT STABLE between
  runs** — #6272 reported exactly one `S2259` in `XWiki.java`, its prerequisite PR reported four
  *different* ones, and master's snapshot listed neither set. So never promise that fixing the finding
  a project-gate check named will turn that check green: say `Analyze` is the verdict and flag the
  instability rather than letting the reviewer discover it.
  **And the duplication twin has a specific trigger this routine will keep hitting: fixing the SAME
  rule in both halves of an already-duplicated block.** Platform #6272 failed
  `6.7% Duplication on New Code (≤3%)` = **2 duplicated lines / 30 new lines**, because its
  `java:S1940` fix rewrote one line inside *each* of two verbatim 19-line copies in `XWiki.java` (the
  `Accept-Language` parsing in `getLanguagePreference` and `getInterfaceLanguagePreference`).
  `api/duplications/show` proves the copy-paste pre-dates the PR — master `[3458, 3565]`, PR
  `[3456, 3563]`, both size 19 — and note **both offsets are −2, the same shift as the reliability
  finding** (master `S2259` at 4788, PR at 4786), because the PR deletes two `import` lines above
  everything. One arithmetic story covers both failed conditions when the shift has a single cause;
  say so rather than writing two unrelated explanations.
  The tempting "fix" — convert only one copy — is the wrong call: it clears the condition and leaves
  the same expression written two ways in one file, which is the intra-file inconsistency reviewers
  object to. State that trade-off explicitly in the comment and offer a JIRA for the real
  de-duplication.
  **Known landmine, still open on master: oldcore `com/xpn/xwiki/XWiki.java:4755`**
  (`checkDeletingDocument` dereferences its `document` parameter with no null check). It no longer
  gates a PR, but it is the finding those two PRs inherited.
- **The other way `Analyze` fails you is NOT an artifact — the gate is right, and it is preventable
  by one query per changed file BEFORE you push.** Distinct from the line-shift case above: there the
  finding moved and the gate was wrong; here the pre-existing finding sits **on a line your fix
  rewrites**, so by the gate's own rule ("the issues your own lines carry") it is yours. This bit two
  of 40 sites in one sweep — commons `MockitoComponentMocker#registerMockDependencies` (`S1130`,
  declaration line carried `S3776` complexity 22) and platform `RightsManager#fillLevelTreeMap`
  (`S1172`, same line carried `S3776` complexity 27). Neither fix changed the complexity by one point;
  rewriting the line was enough.
  **The pre-check is ~10 lines and needs no source reads**: parse `git diff -U0 <base> <branch>` hunk
  headers into the set of *written* line numbers per file, then one
  `issues/search?componentKeys=<projectKey>:<path>&issueStatuses=OPEN` per changed file, and print any
  open issue whose `line` falls in that set and is not one you are fixing. Run it before the first
  push; it caught the platform one *while its `Analyze` was still `in_progress`*, so that PR never
  went red at all, and re-running it afterwards is the proof to put in the reply.
  **The shape to expect it on**: any rule whose fix rewrites a **method declaration line** — `S1172`
  (drop a parameter), `S1130` (drop a `throws`), `S1611`, and the `S116`/`S117`/`S9149` renames —
  because a declaration line is exactly where the *method-level metric* rules are reported: `S3776`
  cognitive complexity (platform 218 / commons 67 / rendering 26, i.e. everywhere), `S107` too many
  parameters, `S1541`. A long legacy method with an unused parameter is almost by definition also
  complex, so the two pools overlap heavily. Cheapest form: at triage time, when bucketing such a rule
  by visibility, also drop any site whose declaration line appears in the project's open
  `S3776`/`S107`/`S1541` line set — one extra query for the whole batch.
  **When it fires, DROP the site** (reducing the complexity is a refactor, not a Sonar cleanup) and do
  the three-part cleanup: revert the file to `master` content (`git diff <master> -- <file>` empty is
  the whole re-verification, no rebuild needed), **reopen** the issue in SonarCloud with a correction
  comment naming the PR, and say in the PR's *Clarifications* what was dropped and why — the gate
  agreeing with you is a better line in a PR body than a green run you had to argue for.
  **"Drop the site" is only the answer when the colliding issue is a METRIC — ask first whether the
  collision is itself mechanical, because then it is yield, not a drop.** The recorded cases were all
  `S3776`/`S107` (a refactor, so a drop), which made the pre-check read as a drop-detector. It is
  really a *co-location* detector, the same lever as the `S7781`+`S6397` pairing: of 4 collisions in
  one JS batch, 3 were `javascript:S2814` on a `var` the batch was converting to `const` (and
  ignoring them would not merely have failed the gate — `let`/`const` cannot be redeclared, so the
  file would have stopped parsing) and 1 was `javascript:S4138` on a `for (var i …)`. Fixing them in
  the same edit turned a 4-site drop into **+9 issues** (6 `S2814`, 2 co-located `S2392`, 1 `S4138`).
  So run the pre-check BEFORE deciding the batch's contents, not just before the push, and classify
  each hit: metric rule ⇒ drop the site; anything with a mechanical fix ⇒ fold it in. **Outcome:
  `Quality / Analyze` went green first try on all three PRs of that sweep** (platform #6303/#6304,
  commons #1946), and so did the `SonarCloud` project gate — no moved-finding artifact, no comment
  to argue. Re-running the check after applying is what makes the zero meaningful; it costs one
  `issues/search` per changed file.
- **`xwiki-rendering` master cannot run ANY test right now, and the fix is a LOCAL, REVERTED pom
  edit — not a dropped verification and not a repair commit.** `xwiki-rendering/pom.xml` (lines
  124-129) pins a surefire `listener` property to `org.xwiki.test.CaptureConsoleRunListener`, a
  JUnit 4 class that no longer exists in `xwiki-commons-tool-test-simple` (the current SNAPSHOT
  ships only `org.xwiki.test.junit5.CaptureConsoleExtension`, and the old name appears nowhere in
  the `xwiki-commons` sources). Every rendering module whose surefire forks dies in the *booter*:
  `SurefireReflectionException: java.lang.ClassNotFoundException: …CaptureConsoleRunListener`,
  `Tests run: 0`, and `-fae` does not save the modules behind it — a 5-module reactor reported one
  FAILURE and four SKIPPED. Two things to do with it: (a) recognise it instantly — a
  ClassNotFound in the *booter* is a POM/classpath fact, so a diff of `src/main/**.java` cannot
  cause it; (b) **delete those six pom lines locally, build, then `git checkout -- pom.xml` before
  committing**, which turned "no verification at all" into 1255 tests green across 5 modules. Put
  the mechanism, the exact error and the fact that the edit was reverted in the PR's *Executed
  Tests* section — that is the recorded "state the breakage precisely, don't repair it in a cleanup
  PR" rule, with the verification actually done rather than skipped.
- **A JavaScript batch DOES have Maven verification — the earlier "none exists" note was wrong.**
  `xwiki-platform-web-war/pom.xml` runs `closure-compiler:minify` (three executions: strict,
  non-strict, merge) and `yuicompressor:compress` over every resource, so
  `mvn install -Plegacy,quality -pl xwiki-platform-core/xwiki-platform-web/xwiki-platform-web-war`
  really does parse and minify each file you touched. Cost: **6:18 cold, 0:32 warm** — cheap enough to
  run per branch. There are still no unit tests for these files (they are outside the Nx/pnpm
  workspace, which covers only `livedata`, `ckeditor`, `blocknote`), so the module build is a syntax
  and structure gate, not a behaviour one.
- **The behaviour evidence is still yours to produce**: `node --check <file>` on every modified file,
  plus a **`node` program proving each transform's equivalence** — `Number.parseInt === parseInt`, a
  loop over `U+0000`–`U+1117F` comparing two regexes, `(x ? x : y) === (x || y)` over every
  falsy/truthy shape, `push(a,b,c)` vs three pushes. Quote the actual output in the PR body, one line
  per rule. For a DOM rule (`S7762`/`S7768`) there is nothing to run: state the receiver/`parentNode`
  identity per site instead, quoting the line where the receiver was assigned.
  **An equivalence proof answers "is it safe", NOT "is it the form we want" — pre-empt the second
  question too.** A batch whose body proved every transform byte-identical still drew two style
  comments from the front-end reviewer: `||` should have been `??` (the codebase prefers the operator
  that does no type conversion, even though the flagged ternary tested truthiness), and a comment
  relocated by an `else`→`else if` merge should have been attached to the condition rather than to the
  first statement. Both were right. So for each rule state *why this form and not the other equivalent
  one*, and when a transform MOVES a comment, say where it went and why. Bonus: chasing the `??`
  question turned up a latent trap `||` had been masking, which is the kind of thing that converts a
  style nit into an accepted fix — so investigate the objection rather than just complying.
  **A JS batch is SOMETIMES routed to a front-end reviewer — it is not the default, so do not budget a
  day of latency for it.** The `S7773`+`S6353` batch (#6186) was ("this is javascript and I'd like an
  opinion from someone with more expertise, cc @manuelleduc"), and merged ~7 h later with no change
  requested. But the later `S6582`+`S7765` batch (#6201, 43 issues over 18 files) was **merged directly
  with no routing and no comment**, 14 minutes after its own judgement sibling. What both have in common
  is the thing worth copying: write the body for a reviewer who did none of the analysis — the
  equivalence proof per rule, and the list of sites you deliberately did NOT convert. Original note,
  still true when routing does happen:
  **Expect a JS batch to be routed to a front-end reviewer even when Vincent LGTMs it** ("this is
  javascript and I'd like an opinion from someone with more expertise, cc @manuelleduc") — so write the
  body for a reviewer who did none of the analysis: state the equivalence proof per rule and list the
  sites you deliberately did NOT convert. An LGTM on a JS batch is not a merge signal; do not treat the
  PR as finished. **Outcome datapoint: it works** — a 43-issue `S7773`+`S6353` PR (#6186) got Vincent's
  LGTM, then `@manuelleduc`'s formal approval ~7h later with no change requested, and merged. So the
  provable JS subset (`parseInt`/`parseFloat`, the `Number.isNaN`-on-a-Number sites, regex character
  classes) clears specialist review as-is; the routing costs a day of latency, not a rework.
  **Second outcome datapoint, and it widens the provable subset**: a 32-issue mechanical PR plus a
  3-issue judgement PR over 11 more JS rules (`S6660`, `S7778`, `S6637`, `S7762`/`S7768`, `S6676`,
  `S7759`, `S1481`, `S7776`, `S6644`, `S7723`, `S7760`) **both merged the same day**, `@manuelleduc`
  raising exactly one ask — intra-file API consistency, answered with a push. So the JS pool is
  ordinary reviewable work, not a specialist minefield; what the review actually rewards is the
  written-out equivalence per rule plus a per-site "what could differ" table on anything uncertain.
  **Second datapoint, and the reassuring one: a JS batch survives being wrong twice.** A 23-issue PR
  (#6197) drew style objections on two rules, went through two push-and-revert cycles that both landed
  back on the originally-pushed code, and `@manuelleduc` merged it himself ~35 minutes after the review
  started (~33 h after opening). Prompt reverts with the reasoning stated are not held against the PR —
  so when a reviewer's principle turns out not to apply, say so and put the code back rather than
  defending the change or withdrawing the PR.
- **A Java+JS split verifies in ONE reactor — put `web-war` in the `-pl` list next to the Java
  modules.** 4 Java modules (oldcore, notifications-filters-api, store-filesystem, filter-stream-xar)
  **plus** `xwiki-platform-web/xwiki-platform-web-war` ran **11:21** on a cold-ish `~/.m2` with
  **1302 tests** green (oldcore 1149) and the WAR leg at 1:32 doing all four
  `closure-compiler:minify` / `yuicompressor:compress` executions. So the "apply both halves, build
  once, split by file" trick works across the Java/JS boundary too, not just between two Java branches
  — and it costs one reactor for two PRs.
- **The `cd`-per-`mvn` guard: put it in unconditionally and do not trust EITHER cwd signal.** The Bash
  tool sometimes reports "cwd was reset to /home/user" and sometimes silently keeps the repo — a
  background `bash build.sh` with a bare `mvn` line happened to inherit the right repo and built fine,
  which is luck, not a rule. Corollary the other way: an `awk`/`grep` guard that demands
  `^cd /home/user/<repo> &&` on every `mvn` line will "fail" a script that actually works, so fix the
  script rather than debating the guard — one edit, no build round lost either way.
- **A batch verified by "the compiler is the whole verification" must include every compiler that sees
  the member — and in XWiki one of them is `ajc`, in a DIFFERENT module.** The `*-legacy-*` modules
  compile `src/main/aspect/**/*Compatib*Aspect.aj` with AspectJ, and an inter-type declaration
  (`public String BooleanClass.displaySelectSearch(…)`) is compiled *as a member of its target class*,
  so it can call that class's **`private`** methods from another module, in a source file `javac`
  never reads and Sonar never scans. That is how the previous run's merged `S1172` PR (#6214) left
  `master` red: oldcore compiled with 1149 tests green, then `xwiki-platform-legacy-oldcore` failed
  `ajc` (`The method getDisplayValue(int) … is not applicable for the arguments (XWikiContext, int)`).
  Two cheap guards for any batch that changes a member of an oldcore class: grep
  `--include=*.aj` for the member name, and put the matching `*-legacy-*` module in the `-pl` list.
- **Datapoint for a one-token three-repo sweep across 33 modules, 162 sites** (cold `~/.m2` for
  platform): commons 7 modules **2:07** (357 tests) + rendering 5 modules **2:03** (1483) + platform
  **21** modules incl. `oldcore` and `legacy-oldcore` **10:30** (2280, oldcore 1188) ≈ **15 min** for
  4120 tests, `revapi:check` green in all 33. A one-keyword edit costs the same reactor as a big one,
  so pick the widest pool the rule offers rather than trimming modules.
- **A second pass over the same modules reports FEWER tests, and that is the Develocity build cache,
  not a regression.** Re-running the commons leg with one module added showed `surefire` executing but
  printing no `Tests run:` for the four modules whose sources had not changed since the first pass
  (their reactor times dropped to 2-8 s), while the changed modules ran their suites normally and the
  build was `BUILD SUCCESS`. So when you re-run a leg to pick up a late site, quote the UNION of the
  two passes in the PR body, and don't read the smaller number as tests having been skipped by the
  change.
- **Datapoint for a three-repo chain from a COLD `~/.m2`, 38 sites**: commons 5 modules **4:02**
  (651 tests, extension-api re-run separately) + rendering 1 module **0:36** (93) + platform **14**
  modules incl. oldcore and legacy-oldcore **13:16** (1831 tests, oldcore 1160) ≈ **18 min** for 2575
  tests. A cold 14-module platform reactor with oldcore in it is still under a quarter of an hour —
  `-fae` kept the one commons compile failure from costing anything but that module, and re-running
  it alone was **36 s**.
- **XWiki is on Java 21** (`xwiki.java.version` in the root pom, `xwiki.java.version.support` 25), so
  Java-21 language rules are FAIR GAME, not "too new". `pool-state.md` had rejected `S6885`
  (`Math.clamp`) and `S6916` (pattern-match guards) with "Java 21" written next to them, which reads
  like a blocker and is not one — check the pom before writing off a modernization rule on version
  grounds. `S6880` (`if`-chain → `switch` pattern expression) was found this way.
- **Datapoint for a three-repo comment-and-constant sweep** (warm `~/.m2`, 119 sites): commons 1
  module **223 tests** + rendering 2 modules **352** + platform **11** modules incl. oldcore and
  `legacy-oldcore` **1501** (oldcore 1149) = **2076 tests**, all green, in a single chained
  background run of well under the 10-minute tool timeout. An 11-module platform `-pl` list
  spanning `xwiki-platform-core` AND `xwiki-platform-tools` works from the repo root in one reactor.
- **Datapoint for a three-repo annotation-only sweep across 77 modules** (warm `~/.m2`, 464 sites):
  commons **22** modules **6:10** (1177 tests) + rendering 3 modules **1:26** (524) + platform **52**
  modules incl. oldcore **25:37** (2989) ≈ **33 min** for 4690 tests. A 52-module platform `-pl` list
  is entirely practical; `revapi:check` ran in all 77 and passed.
- **Datapoint for a signature-narrowing three-repo sweep** (warm `~/.m2`): commons 5 modules **3:35**
  (336 tests) + rendering 1 module **0:49** (182) + platform 10 modules **1:49** — then a re-run of
  the 5 modules the platform failure had SKIPPED, **7:28** (1476 tests, oldcore 1149 of them, 3:45).
  Two lessons in the same run: `-fae` does **not** save the modules that depend on the failed one
  (oldcore sat behind a leaf `sheet-api` compile error and was skipped along with four others), and
  re-running only the failed-plus-skipped set is a 7-minute recovery rather than a second full
  reactor. Whole three-repo chain incl. the recovery ≈ 14 min for 2300 tests.
- **Datapoint for a narrow three-repo `src/main` sweep** (warm `~/.m2` for commons/rendering, cold for
  the platform leg): commons 2 modules **3:11** (33 tests) + rendering 2 modules **1:05** (214) +
  platform **14** modules incl. oldcore **13:29** (1623) = **~18 min** for 123 sites, all green.
- **Datapoint for a thin-spread single-repo sweep** (warm `~/.m2`): **17 platform modules** including
  both `oldcore` and `legacy-oldcore`, 32 files changed, **15:49** with **1605 tests** green (oldcore
  1145 of them). Two commits verified in that one reactor and split into two PRs afterwards.
- **Datapoint for a two-repo chain from a COLD `~/.m2`**: commons 5 modules **5:00** (620 tests) +
  platform **12** modules incl. oldcore **10:57** (1794 tests) = **~16 min** wall clock end to end,
  both green, downloads included. oldcore alone was 1143 of those tests. A 12-module platform reactor
  with oldcore in it stays cheap even cold — keep the thin-spread sites.
- **Datapoint for a two-PR-in-one-reactor split** (warm `~/.m2`): commons 1 module **2:49** (89 tests)
  + platform **22** modules incl. oldcore **16:52** (2231 tests) verified BOTH halves of a 107-site
  rename (58 test-file sites and 49 `src/main` sites in disjoint files) in one pass, then the branch
  was split by file set. Two PRs for one build.
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
- **Datapoint for a 9-module reactor spanning Java AND two `*-war` JavaScript modules** (warm `~/.m2`):
  oldcore 7:31 (1149 tests), rest-server 1:55 (131), search-solr-api 1:28 (120),
  legacy-events-hibernate-api 0:42 (41), chart-macro 0:25 (22), notifiers-default 0:54 (16),
  **web-war 1:16** and **tree-war 0:15** (both running all their `closure-compiler:minify` executions)
  = **15:11 for 1479 tests**. A JS batch that spills into a SECOND `*-war` module (here
  `xwiki-platform-tree-war` for one site) still verifies in the same reactor — add the module, don't
  drop the site.
- **A logging fix can be contradicted by a TEST asserting the RAW log message.** Converting a
  concatenated log call to the SLF4J parameterized form (`warn("[DEPRECATED] " + m)` →
  `warn("[DEPRECATED] {}", m)`) renders identically but changes `LogEvent.getMessage()`, and
  `LoggingScriptServiceTest#deprecate` asserts exactly that string — i.e. the flattened form is the
  contract, so revert rather than adapt the test. Pre-check for free by grepping the module's tests for
  the message's literal prefix. Generalises to any fix that changes a log call's *pattern/argument
  split* rather than its rendered output, and it is invisible to the compiler.
- **`-fae` is what lets a one-site failure cost one site instead of one run.** With it, the single
  failing module was the only failure and all eight others' greens stood, so the batch shipped minus
  that site in the same turn — no second reactor. Use it by default on a multi-module Sonar batch, not
  only when a failure is *expected*.
- **Datapoint for a two-batch single-repo sweep** (warm `~/.m2`, 40 sites, 33 files): **23 platform
  modules** incl. `oldcore` (1163 tests) and `legacy-oldcore`, verified in two passes because of one
  Checkstyle-metric failure — first pass 20 modules / **2268 tests** green, then the recovery pass
  (the failed module plus the 3 the reactor skipped behind it) 4 modules / **135 tests** in **~2:40**.
  Two PRs out of that one reactor. Recovery after a *metric* failure is as cheap as after a compile
  error, so do not re-run the whole reactor.
- **After a COMPILE error, rebuild only the failed module plus the ones the reactor skipped behind
  it.** The modules that already reported `Tests run:` in the failed run are green for sources you
  have not touched since, so re-verifying them is pure wall clock (a 3-module re-run replaced an
  11-module one). Confirm the claim first with `git diff` between the two applications of the batch —
  if the second application changed only the failing file, the earlier greens still hold.
- **A STALE `target/` in a working copy that has served several reactors produces the exact same red
  as the order-dependence below — and the recorded "build a master worktree" control MISDIAGNOSES it,
  because a fresh worktree is clean by construction.** Cost four builds and a wrong conclusion:
  `legacy-oldcore` failed **30/48** with `Failed to get [role = DocumentReference hint = [current]]`
  three times in `/home/user/xwiki-platform` (twice on the change, once on a bisected commit that
  only added a null guard in an unrelated action + its test), while a `git worktree add /tmp/masterwt`
  control passed **48/48** on the identical two-module reactor. That pattern reads as "master green,
  mine red ⇒ my regression" and it was wrong: `mvn clean install` on the *same commits in the same
  directory* went green (oldcore 1187, legacy-oldcore 48/48). The confound was the worktree's
  freshness, not master's code — a stale woven-aspect / `META-INF/components.txt` state in
  `legacy-oldcore/target` left over from earlier, differently-scoped reactors.
  Three consequences: (a) when a failure is a **component-descriptor lookup** rather than an
  assertion, suspect stale `target/` before suspecting your diff; (b) the honest control is `clean` on
  **your own tree in place**, not a fresh worktree elsewhere — a worktree changes two variables at
  once; (c) once a session has run reactors of different shapes over the same checkout, put `clean`
  in the verification build. It cost ~30 s here (2:03 vs 1:41 for oldcore) and would have saved three
  builds. The bisect is what exposed it: a commit that *cannot* cause the failure still causing it is
  the signal that the variable under test is not the code.
- **`xwiki-platform-legacy-oldcore`'s JUnit4-era tests are ORDER-DEPENDENT — one red run there is not
  a regression.** A standalone `-pl legacy-oldcore` run failed 30 of 48 tests with
  `Can't find descriptor for the component … DocumentReference hint [current]` at `setUp`, then the
  SAME tree passed 48/48 twice in the oldcore+legacy reactor, and a pristine master worktree also
  passed. The failing tests set static `Utils` component-manager state, so the verdict depends on
  execution order. Two cheap checks settle it before you believe a red: re-run the same tree (a
  disagreement between two runs IS the proof), and build the same modules from a master worktree
  (`git worktree add /tmp/masterwt <masterSha>` — cheap, and it also re-installs master's artifact,
  so rebuild your own module afterwards). Do not report such a red as "flaky" without both.
  **Correction, measured: "the oldcore+legacy reactor" is NOT what saves it — the reactor's WIDTH is.**
  A 2-module `-pl oldcore,legacy-oldcore` run fails **30 of 48** with that exact signature
  (`java.lang.RuntimeException: Failed to get [role = DocumentReference hint = [current]]` at
  `setUp`, via `Utils.getComponent`), twice in a row, while a **23-module** reactor containing the same
  two modules and the same tree passes **48/48**. So do not use a narrow oldcore+legacy re-run as the
  "same tree" check.
  **The cheapest decisive proof is a one-file A/B, not a master worktree**: revert *only* the file you
  suspect (`git checkout <masterSha> -- <file>`), re-run the identical narrow composition, and see the
  identical 30/48. That took **1:57** — against ~10 min for a worktree — and it exonerates the change
  outright rather than arguing about it. It works because the composition, not the content, is the
  variable; use it whenever a red appears in a *narrower* re-run than the one that was green.
- **A module can be red on master because of an EXTERNAL dependency, and the proof costs nothing.**
  `xwiki-platform-security-authorization-api` failed `testCompile` on `RightSetTest#testSetEquals()`
  — an `@Override` against a method `commons-collections4`'s `AbstractSetTest` no longer declares. The
  argument that settles it without a second build: **the failing file is byte-identical to
  `origin/master` (`git show --name-only <yourCommits>` does not list it) and nothing in your diff
  touches the classpath**, so no source edit of yours can produce it. Then drop that module from `-pl`
  and re-run only the modules the reactor marked SKIPPED behind it (they resolve the excluded module as
  a remote SNAPSHOT); keep the fixes you made in it, and state the file, the line and the evidence
  under *Clarifications*.
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
- **When the master breakage is THIS ROUTINE'S OWN regression, "don't repair master in a cleanup PR"
  becomes "repair it in its own PR, the same run".** A reactor failure that traces back to the
  previous run's merged Sonar PR is not someone else's problem and nobody else is fixing it: it is
  blocking every build of that module, including the verification of the batch in flight. The shape
  that worked (platform #6218, one file, two lines): keep the cleanup batch untouched on its own
  branch, cut a third branch from `master` with only the repair, build the affected module with
  *both* applied (that single build proves the repair works AND verifies the batch's sites in that
  module), then state in the cleanup PR's *Clarifications* that the module is red on `master`, why,
  which PR fixes it, and that it builds green with the fix on top. Name the offending PR and the
  mechanism in the repair PR's description — a regression report is more useful than a silent patch.
- **Do not repair unrelated master breakage inside a Sonar cleanup PR.** Ship the batch, and state the
  breakage precisely in *Clarifications*: the offending file and line, the commit that introduced it,
  the evidence that your diff is elsewhere, and an offer to rebase once master is green. Fixing it
  would muddle the review and can conflict with whoever is already repairing it.
- **Checkstyle METRIC rules are a drop condition, and the 120-column rule is not the only one.** Any fix
  that ADDS a statement or LENGTHENS a boolean expression can be rejected by a metric the line-length
  guard cannot see: **`BooleanExpressionComplexity` (max 3 operators)**, **`ExecutableStatementCount`
  (max 30 per method)** and — third metric, and the one that cost this routine its most recent build
  round — **`CyclomaticComplexity` (max 10 per method)**, where a `case` label counts and the
  `else if` it replaced did not, so an `if`-chain → `switch` conversion (`S6880`) raises the count by
  one per branch — and now a **fourth**, **`ClassFanOutComplexity`** (max 20 referenced types per
  class), which is the first one triggered by a fix that adds no statement and lengthens no
  expression: `java:S3398` *relocates* a method into a class, so the class inherits every type that
  method mentions (22 > 20 on commons `ResourceLoader.JarInfo`). So the generalisation is wider than
  "control-flow shape": treat any fix that changes a method's control flow **or moves code between
  classes** as metric-exposed. The list is not exhaustive; pre-count in the apply script. They run in `checkstyle:check` *after* the tests, so each
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
  ambiguity was settled). **It settles a REVIEWER'S objection too, and that is its highest-value use.** Asked
  "isn't this change less performant?" about `List#reversed()`, a 30-line `java R.java` (single-file
  source mode, no project, no JMH) printed the view's runtime class
  (`java.util.ReverseOrderListView$Rand`), proved it is a lazy view by mutating the base list, and
  timed both loop forms over 5M elements in alternating rounds. Post the actual output, and label a
  crude microbenchmark as one — "no measurable penalty" is defensible where "faster" is not.
  **Outcome datapoint: the PR merged four minutes later with no further discussion** — a measured
  answer plus an explicit offer to drop the one contested site resolves a performance objection,
  where an argument from first principles would have invited a second round.
  **The same throwaway file settles EQUIVALENCE questions, not just compile
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
- **A `cd` into the SCRATCHPAD mid-chain breaks every later git command in the same call.** Third
  symptom of the known cwd hazard, and it is not about multi-repo chains:
  `git checkout -B <b> <sha> && cd <scratchpad> && python3 apply.py --write && git add -A && git commit`
  runs the checkout in the repo and the commit in the scratchpad (`fatal: not a git repository`), and
  `git diff --name-only` degrades into `git diff --no-index` *usage output* rather than an error, which
  reads like a flag problem. Put `cd /home/user/<repo> &&` on every git line too, or run the apply
  script by absolute path and never leave the repo.
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
- **Marking a review thread RESOLVED is not possible here** — it is a GraphQL-only mutation and GraphQL
  is blocked for the session, and the thread IDs it needs come from GraphQL too. Answering in the thread
  is the only closure available, so make the reply say plainly what was done (and in which commit); do
  not spend calls hunting for a REST route.
- **Vincent's PR lock blocks YOUR OWN comments too** (`403 issue is locked`), so a reviewer reply needs
  `DELETE …/issues/{n}/lock` → post the comment → `PUT …/issues/{n}/lock -f lock_reason=resolved` again.
  Do the unlock/re-lock in ONE command so the window stays short. Reviewers can still leave reviews on a
  locked PR, so expect to need this.
- **None of the three repos ships a `pull_request_template.md` any more** (checked 2026-09-01:
  `find <repo> -maxdepth 3 -iname '*pull_request_template*'` is empty in all three; platform's
  `.github/` holds only `renovate.json5` and `workflows/`). Keep writing the body to the headings
  below anyway — every merged PR of this routine uses them and reviewers read them — but do not spend
  a call looking for the file. Original note, now stale: ~~all three repos ship
  `.github/pull_request_template.md`~~ — mirror its headings (`# Jira URL`,
  `# Changes` / `## Description` / `## Clarifications`, `# Screenshots & Video`, `# Executed Tests`,
  `# Expected merging strategy`). For a Sonar sweep: "None — this is a `[Misc]` SonarQube cleanup
  commit" under Jira URL, the per-rule table under Description, the *dropped* sites and why under
  Clarifications, the exact `mvn` line + test counts under Executed Tests, and squash / no backport.
- Creating the PR auto-subscribes the session to PR webhooks. XWiki CI is Jenkins and reports later, so
  `get_status` is `pending`/`total_count:0` right after creation — NOT a failure. Webhooks don't deliver
  CI-success / new-push / merge-conflict transitions; for long watches schedule a ~1h self check-in and
  re-arm silently until merged/closed. Reply to reviewers only when genuinely necessary.
- **A reviewer QUESTION on a merged PR is not an objection — answer it, then ship the clarification
  straight to `master`.** "Explain why this is ok / it could need a comment" asks for reasoning, so
  verify the mechanism before replying (here: read the abstract method's own Javadoc and the sibling
  implementation, then settle the JDK behaviour with a 10-line reflection program rather than asserting
  it). If the answer is "the code is right but non-obvious", the deliverable is a comment commit, not a
  revert. Push it to `master` only when explicitly asked to — and still build the module first: a
  comment can trip Checkstyle, and there is no PR gate to catch it.
- **A reviewer's GENERAL PRINCIPLE is not a verdict on YOUR construct — check whether it applies, and
  reply with that check instead of pushing.** #6197 cost two full round trips this way, both reverted to
  what was originally pushed. (1) `@manuelleduc` asked *"Shouldn't this comment apply to the `else if`?"*;
  I moved it to the only placement that binds a comment to a condition (trailing), and Vincent then said
  he dislikes trailing comments and that my original placement was the house style. (2) The same reviewer
  said `??` is preferred over `||` because it does no type conversion — true as JS style — so I switched
  all three sites; he then replied *"don't replace `||` with `??` if it's a behavioral change"*, which for
  `S6644` it always is, since the rule fires on a **truthiness** ternary. Both times the principle was
  right and its application to the flagged construct was wrong, and I was the one holding the information
  needed to see that. So: an ask with correctness content (a bug, a CI failure, a concrete wrong value) →
  push the fix. A general style principle, or a question ("Shouldn't this…?", "Personally I…") → verify it
  against the actual construct, then answer with the finding; change the code only once the answer says to.
  Two reviewers disagreeing in one thread is the signal it was never yours to decide.
- **Reverting is cheap when you can prove the revert lands on already-built code.** After both round trips
  the files went back byte-for-byte to an earlier verified commit — `git diff <builtSha> -- <files>`
  returning zero differing lines is the whole verification, so `node --check` plus that diff is enough and
  no rebuild is needed. State it in the reply; it is also what makes a same-day revert uncontroversial.
- **Replying inside a review THREAD needs the replies endpoint, and the lock dance in one command**:
  `gh api "repos/{o}/{r}/pulls/{n}/comments/{commentId}/replies" -X POST -F body=@reply.md`. Get the
  id from `gh api "repos/…/pulls/{n}/comments?per_page=20" --jq '.[]|"\(.id) \(.user.login) L\(.line)
  \(.path) \(.body[0:80])"'` — cheap, and it does NOT dump patches. Chain
  `DELETE …/issues/{n}/lock` → reply → `PUT …/issues/{n}/lock` in a single call. Your own reply then
  arrives back as a `pull_request_review_comment.created` event authored by `claude[bot]` — skip it.
- **When a batch's UNIT OF JUDGEMENT is coarser than its site count, put the DISTINCT-VALUE COUNT in
  the PR body — not just the shape table.** The `S1186` sweep's first review comment was an LGTM
  carrying a real worry: *"going to be hard to verify every single comment. I propose that we adopt
  them."* The body already described the six class shapes, but it never said the number that actually
  answers the worry — the 172 comments are only **46 distinct sentences** (commons 10, rendering 2,
  platform 34), one per class, so a wrong sentence is wrong for that file and nothing else. Posting
  the per-class table turned "verify 48 comments" into "verify 10 judgements over 11 files". Say it
  up front next time: whenever a batch repeats generated content (a comment, a rename scheme, a
  message), state `N sites / M distinct values` and give the M-row table, because a reviewer who
  cannot bound the check will either rubber-stamp it or stall on it — and rubber-stamping is the
  worse outcome for a batch whose only risk is that a sentence might be untrue. Pair it with the
  sites you did NOT touch: naming the 6 uncommented ones is what keeps an "adopt them all" from
  covering the sites that genuinely need a maintainer.
- **The most likely reviewer objection to a Sonar sweep is INTRA-FILE INCONSISTENCY, not the
  transform.** Sonar under-reports several rules (`javascript:S7762`/`S7768` flagged 4 of 8 sites of
  the same shape in one file), so a batch keyed to the issue list leaves a file speaking two dialects
  and the reviewer points at an *un-flagged* line next to a converted one. Cheap prevention: after
  applying a shape-based batch, grep the pattern across every changed file and convert the rest too —
  they add nothing to the issue count and cost one commit. When a safe/judgement split exists, put the
  un-flagged extras in the MECHANICAL PR and state the splitting principle in both bodies, so the file
  stays coherent on a stated line whichever PR merges. This objection is answered by a push, not an
  argument: verified follow-up commit, then a reply naming every site converted and why the remaining
  ones are the other PR's. This is the ONE reviewer ask that survives the "verify before you push"
  rule above, and only because the check passes trivially — the extra sites are the identical shape
  and provably equivalent, so there is no construct-specific reason the principle could fail on them.
  Run the check anyway and put its result in the reply.
- **Handling a reviewer objection** (verify the mechanism, judge whether the objection is about intent
  clarity, withdraw rather than argue, then ship the `@SuppressWarnings` + rationale version as its own
  PR and reopen the issues) is in the `xwiki-fix-sonarqube-issue` skill. **The dev.xwiki.org JavaCodeStyle page IS reachable** (plain
  `curl` returns HTTP 200); an older note here said it 403s behind Cloudflare, and acting on that
  stale note is what made a sweep miss a documented convention — see the bullet below. **Re-test a
  recorded "source unreachable" before trusting it**, and read
  <https://dev.xwiki.org/xwiki/bin/view/Community/CodeStyle/JavaCodeStyle/> directly when a
  convention question is what the fix turns on; `okf/conventions/` is the cache, not the source.

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
  - **The split axis is "what open question does this raise", and one judgement PR can carry SEVERAL
    axes as separate commits.** A run split 92 issues into 53 mechanical + 39 judgement where the
    judgement half held two unrelated groups: 5 sites whose *value* (not just truthiness) is
    observable, and 34 sites whose transform is provably safe but sits in a **vendored block** — a
    policy question, not a correctness one. One commit per group, each named for its group, plus a
    body section per group saying exactly what is being asked; then the maintainer can drop one
    group by dropping one commit. Cheaper than a third PR and it keeps the mechanical half at zero
    open questions.
  - **Outcome datapoint: the split works, and the judgement half is a coin toss — not a lost cause.** A
    51-issue mechanical PR and a 3-issue judgement PR (`S3400`, `S4165`) opened together from one reactor
    were merged and closed-unmerged respectively, within the same hour, the closure carrying **no comment
    at all**. But a later pair went 32 + 4 and **both merged**: the judgement half (`S3655`, `S4030`)
    collected an "LGTM" plus a second committer's approval and merged two days after the mechanical half.
    The discriminator is what the change costs the reader — trading a documented method for a constant
    (`S3400`) was unwanted, whereas deleting a provably dead local collection was not. So write the
    judgement PR to be merged, not as a formality. A third pair (32 + 3, the JS DOM-API sites) went the
    same way and the judgement half merged **FIRST**, within ten minutes of the review round — what it
    had was a per-site table naming *exactly what could differ* (`removeChild` throws where `remove()`
    is silent; a default parameter only applies to `undefined`) plus an explicit "closing this costs
    nothing". Write that table and the judgement half stops being a coin toss.
    **When the judgement half merges first, the mechanical PR needs `master` merged into it**, since by
    construction the two touch the same files: `git fetch --deepen=250 origin master` (the clones are
    shallow, so without this there is no real merge base), then `git merge --no-commit --no-ff
    origin/master` to see whether it is even contentious — hunks four lines apart auto-merged here.
    Rebuild the MERGED tree before pushing: it is new content neither earlier build saw. Two more consequences. (1) Keep splitting — had those three ridden along, they would have taken 51
    good fixes down with them — and when the judgement half is fine it merges anyway, so the split costs
    nothing. (2) **A silent close IS the review verdict**: record the rule as a
    permanent drop for that repo rather than waiting for an explanation or re-attempting it later.
  - **A closed PR makes your SonarCloud comments WRONG — fix the record the same turn you learn of it.**
    The issues were already *Accepted* with a "Fixed by <PR>" comment, so they now claim a fix that will
    never land. Re-query the keys by `rules=…&issueStatuses=ACCEPTED` (the accept script rewrites its own
    key file to empty on success, so the original list is gone), post a correction comment naming the
    closed PR, then `do_transition transition=reopen` and verify the status went back to `OPEN`.
  - **The split axis can be LANGUAGE, not risk, and it works the same way.** A 25-issue Java PR and a
    23-issue JavaScript PR cut from one reactor (see the `web-war`-in-the-`-pl`-list datapoint above)
    kept the JS half — which gets routed to a front-end reviewer and costs a day of latency — from
    holding up the Java half: the Java PR merged in ~14 h, uncommented, while the JS one was still
    waiting. Same mechanics as the safe/unsure split (disjoint file sets, one build, split by file
    afterwards), and the sibling cross-links in both bodies are what tell the reviewer it is one sweep.
  - **Choose the sibling PR's sites so they land in a module the safe batch ALREADY builds** (oldcore is
    the usual candidate). Preferring FILES the safe batch does not touch keeps the split-by-file step
    trivial — but a **shared file is NOT a drop condition when both branches are YOURS and both are cut
    from the same master**: git merges hunks more than a few lines apart in either order. Three files
    were shared between a 32-issue and a 3-issue JS pair with the nearest hunks ~20 lines apart and
    neither PR blocked the other. Cut the sibling branch straight from master (`git checkout -B
    <branch> <masterSha>`) and apply its edits there, rather than committing on top and splitting.
    The drop rule stands only for a file claimed by **someone else's** open PR.
  - **The two batches do NOT have to be file-disjoint — a REPRODUCIBLE apply script replaces the
    split-by-file step.** When both halves touch the same file in different methods (here oldcore's
    `XWiki.java`: `S6880` in `getHibernateStore`/`onEvent`, `S2142` in `initializeWiki`), splitting a
    combined commit by file is impossible and hunk-staging is fragile. Instead keep each rule in its
    own assert-guarded script keyed to **master** content, apply both to one tree, build once, then
    `git reset --hard <masterSha>` and re-run **one script per branch**. The guard that makes this
    sound is cheap and worth doing every time: save `git diff` before the reset, re-apply *both*
    scripts, and assert the new `git diff` is byte-identical — if it is, each branch's content is a
    subset of the tree the reactor actually verified. It also makes a mid-run drop free (one site was
    reverted after a Checkstyle metric failure: comment out its `edit(...)` call, re-run, done), and
    the per-commit `--stat` sums back to the verified tree's (162+/209- plus 21+/0- = 183+/209-),
    which is a second free check.
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
- **Keep an OKF change MINIMAL — it is shipped config, not a notebook.** Asked to trim
  `xwiki-dev-llm#72`: *"make the changes minimal for the sonarqube OKF parts and in general try to
  avoid duplications as much as possible. Make the text succinct and remove non-essential
  information."* The `S6355` entry had grown to ~55 lines by absorbing the sweep's history (which
  review comment prompted what, per-site examples, counts) and by restating the convention that the
  same PR had just added to `conventions/versioning.md`. Trimmed to 24 lines: only what a *Sonar
  fixer* needs beyond the convention file, with a `[[versioning]]` pointer instead of a paraphrase.
  The division of labour to hold to: **the OKF gets the rule, this repo gets the story** — drop
  counts, per-site tables, "asked for in review", build datapoints and outcome history are memory
  content, and a family file that repeats another OKF file is a review comment waiting to happen.
- **An OKF PR must be branched off a FRESH plugin master, kept small, and pushed the same turn.**
  `xwiki-dev-llm` takes several commits a day and every shipped change bumps the version in four
  manifests, so a branch cut at the start of a long run conflicts on exactly those lines and gets closed
  rather than merged ("Not applying, because code conflicts and it'll be recreated by the routine later
  on if still needed"). Fetch master right before writing the OKF edit, do the version bump last, and
  don't let the branch sit while a build runs. When one is closed for conflicts, do **not** reopen or
  re-push it — condense what it held here and let a later run re-author it on a clean master.
- **An OKF PR that only adds NUANCE to an already-documented rule is not worth opening.** Two
  successive runs had theirs closed with the same words — *"not critical and there's a conflict, if
  it's useful a new PR will be created"* — and the version-bump conflict is structural (parallel
  sessions bump `1.0.N` constantly, so re-deriving the version at push time does not save you; the
  branch conflicts again while it waits for review). Reserve an OKF PR for a rule with **no existing
  entry at all**, or for a correction to an entry that is actively wrong. When the family file already
  has the rule and you only sharpened a shape or two, record it here under *Owed to the OKF* and let a
  later run fold it into a PR it was opening anyway. **A brand-new rule entry is not exempt**: the
  `S117` PR (#49) documented a rule with no OKF entry at all and was closed with the same words. What DOES get merged is a **correction to an entry that is actively wrong**: `#61` moved `S3252` off the denylist (the listed reason belonged to `S1845`) and merged the same day, version bump and all. **Third datapoint, and it settles the rule: `#67` (the `S3415` "default to dropping" verdict) merged
  ~30 minutes after opening, version bump and all** — a close-then-reopen within four seconds is that
  merge flow, not a rejection, so do not react to the `closed` webhook by re-authoring anything. What
  made it land, and worth copying: the body led with the *cost* of the wrong entry (the rule was almost
  skipped) and cited the sweep it unblocked **by its merged PR number**, so the reviewer could see the
  correction had already been validated in production. Open the code PR first, let it merge, then cite
  it. So the discriminator is *does the OKF currently mislead a future run*, not how new the content is — and the strongest thing you can put in such a PR is the sweep it unblocked (123 sites, three PRs, all merged uncommented). Write the condensed *Owed to the OKF*
  entry in the SAME turn you open the PR, never after review.
- **The OKF admits only rules WITHOUT false positives — a conditional rule belongs here, not there.**
  The empty-`catch` `// TODO:` convention was added to `conventions/code-comments.md` the same day a
  review asked for it, and was then removed on request: *"there can be false positives and we need
  only rules that don't yield false positives."* And the FP was one this very sweep had already hit —
  a `catch` of a domain "not found" exception used as a signal must NOT be TODO-ed. So before writing
  an OKF rule, ask what its counter-example is; if you can name one (this routine usually can, from
  the drop list it just built), the rule is a *rule-file* entry here, with the condition attached, and
  the OKF gets nothing. That is a sharper test than "is it durable" or "is it documented on the dev
  wiki", and it explains why the entries that DO merge are corrections and unconditional conventions
  (the multi-line Javadoc rule survived the same review untouched).
- **Owed to the OKF, batch 11** (NOT opened as a PR — this is the THIRD time this content comes up
  and the recorded rule says an *addition* to an existing list gets closed for the version-bump
  conflict, which is exactly what happened to it inside `#53` (batch 2). Fold it into a PR a later run
  opens anyway; if a run ever does open it, open it ALONE and lead with the measured cost.) One
  universal drop condition, for `okf/sonarqube/index.md`:
  - The universal drop-condition list names only the **120-column** Checkstyle rule and calls it
    "consistently the single biggest cause of dropped fixes", which reads as *line length is the
    Checkstyle risk*. It is not: three **metric** rules have now each cost this routine a build round,
    because they run in `checkstyle:check` *after* the tests —
    **`BooleanExpressionComplexity`** (max 3 operators, `S1871`/`S2589` merges),
    **`ExecutableStatementCount`** (max 30/method, `S3358` hoists) and now
    **`CyclomaticComplexity`** (max 10/method, `S6880` `if`-chain → `switch`, where a `case` label
    counts and the `else if` it replaced did not) and **`ClassFanOutComplexity`** (max 20 types per
    class, `S3398`'s move-a-method-into-the-inner-class). The generic statement is what belongs
    there: *any fix that changes a method's control-flow shape, lengthens a condition, or moves code
    between classes is metric-exposed; count the construct in the apply script and drop the site
    when it would cross the cap* — splitting the method or the class is a refactor, not a Sonar
    cleanup, and the rejection is the codebase stating the merged form is not more readable. One
    refinement the `S3398` case adds: when several flagged sites feed the same cap, **rank them by
    cost and apply the cheap ones** rather than dropping the rule — 2 of 3 shipped that way.
- **Owed to the OKF: nothing — `xwiki/xwiki-dev-llm#103` OPEN** (the `S5993` denylist entry deleted,
  the rule moved into `syntax-rules` and added to the rule map, and the `S2386` bullet's "the same
  break as S5993" cross-reference fixed). Eighth "actively-wrong entry" correction, and the first
  where the entry was wrong *because a previous correction of the same entry* (#78) narrowed the claim
  instead of testing it — so a corrected entry is not a verified one. Written to the shape that keeps
  merging: minimal, leading with the cost (163 issues written off), and citing the three sweep PRs
  (#6293/#1945/#427) that were already open when it was filed. Note the version rule inverted on
  2026-09-01 — the PR must NOT touch any version field, and `node scripts/validate.mjs` enforces that.
- **Owed to the OKF: nothing — `xwiki/xwiki-dev-llm#79` MERGED** (plugin 1.1.11 live on master; the `S1130`
  "`src/main` is a permanent drop" line corrected to non-`private`, plus the unreachable-`catch`
  cascade). Seventh "actively-wrong entry" correction, and the first found in a **family file**
  rather than the denylist — the entry contradicted itself two bullets apart (it already argued that
  `private` is proof, for test sources). Written to the shape that keeps merging: one bullet rewritten
  in place, no new family file, no rule-map row, leading with the cost (19 sites written off) and
  citing the sweep PR (#6236) that had just shipped them. Merged the same hour, so the discriminator is
  now seven for seven: **a correction to an actively-wrong entry merges; a nuance addition and a
  brand-new entry get closed for the version-bump conflict.** The three rules with no OKF entry at all
  (`S2177`, `S6880`) or with a conditional transform (`S1130`'s own mechanics) stay in `rules/` here.
- **Owed to the OKF: nothing — `xwiki/xwiki-dev-llm#78` MERGED** (plugin 1.1.10; the `S5993` denylist
  entry narrowed to non-`internal` packages, verified live on master). Sixth "actively-wrong /
  partial-pool entry" correction to land, uncommented and same-day, so the recorded discriminator
  holds without exception now: **a correction merges, a nuance addition and a brand-new entry get
  closed for the version-bump conflict.** What it was written to, and worth copying verbatim: minimal
  (one denylist line rewritten in place — no new family file, no rule-map row), leading with the
  *cost* of the entry as it stood (244 issues never touched), and citing the three sweep PRs it
  unblocked. Those were still **open** when it was filed and were edited to "**merged**" as each
  landed — the same tactic as `#71`, and it again did not weaken the PR, so keep opening the
  correction the same turn rather than waiting on the sweep. The rule itself is conditional (the
  `internal` split), so its transform stays in [rules/java-S5993.md](rules/java-S5993.md) here and
  the OKF got only the two-line escape.
- **Owed to the OKF, batch 10** (NOT opened as a PR — three rules with no OKF entry at all, and the
  recorded rule is that a brand-new entry gets closed for the structural version-bump conflict; the
  same run's `S6213` *correction* went out on its own, minimal, as `xwiki/xwiki-dev-llm#77` — the
  shape that does merge). Fold
  these into a PR a later run is opening anyway; full text in `rules/`:
  - **`java:S108`** belongs next to `S1186` — comment-only remediation (the rule's own *Exceptions*
    section: it ignores a block containing a comment), platform-only, clustered in four oldcore
    files, and the drop condition is truthfulness, with `src/test` the place to suspect.
  - **`java:S9142`** for `constant-and-resource-rules` — hoisting a regex out of a loop is
    javadoc-definitional, with ONE drop shape: `String#split` on a single non-metacharacter compiles
    no `Pattern`, so the rule is a false positive there and the "fix" is a pessimization.
  - **`java:S9355`** for `syntax-rules` — as safe as `S7476`; `/*`→`/**` for a genuine Javadoc,
    delete the `(non-Javadoc)` boilerplate on an `@Override` rather than promoting it.
- **Owed to the OKF: nothing — `xwiki/xwiki-dev-llm#72` MERGED** (plugin 1.1.1). It moved `S6355` off
  the denylist into `syntax-rules`, left `S1123` on it with its own reason, and added the missing
  `@Deprecated` section to `conventions/versioning.md`. **Fifth "actively-wrong entry" correction, and
  the pattern keeps merging** — but note two things about how it got there: it needed a **trim** round
  (*"make the changes minimal … avoid duplications … remove non-essential information"*), so write the
  entry minimal the first time; and it cited its sweep PRs while they were still open, which did not
  weaken it. **A consequence to expect: merging one OKF PR makes every other open one conflict** on
  the four version lines — `#71` (the previous run's `S1172` correction, still open and uncommented)
  went `dirty` the moment `#72` merged. Whichever of two OKF PRs is second needs a master merge and a
  re-derived bump, so land them one at a time and fix the loser immediately (a session may only push
  to its own designated branch, so an older run's branch has to wait for its owner or be re-authored).
- **The citation experiment RESOLVED: `xwiki/xwiki-dev-llm#71` MERGED** (plugin 1.1.7). The `S1172`
  correction — an actively-wrong entry, the fourth of that kind after `S1117`, `S3252` and `S3415`,
  so four for four now. It was opened citing the three sweep PRs it unblocked (platform #6214,
  commons #1918, rendering #408) while they were still **open**, with each merge added to the body as
  it landed. That did not weaken it, so **don't wait for a sweep PR to merge before opening the OKF
  correction** — open it the same turn and update the body as the evidence arrives.
- **Owed to the OKF, batch 9** (NOT opened as a PR — `java:S3457` has no OKF entry at all, and the
  recorded rule is that a brand-new entry gets closed for the structural version-bump conflict just as
  a nuance addition does; fold it into a PR a later run is opening anyway). One rule, `java:S3457`,
  belongs in `constant-and-resource-rules` (or its own row) — full text in
  [rules/java-S3457.md](rules/java-S3457.md):
  - The rule key bundles **five unrelated defects** and the OKF's Sonar corpus has nothing on it, so a
    run either skips the whole 41-issue pool or triages it from scratch. The five shapes and their
    verdicts are decidable from the `message` alone, with no source read.
  - Two are clean: an SLF4J `{}` placeholder used inside a `String.format` (the argument is silently
    dropped — a real defect, `{}` → `%s`, same length) and an unused format argument (usually a caught
    exception that should have been the exception's cause).
  - Three are drops: `%n`-for-`\n` (behaviour change), `Formatter.format("<literal>")` (no cheap
    equivalent — `Formatter` has no `append`), and the `toString()` shape, which is already owned by
    `okf/conventions/logging.md` and needs a cross-reference from the Sonar corpus (the same missing
    pointer that cost the `S2629` withdrawal).
  - Plus the generic build lesson, which belongs in `verification.md`: a fix that changes a log call's
    *pattern/argument split* rather than its rendered output can be contradicted by a test asserting
    `LogEvent.getMessage()`.
  - The JavaScript rules of the same run (`S6582`, `S7765`, `S7721`) were **not** put in the OKF: its
    sonarqube corpus is Java-only, so they live in `sonarqube/rules/` here.
- **Owed to the OKF, batch 8** (opened as `xwiki/xwiki-dev-llm#67`, a *correction* not a nuance addition,
  so it should merge like `#61` did; if it is closed for a conflict, re-author it on fresh master — do
  NOT re-push that branch). One entry, `S3415` in `test-code-rules`:
  - The section opened with **"Usually unsafe — default to dropping it"**, which is a prior rather than
    a condition and is wrong by a factor of five: a full platform sweep needed a drop on about one site
    in six. Rewritten so the two drop shapes it already listed (asymmetric `equals`,
    `assertNotEquals(obj, null)`) are *the drop list*, preceded by the free classifier — **does either
    operand read `null`?** Also narrowed "read the asserted type's `equals`" to the case where the type
    name suggests asymmetry (`Regex*`, a matcher), since doing it per site is what made the rule look
    expensive.
  - Two mechanics added: Sonar flags only *some* of a file's reversed assertions (one oldcore test had
    five times as many as reported), so convert the whole file or it reads two ways; and a swap never
    changes overload resolution even across numeric types (`Math.signum(int)` returns `float`, and both
    argument orders resolve to `assertEquals(double, double)`).
  - The JavaScript rules of the same run (`S7781`, `S6660`, `S6644`) were **not** put in the OKF: its
    sonarqube corpus is Java-only, so they live in `sonarqube/rules/` here.
- **Owed to the OKF, batch 7** (four rules with no OKF entry at all plus one drop taxonomy; fold into
  a PR a later run is opening anyway):
  - **`S116` field naming, sharpening the batch-5 entry** — the drop line is not `src/main`, it is
    **non-`private`**. A `private` field cannot escape its compilation unit, so oldcore `src/main`
    fields (`XWikiContext.engine_context`, `XWikiAttachment.attachment_archive`) rename with the
    compiler as the whole verification; only the one `protected` site was dropped (16/17). Two guards:
    print every occurrence first — they are nearly all `this.X`, and the setter parameter usually
    already carries the target name, so `this.engine_context = engineContext` becomes correct rather
    than self-assigning — and grep the name repo-wide once, because a field name that also appears in
    an `.hbm.xml` or a reflective string is the real hazard.
  - **`S6876`/`S6877`, one entry for `modernization-rules`** — both mean "iterate `List#reversed()`":
    S6876 replaces a backward `ListIterator` walk, S6877 a `Collections.reverse()` on a fresh copy.
    Single drop condition: the body mutates through the iterator (`it.remove()`/`it.set()`). Delete the
    orphaned `java.util.ListIterator` import, and keep the defensive `new ArrayList<>(…)` copy when the
    source is a `Set`. **Expect a performance question in review**: `reversed()` is a lazy view
    (`ReverseOrderListView`), so S6876 is the identical backwards walk plus one wrapper object per
    loop, and S6877 is strictly cheaper — it deletes an O(n) in-place reversal of a copy.
  - **`S3655`, for `simplification-rules`** — fixable only when it is a one-liner: `isPresent()` plus a
    second `get()` on the same expression → `map(pred).orElse(false)`, and
    `reduce((a, b) -> a + sep + b).get()` → `collect(Collectors.joining(sep))` (equivalent for a
    non-empty stream, which `Enum.values()` always is). Drop when presence is established by an earlier
    statement Sonar cannot follow.
  - **`S4030`, for `dead-code-rules`** — clean when every occurrence of the local is its declaration
    plus `add(…)` calls. Remove the enclosing `else` if that `add` was its only statement, or the fix
    trades S4030 for a fresh `S108`.
  - **`S125`, a drop taxonomy for the existing entry** — four shapes, all decidable from the flagged
    block: a `TODO`/`FIXME` above *or below* it (one site was anchored by a TODO two lines later saying
    "see previous commented code"), a sentence explaining why the code is not run, prose Sonar mis-reads
    as code, and a block whose removal would empty an `if`. The clean subset is an unanchored leftover —
    disabled `verify(…)`/`when(…)` pairs in tests, dead alternative implementations. 11 fixed / 17
    dropped on the fresh platform pool.
- **Owed to the OKF, batch 6** (not opened as a PR — one sharpens an existing entry and the rest are new
  rules, and the version-bump conflict is structural; fold into a PR a later run is opening anyway):
  - **`S3824`, sharpening the existing `index.md` denylist note** — the note says "check the guarded
    block", which is right but incomplete. Four drop shapes, all decidable from the flagged block:
    (a) the mapping function can return `null` (`computeIfAbsent` does not store it, so a cached
    negative result is lost); (b) the block feeds a second collection/map as well as the `put`;
    (c) the mapping function calls another component from inside the map operation; (d) the flagged
    `get()` is a context save/restore, not a lazy init. 27 of 35 platform sites converted on that test.
    `XWikiContext` extends `Hashtable`, so the map methods work on it directly (cast the result).
  - **`S2129`/`S1158`, for `simplification-rules`** — `new Boolean(x).toString()` → `Boolean.toString(x)`
    clears BOTH rules on one line; `new String(s)` inside a `cloneResult(String)` override is safe
    because `String` is immutable.
  - **`S2129` — when the deleted wrapper was a DEFENSIVE COPY, the fix is incomplete without a comment.**
    `DefaultLESSCompiler.cloneResult(String)` returned `new String(toClone)`; the abstract contract says
    "returns a clone of the result to avoid returning the instance stored in the cache", and the sibling
    `ColorTheme` implementation really does copy because `ColorTheme` is mutable. Removing the copy is
    correct — `String` has no mutators, so there is nothing for a caller to corrupt, and on a modern JDK
    `new String(s)` does not even copy the characters (it shares the backing array; only the header
    object differs — verified with reflection). But the method is now named `cloneResult` and does not
    clone, which is exactly the shape the next cleanup pass "fixes" back. Add a `//` comment giving the
    immutability reason. Generalises: whenever a removal makes a method's NAME stop describing its body,
    the comment is part of the fix.
  - **`S6395`, for `simplification-rules`, with a pointer from the `S6035` denylist entry** — unwrapping
    `(?:["'])` → `["']` is safe when the regex is a `private static final Pattern`; only a
    `public static final String` regex carries the Revapi `constantValueChanged` break that got `S6035`
    denylisted. The denylist currently reads as if the whole regex family were off-limits.
  - **`S2388`, for `syntax-rules`** — prefixing an inner class's call with `super.` is behaviour-neutral
    *only if* the inner class does not itself declare that method; grep it before batching, because if
    it overrides the method `super.` silently changes the target.
- **Owed to the OKF, batch 5** (not opened as a PR — one entry sharpens an existing rule, the other
  adds a rule, and both would hit the structural version-bump conflict; fold them into a PR a later
  run is opening anyway):
  - **`S1130`, in `dead-code-rules`** — add the cascade: the rule reports only the frontier of a test
    class's private-helper call graph, so fixing the flagged sites re-flags the next ring on the next
    scan. Strip every `throws` in the test file at once. Full text under *Batch mode* above.
  - **`S1117`, a NEW entry for `syntax-rules`** — "rename this local variable which hides the field
    declared at line N". It is **not** the unsafe twin of `S117` the denylist note claims, provided
    the rename is scoped to the block rather than the method: the shadow runs from the declaration to
    the enclosing block's closing brace, so every bare occurrence there is provably the local. Use
    `(?<![\w$.])name(?![\w$])` (the `.` spares `this.name` and `x.name`), assert the new name is unused
    in the range and that the name is not inside a string literal there. Commons: 15/15, 0 drops, 8 of
    them same-type locals; platform then went 107/107 with 0 drops. Correct the `S117` entry's closing
    sentence, which currently sends a reader away from a viable pool. Three additions the platform
    sweep proved necessary, all detailed under *Batch mode* above: the **header** scope case
    (`for`/`catch`/parameter declarations are not brace-scoped from the declaration), **pattern
    variables** are flow-scoped and must be hand-edited, and each invented name must be checked
    against the class's **field set** so the fix does not re-create the rule. `src/main` is not a drop
    condition — a local cannot escape its method — but it is the natural safe/unsure PR split, exactly
    like `S117`'s locals-vs-parameters split.
  - **`S116` field naming, also for `syntax-rules`** — "rename this field X to match `^[a-z][a-zA-Z0-9]*$`".
    Distinct from the denylisted **`S115`** (`static final` constants): S116 fires on *instance* fields
    written in CONSTANT_CASE. Clean when every reader is in the same module (grep the name, then the
    compiler is the whole verification); a drop when the field is `protected`/`public` on a class
    published in a `test-jar`, or anywhere in `src/main`, because that is a cross-module rename.
    Replace the longer name first when one field name contains another as a substring.
- **Owed to the OKF, batch 3** (closed PR `xwiki/xwiki-dev-llm#48`; two shapes that sharpen the
  EXISTING `test-code-rules` S5778 section rather than adding a rule):
  - **S5778 — "hoist the nested invocation" is not "hoist the outermost one."** When the thrower is the
    outer call and its arguments are the fixtures (`new EntityReference(null, new EntityReference(a),
    new EntityReference(b))` — the `null` name is what throws), hoist the **arguments** and leave the
    outer constructor in the lambda. A batch keyed to "the first nested `new X(…)`" takes the whole
    body, compiles, and silently moves the exception outside `assertThrows`. Guard: assert the hoisted
    span is not the entire lambda body.
  - **S5778 — a block-bodied lambda is a fix, not a drop.** `() -> { Foo f = new Foo(); f.doIt(); }` is
    the same defect in braces: hoist the declaration and the block collapses to an expression lambda,
    or to a method reference when only the call is left (`assertThrows(RuntimeException.class,
    message::getType)`). Only statements that cannot throw may leave the block.
- **Owed to the OKF, batch 4** (closed PR `xwiki/xwiki-dev-llm#49`, same "not critical + a conflict"
  reason as batch 2 — re-author on fresh master, do NOT re-push that branch). One rule, `S117`
  "rename this local variable to match `^[a-z][a-zA-Z0-9]*$`", belongs in `syntax-rules`:
  - It fires on **method parameters as well as locals** even though the message always says "local
    variable", and *what is declared* is the natural safe/unsure split — a local cannot escape its
    method (compiler + module tests are the whole verification), while a parameter of a public legacy
    API changes no signature and nothing in Revapi but is visible in Javadoc/IDE completion, so it
    belongs in its own PR. Update the `@param` tag in the same edit. **Outcome datapoint: both PRs were
    merged, the parameter one without an objection** — so the split is cheap insurance, not a sign the
    parameter half is unwelcome; keep splitting (it lets the mechanical half merge first) but do not
    drop parameter sites for fear of the review.
  - **The `this.<name>` trap**: an oldcore parameter usually shadows a field of the same name, so the
    substitution must never rewrite a `this.X` access (`(?<![\w$])(?<!this\.)name(?![\w$])`); the field
    keeps its own name and `this.engine_context = engineContext;` is correct.
  - Mirror-image guard: **assert the NEW name occurs zero times in the rename scope**, or the rename can
    capture a bare reference that used to resolve to a same-named field — which compiles.
  - Renaming a local **file-wide** is safe when the identifier is only ever declared as a local;
    transliterate a non-ASCII name (`prefβ` → `prefBeta`) rather than re-lettering it; watch the
    120-column rule on the *uses*, not the declaration.
  - Add a denylist note that the neighbouring **`S1117`** (hides-a-field rename) is the unsafe one — a
    missed occurrence there silently re-binds to the field instead of failing to compile.
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
- **A version-bump conflict on an OKF PR is not always fatal — one was REBASED for me rather than
  closed.** The `S1172` correction (#71) was pushed at `1.0.33`; by the next day master was at
  `1.1.6` (a minor bump plus five patches in ~24 h) and the PR had been rebased onto it with the bump
  re-derived to `1.1.7`, content intact — and it **merged** minutes later. So the recorded expectation
  ("it gets closed for the conflict") holds for a *nuance addition*; a correction to an actively-wrong
  entry gets fixed up and landed instead. Two consequences: keep such a PR small enough that a rebase is trivial, and
  after any wake-up on an OKF PR **check the head SHA before touching the branch** — reset the local
  branch to the remote rather than pushing your stale commit over someone's rebase.
- **AN OKF PR MUST NOT TOUCH THE PLUGIN VERSION AT ALL — the rule inverted on 2026-09-01 and every
  older note here about "re-deriving the bump" is obsolete.** `scripts/validate.mjs` on master now
  *fails* a PR that changes any version field, compared against the branch's **merge base**: "a pull
  request must not — it makes every concurrent PR conflict … the release is cut on master by
  `scripts/release.mjs`". To force a non-patch release, put a `Release-Bump: minor` (or `major`)
  trailer in a commit message instead of editing a manifest. This removes the structural conflict that
  had been the single most common reason these PRs were closed unreviewed — so an OKF PR is now
  cheaper to open than every note below assumes.
  **Two consequences for a wake-up on an older OKF PR.** (a) The reviewer may push a "drop the plugin
  version bump" commit onto your branch; reset to the remote and keep it. (b) CI then goes red for a
  confusing reason — the check runs the copy of `validate.mjs` **on the branch**, which still enforces
  the old "bump above master's tip" rule, so it reports `1.1.8 is not greater than 1.5.1`. The fix is
  a `master` MERGE, not a version edit: it brings the new validator, the manifests take master's
  version unchanged, and the merge base advances to master's tip so the version check sees no change.
  Run `node scripts/validate.mjs` *after* committing the merge — before the commit the merge base is
  still the old one and it reports a phantom bump.
- **What the OKF already covers moves fast.** Between one run's PR and the next, parallel sessions
  documented eight more rules and the plugin went 1.1.x → 1.5.x in a week. Re-read the rule map and the
  family file on master before writing anything and drop whatever landed meanwhile: a PR that
  re-documents an existing rule is noise. (The version half of this warning no longer applies — see
  the bullet above.)
- **Recording learnings (memory repo → `main`).** The xwiki-platform fix lives on a feature branch but
  learnings go to this repo's `main`. Do NOT edit on the feature branch then stash/checkout/pop (main
  has diverged; the pop bakes `<<<<<<<` markers into the commit). Instead `git checkout main && git pull
  origin main` FIRST, then edit and commit directly on main. Route the learning per the WRITE protocol
  at the top — durable rule-correctness facts go to the OKF as a PR, not here.
