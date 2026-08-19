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
- **Accept all issues in a loop** (per issue: `add_comment` + `do_transition accept`). Each issue is
  ~2 curls ≈ 4s, so 20+ issues blow a 2-min timeout — run the accept loop as a BACKGROUND task, and/or
  make it idempotent (re-query which keys are still OPEN). **Budget the wall clock from a measured
  per-key rate, not from a past run's total** — the proxy's throughput varies by an order of magnitude
  between runs (168 keys in ~30 min once, but only 50 keys in ~50 min on another, i.e. ~60 s/key). At
  the slow rate the loop outlasts the whole write-up, so launch it FIRST and check progress with one
  `issueStatuses=ACCEPTED` count query rather than tailing its log. **An unthrottled loop silently loses about
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
  **Expect a JS batch to be routed to a front-end reviewer even when Vincent LGTMs it** ("this is
  javascript and I'd like an opinion from someone with more expertise, cc @manuelleduc") — so write the
  body for a reviewer who did none of the analysis: state the equivalence proof per rule and list the
  sites you deliberately did NOT convert. An LGTM on a JS batch is not a merge signal; do not treat the
  PR as finished. **Outcome datapoint: it works** — a 43-issue `S7773`+`S6353` PR (#6186) got Vincent's
  LGTM, then `@manuelleduc`'s formal approval ~7h later with no change requested, and merged. So the
  provable JS subset (`parseInt`/`parseFloat`, the `Number.isNaN`-on-a-Number sites, regex character
  classes) clears specialist review as-is; the routing costs a day of latency, not a rework.
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
- **After a COMPILE error, rebuild only the failed module plus the ones the reactor skipped behind
  it.** The modules that already reported `Tests run:` in the failed run are green for sources you
  have not touched since, so re-verifying them is pure wall clock (a 3-module re-run replaced an
  11-module one). Confirm the claim first with `git diff` between the two applications of the batch —
  if the second application changed only the failing file, the earlier greens still hold.
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
- **A reviewer QUESTION on a merged PR is not an objection — answer it, then ship the clarification
  straight to `master`.** "Explain why this is ok / it could need a comment" asks for reasoning, so
  verify the mechanism before replying (here: read the abstract method's own Javadoc and the sibling
  implementation, then settle the JDK behaviour with a 10-line reflection program rather than asserting
  it). If the answer is "the code is right but non-obvious", the deliverable is a comment commit, not a
  revert. Push it to `master` only when explicitly asked to — and still build the module first: a
  comment can trip Checkstyle, and there is no PR gate to catch it.
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
  - **Outcome datapoint: the split works, and the judgement half is a coin toss — not a lost cause.** A
    51-issue mechanical PR and a 3-issue judgement PR (`S3400`, `S4165`) opened together from one reactor
    were merged and closed-unmerged respectively, within the same hour, the closure carrying **no comment
    at all**. But a later pair went 32 + 4 and **both merged**: the judgement half (`S3655`, `S4030`)
    collected an "LGTM" plus a second committer's approval and merged two days after the mechanical half.
    The discriminator is what the change costs the reader — trading a documented method for a constant
    (`S3400`) was unwanted, whereas deleting a provably dead local collection was not. So write the
    judgement PR to be merged, not as a formality. Two consequences. (1) Keep splitting — had those three ridden along, they would have taken 51
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
- **An OKF PR that only adds NUANCE to an already-documented rule is not worth opening.** Two
  successive runs had theirs closed with the same words — *"not critical and there's a conflict, if
  it's useful a new PR will be created"* — and the version-bump conflict is structural (parallel
  sessions bump `1.0.N` constantly, so re-deriving the version at push time does not save you; the
  branch conflicts again while it waits for review). Reserve an OKF PR for a rule with **no existing
  entry at all**, or for a correction to an entry that is actively wrong. When the family file already
  has the rule and you only sharpened a shape or two, record it here under *Owed to the OKF* and let a
  later run fold it into a PR it was opening anyway. **A brand-new rule entry is not exempt**: the
  `S117` PR (#49) documented a rule with no OKF entry at all and was closed with the same words. What DOES get merged is a **correction to an entry that is actively wrong**: `#61` moved `S3252` off the denylist (the listed reason belonged to `S1845`) and merged the same day, version bump and all. So the discriminator is *does the OKF currently mislead a future run*, not how new the content is — and the strongest thing you can put in such a PR is the sweep it unblocked (123 sites, three PRs, all merged uncommented). Write the condensed *Owed to the OKF*
  entry in the SAME turn you open the PR, never after review.
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
