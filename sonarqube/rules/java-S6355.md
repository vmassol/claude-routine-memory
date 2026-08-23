# `java:S6355` — "Add 'since' and/or 'forRemoval' arguments to the @Deprecated annotation"

**OKF-denylisted (with `S1123`) as "needs the deprecating version; see [[versioning]]". That reason is
true only of the sites where the version is genuinely unknown — and in XWiki it usually is not: the
same element's `@deprecated` Javadoc tag states it, one line above the annotation.** Fifth rule
rescued from a bad denylist entry after `S1117`, `S3252`, `S3415` and `S1172`, and the largest pool
any run has found: **768 open (platform 630, commons 119, rendering 19), of which 464 are derivable.**

```java
    /**
     * @deprecated since 8.3RC1, use {@link #AbstractCache(CacheConfiguration)} instead
     */
-   @Deprecated
+   @Deprecated(since = "8.3RC1")
```

Nothing is invented: the edit moves a documented fact into the form Java 9's `@Deprecated` contract
and the IDEs can read. `@Deprecated(since = "…")` is already the house convention (125 platform files,
15 commons files carry it).

## The free classifier

The whole triage is one pass over the Javadoc block above the flagged line — **no snippet reads, no
whole-file reads, and it is scriptable end to end**:

1. From the flagged line walk up over `@…`/blank lines to the Javadoc `*/`, then back to its `/**`.
2. Cut the `@deprecated` tag's text at the next block tag (`\n\s*\*\s*@\w`) — otherwise a later
   `@since` (which means *when the element appeared*, not when it was deprecated) is captured instead.
3. Take the version from `(?:since|starting with|as of|from)\s+(\d+\.\d+(?:\.\d+)?(?:(?:RC|M|BETA|DEV)\d+)?)`.
   Requiring a `since`-style keyword is what keeps a version-looking token in a `{@link}` or in prose
   out. Bad-looking version strings in the wild (`5.ORC1` with a letter O, `4.4MA`) fail the regex on
   their own, which is the correct outcome — do not "repair" them.

Buckets measured over all 768: **439 single-version (mechanical)**, **25 multi-version (judgement)**,
**304 drops** — 123 with no Javadoc at all, 21 with a Javadoc but no `@deprecated` tag, 160 with a
tag that names no version (`@deprecated use {@link X} instead`). Those 304 are the sites the denylist
entry is really about: leave them open, never guess a number.

A `since` keyword away from the head of the tag is still the deprecation version, not a false
positive — `replaced by {@link #exists(DocumentReference)} since 2.2.1`, `not used anymore since 6.0`,
`does not do anything since 11.5RC1`. 24 of 439 read that way and all were right; classify on the
regex, not on the position.

## Multi-version tags are the only judgement call

`@deprecated since 14.10.2, 15.0RC1 …` (also `7.4.5 and 8.2RC1`, `14.10.2 / 15.0RC1`) — the API was
deprecated on a stable branch *and* on master, and `since` takes one string. 25 sites (platform 21,
commons 1, rendering 3). Shipped as the sibling judgement PR using the **last (highest)** version,
with the question stated: highest (the line this code lives on) or earliest (it *has been* deprecated
since then)?

## Why it is safe

* **Zero line-drift risk, and this is worth re-verifying rather than assuming**: Sonar flags the
  annotation line itself — `lines[issue.line - 1].strip() == "@Deprecated"` held for **768 of 768**
  issues. So the apply script's assertion *is* the site location; no pattern matching, no
  `textRange` arithmetic.
* **Revapi does not report it.** `java.annotation.attributeAdded` on `@Deprecated` passed
  `revapi:check` in every module of a 77-module three-repo reactor, public API and `oldcore` included
  (there is no ignore entry for it in `xwiki-commons-tool-verification-resources/…/revapi.json`, so
  it is genuinely not classified as a break). Probe it in one module first anyway — `mvn package
  revapi:check -Plegacy,quality -DskipTests -pl <a module with a public-API site>`, 2:16 — before
  applying hundreds of edits.
* No behaviour, no signature, no line-length risk (the annotation line is short), and Checkstyle has
  nothing to say about it.
* **0 drops in 464 applied sites**, three repos, 4690 tests green — and the accept pass confirms the
  arithmetic exactly: platform 370 ACCEPTED / 260 still OPEN, commons 81 / 38, rendering 13 / 6, i.e.
  the OPEN remainder equals the recorded non-derivable count in every repo.

## The sibling rule `java:S1123` — analyzed, NOT attempted

"Deprecated elements should have both the annotation and the Javadoc tag" (platform 171, commons 35,
rendering 3). One rule key, two opposite shapes, and neither is free:

* *"Add the missing `@deprecated` Javadoc tag"* — writing the tag means writing **why and what to use
  instead**, i.e. prose only the API's author can supply. A bare `@deprecated since X` clears the rule
  and leaves worse documentation than it found.
* *"Add the missing `@Deprecated` annotation"* — mechanical (the version is in the tag), but it makes
  every existing call site emit a deprecation warning and changes what tools report about a published
  API. That is a decision, not a cleanup.

## Where the pool sits

Thin-spread with one very dense file per repo. Platform: `oldcore` 199 of 349 mechanical sites (then
`bridge` 20, `rest-server` 12, `extension-script` 11, ~1-9 each over 43 more modules); commons:
`extension-api` 18, `legacy-component-api` 12, `job-api` 9, then singletons over 19 more modules;
rendering: `xwiki-rendering-api` 5, `legacy-api` 4, `transformation-macro` 1. It regenerates from
every new deprecation that omits `since`.
