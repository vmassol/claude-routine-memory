# Cross-linter twins — `java:S107` `java:S3776` `java:S112` `java:S1319` (and the whole scan)

**XWiki runs Checkstyle *inside the build*, so an existing `@SuppressWarnings("checkstyle:X")` is a
decision the build enforced, not an opinion.** When a Sonar rule is the same finding under a
different name, the Sonar key is the only thing missing from that annotation — the least arguable
change this routine can make, and it needs no new prose at all.

This is the `S1214` lever (recorded in [java-S1214.md](java-S1214.md)) turned into a **project-wide
scan** rather than a per-rule hunch.

## The scan (one pass, no source reads beyond the annotation block)

For every open, unclaimed, not-already-dropped issue: take the annotations attached to the flagged
declaration and look for `checkstyle:(\w+)`. Print `(repo, rule, twin)` histogram. Cost: one file
read per flagged file, no API calls. On a swept day it returned 30 rows across all three repos.

**Implementation gotcha that cost two passes:** walking up over "lines that look like annotations"
breaks on a multi-line `@SuppressWarnings({…})`, silently dropping 9 of 17 hits and making the pool
look tiny. Collect the **non-comment, non-blank lines above the flagged line**, find the last
`@SuppressWarnings`, **balance its parentheses forward**, and require that everything between its
end and the declaration is another annotation.

## The twin map (verified against `xwiki-commons-tool-verification-resources/…/checkstyle.xml`)

| Sonar rule | Checkstyle check | Same finding? |
|---|---|---|
| `java:S107` too many parameters | `ParameterNumber` | **Yes — both use max 7** (XWiki keeps the Checkstyle defaults) |
| `java:S3776` cognitive complexity | `CyclomaticComplexity`, `NPathComplexity`, `JavaNCSS`, `ExecutableStatementCount`, `MethodLength`, `NestedIfDepth`, `BooleanExpressionComplexity` | **Yes in substance** — different formula, same "this method is accepted as complex" decision |
| `java:S112` generic exception | `IllegalThrows` | Yes for `throws Throwable`/`Error`/`RuntimeException`; **NO for `throws Exception`** (`IllegalThrows` does not flag it) |
| `java:S1319` declare with an interface type | `IllegalType` | Yes |
| `java:S1141` nested try | `NestedTryDepth` | Yes |
| `java:S1192` duplicated literal | `MultipleStringLiterals` | Yes |
| `java:S1113` `finalize()` | `NoFinalizer` | Yes |
| `java:S1142` too many returns | `ReturnCount` (max 5) | Yes |
| `java:S138` method length | `MethodLength` | Yes |
| `java:S1135`/`S1134` | `TodoComment` | Yes |

**The drop condition is that the twin must be the SAME finding.** `XARMojo#performTransformations`
carries a *complexity* suppression and has an open `java:S112` on its `throws Exception` — no
recorded decision about the thrown type, so that one stays open. Checking the twin, not merely the
presence of a `checkstyle:` key, is the whole gate.

## Per-rule notes

* **`java:S107`** — because both thresholds are 7, every S107 site in a Checkstyle-*enforced* file
  must already carry the suppression; the ones that do not are in files the module pom excludes from
  Checkstyle (oldcore legacy, rest-server). That splits the rule for free into a **mechanical half**
  (already suppressed → merge the key) and a **judgement half** (published signatures in
  Checkstyle-excluded files → new annotation + a per-class reason, own PR).
* **`java:S3776`** — OKF-denylisted as "a genuine refactor, never a mechanical fix", which is true of
  the rule and false of the subset whose method the project already exempts from its own complexity
  checks. Same escape axis as `S2386`: the entry objects to the *remediation*, not to the finding.
  Several sites even carry the finished sentence (`// SAX requires fairly complex code`,
  `// XWikiDomSerializer copied from DomSerializer`).
* **`java:S1319`** — the clean shape is a type **dictated by an upstream API**:
  `JakartaServletBridge#toJavax/#toJakarta` must use `EnumSet`, because both Servlet APIs declare
  `addMappingForUrlPatterns(EnumSet<DispatcherType>, …)`. The other shape (a `public static` method
  returning `Hashtable` since XWiki 1.0) is the published-signature argument, i.e. the judgement PR.

## Mechanics

* **Insert-only / one-string edits**; `@SuppressWarnings` has `SOURCE` retention, so the bytecode is
  byte-for-byte identical and neither Revapi nor JaCoCo can be affected. Say so in the body.
* **Repeating the same key twice in one file is safe**: Checkstyle's `MultipleStringLiterals`
  defaults to `ignoreOccurrenceContext=ANNOTATION`, which is also why files that already carry
  `"checkstyle:CyclomaticComplexity"` on two methods build green.
* **Re-wrap the annotation** rather than appending to an over-long line — assert ≤120 on every line
  the script emits, with the continuation at indent + 4.
* **Do NOT add a comment to a key-merge site.** The decision and (where present) its rationale are
  already in the code; inventing a new sentence for each is the `S1214` failure mode (a borrowed
  sentence justifies the file it was written in and nothing else). Reserve the prose for the sites
  that get a *new* annotation.

## Outcome

47 issues in four PRs, 2026-09-16 — commons [#1983](https://github.com/xwiki/xwiki-commons/pull/1983)
22, rendering [#441](https://github.com/xwiki/xwiki-rendering/pull/441) 2, platform
[#6407](https://github.com/xwiki/xwiki-platform/pull/6407) 7 (mechanical) and
[#6408](https://github.com/xwiki/xwiki-platform/pull/6408) 16 (judgement) — on a day the
never-mentioned-rule diff was 0 over five severity × twelve language facets and 28 open agent PRs
held 248 files.
