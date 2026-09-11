# The false-positive suppression pool (`@SuppressWarnings("<rule key>")`)

Not one rule — a **pool that cuts across rules**, and the one the routine had never used. When every
mechanical rule is a recorded drop and the workable files are claimed by open PRs, what is left is
the set of issues that are *wrong about XWiki*, and for those the OKF's prescribed resolution is a
real fix: `@SuppressWarnings("java:SXXXX")` plus a `//` comment stating why, **in the code**, not an
*Accepted* transition in SonarCloud. SonarCloud closes the issue at the next analysis because the
rule stops raising it.

## Why it is safe, and why a reviewer accepts it

* **Zero behaviour risk by construction** — the diff is an annotation and a comment.
* **There is in-repo precedent, and one grep proves it** (put the numbers in the PR body):
  `grep -rho '@SuppressWarnings([^)]*:S[0-9]*[^)]*)' <repo> --include=*.java | sort | uniq -c`
  → platform 36 files (`java:S5785` ×43, `S2629`, `S1144`, `S1149`, `S4973`, `S2077`, `S3077`,
  `javabugs:S2190`, `javasecurity:S6096`, `javasecurity:S5145`), commons 13 files, rendering 6.
  So the idiom is established for `java:`, `javabugs:` **and** `javasecurity:` keys alike.
* **`javabugs:` suppressions are honoured by the dataflow engine** (separately verified on
  `javabugs:S2259`), so a dataflow false positive is resolvable in code.

## The multiplier: one annotation can clear many keys

SonarCloud reports a dataflow finding **once per call path** and a parameter finding **once per
parameter**, all on the same declaration. So the issue count of a suppression site is often far
above 1, and the *unit of judgement* is one sentence:

* `EntityReference#setName` — **5** `javabugs:S6416` on the same `throw`, one annotation.
* `MethodArgumentUberspectorTest.ExtendingClass` — **6** `java:S1172`, one class-level annotation.
* A `finalize()` override — **2** issues per file (`java:S1113` on the override, `java:S5738` on
  `super.finalize()`), plus a second `S5738` on the declaration line, i.e. 3 per file.

Count by KEY, and query `issues/search?...&rules=<key>&ps=500` before writing the PR body: a
three-annotation diff cleared 11 issues here.

## The drop condition is truthfulness, exactly as for the comment-only rules

Suppress only what you can *argue* in the comment. Three shapes were refused in the run that opened
this file, and refusing them is what makes the rest credible:

* **A `javasecurity:` finding** (`S6096` zip-slip-shaped path construction, `S5145` log injection) —
  do not wave a security finding away on a sweep's budget; it needs the sanitisation argument.
* **A concurrency/lifecycle finding** (`java:S2696` lazy-init on a static field) — it needs a
  synchronisation decision, not a sentence.
* **A finding whose "false positive" claim is really "it is caught downstream"**
  (`javabugs:S6466` `chunks[0]` after `split(":")`, inside a `catch (Exception)`) — that is a real
  edge case, merely handled.

## Verified FP shapes found so far (all three repos)

| Rule | Shape | Why it is wrong here |
|---|---|---|
| `java:S1150` "implement Iterator rather than Enumeration" | the javax↔jakarta `Enumeration` bridges (`JakartaToJavaxEnumSet`/`…Enumeration`, `JavaxToJakartaEnumeration`) and `ResourceLoader.ResourceEnumeration` | adapting/returning an `Enumeration` **is** the class's contract — for the bridges by definition, for the loader because `ClassLoader#getResources` imposes it |
| `java:S1172` "unused method parameter" | a test fixture class of deliberately conflicting overloads (`conflictingMethod(String,String)` / `(Integer,Integer)` / `(String,String,String...)`) | the parameters are the thing under test — they exist to give the resolver candidates |
| `java:S3011` "this accessibility update should be removed" | a test-framework helper calling `setAccessible(true)` and restoring the previous value in `finally` | it is how `@BeforeComponent`/`@AfterComponent` methods in package-private test classes are invoked |
| `javabugs:S6416` "fix this IllegalArgumentException" | a validating setter whose Javadoc already carries `@exception IllegalArgumentException …` | the throw is the documented contract; the Javadoc tag *is* the argument, so write the comment from it |
| `javabugs:S6322` "`remove` will throw `UnsupportedOperationException`" | `getChildren().remove(i)` guarded by an `indexOf(...) == -1` check | the accessor returns the immutable `Collections.emptyList()` **only** when the collection is null/empty, and the guard has already excluded that |
| `java:S1113` + `java:S5738` | a `finalize()` kept as a last-resort resource-release safety net | one of the two sites already carried `@SuppressWarnings("checkstyle:NoFinalizer")`, i.e. the decision had been taken and only half recorded — look for that half-record, it is the strongest evidence |

**The half-recorded decision is the best signal there is.** A site already carrying a
`checkstyle:` suppression, or a Javadoc `@exception`/`@throws` tag for the very exception the rule
calls a bug, is a maintainer who already answered the question. Grep for those before triaging by
hand.

## Interaction with the `Analyze` gate

A suppression is normally an **insertion** above the declaration, so it rewrites no existing line and
cannot inherit a pre-existing finding — unlike the rules whose fix rewrites the declaration itself
(`S1130`, `S1172`, the renames). That makes the pool unusually safe against the "your own lines carry
it" check. The exception is a rule whose fix *edits* the declaration (removing `implements Cloneable`
for `java:S2157`): run the pre-check on the replaced lines, not the written ones — see
`learnings.md`.
