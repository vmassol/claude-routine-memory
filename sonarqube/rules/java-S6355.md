# `java:S6355` — "Add 'since' and/or 'forRemoval' arguments to the @Deprecated annotation"

**OKF-denylisted (with `S1123`) as "needs the deprecating version; see [[versioning]]". That reason is
true only of the sites where the version is genuinely unknown — and in XWiki it usually is not: the
same element's `@deprecated` Javadoc tag states it, one line above the annotation.** Fifth rule
rescued from a bad denylist entry after `S1117`, `S3252`, `S3415` and `S1172`, and the largest pool
any run has found: **768 open (platform 630, commons 119, rendering 19), of which 464 are derivable.**

```java
    /**
-    * @deprecated since 8.3RC1, use {@link #AbstractCache(CacheConfiguration)} instead
+    * @deprecated use {@link #AbstractCache(CacheConfiguration)} instead
     */
-   @Deprecated
+   @Deprecated(since = "8.3RC1")
```

**Both halves are the fix.** Vincent asked for the Javadoc strip on review of the first sweep: the
version is *moved*, not duplicated. Do it in the same run (as a second commit, so the reviewer can see
the annotation change on its own) rather than leaving the tag restating what the annotation now says.

Nothing is invented: the edit moves a documented fact into the form Java 9's `@Deprecated` contract
and the IDEs can read. `@Deprecated(since = "…")` is already the house convention (125 platform files,
15 commons files carry it).

## The free classifier

The whole triage is one pass over the Javadoc block above the flagged line — **no snippet reads, no
whole-file reads, and it is scriptable end to end**:

1. From the flagged line walk up over `@…`/blank lines to the Javadoc `*/`, then back to its `/**`.
2. Cut the `@deprecated` tag's text at the next block tag (`\n\s*\*\s*@\w`) — otherwise a later
   `@since` (which means *when the element appeared*, not when it was deprecated) is captured instead.
3. Take the version from
   `(?:since|starting with|as of|from)\s+(\d+\.\d+(?:\.\d+)?(?:(?:RC|M|BETA|DEV)\d+)?)(?![\w.])`.
   Requiring a `since`-style keyword is what keeps a version-looking token in a `{@link}` or in prose
   out. **The trailing `(?![\w.])` is not optional and shipping without it cost a review round:** the
   pool contains MALFORMED versions (`4.4MA`, `5.2M`, `14.0CR1`, `5.ORC1`), and without the boundary
   the regex silently matches their numeric *prefix* — the annotation gets `since = "4.4"` and the
   strip pass then leaves the orphaned `MA` sitting in the Javadoc tag
   (`@deprecated MA use {@link #getRoleType()} instead`). Compiles, passes every gate, and is the
   first thing a reviewer sees. With the boundary these sites simply drop out, which is the right
   outcome — do not "repair" a malformed version silently; if a reviewer rules on the intent
   (`4.4MA` → `4.4M1`) apply that, and read the analogous ones the same way (`14.0CR1` → `14.0RC1`).

   **Audit for it after the fact in one pass, no source reads**: for each site, look at the character
   following the captured version *in the original tag text* — an alphanumeric or a `.<digit>` means
   the capture was a truncation. 3 of 464 sites, and it found all three including the two the reviewer
   had not seen yet.

Buckets measured over all 768: **439 single-version (mechanical)**, **25 multi-version (judgement)**,
**304 drops** — 123 with no Javadoc at all, 21 with a Javadoc but no `@deprecated` tag, 160 with a
tag that names no version (`@deprecated use {@link X} instead`). Those 304 are the sites the denylist
entry is really about: leave them open, never guess a number.

A `since` keyword away from the head of the tag is still the deprecation version, not a false
positive — `replaced by {@link #exists(DocumentReference)} since 2.2.1`, `not used anymore since 6.0`,
`does not do anything since 11.5RC1`. 24 of 439 read that way and all were right; classify on the
regex, not on the position.

## Stripping the version out of the `@deprecated` tag

Three shapes, all on the single line that holds the phrase (it is never split across lines in 464
sites), and all scriptable off the same `since`-keyword regex used to find the version:

* **Leading** — `since 8.3RC1, use {@link X} instead` → `use {@link X} instead`. Capitalise the
  remainder when the tag started with a capital (`Since 10.11 the MoveRequest …` → `The MoveRequest …`).
* **Embedded / trailing** — `replaced by {@link #exists(…)} since 2.2.1` → `replaced by
  {@link #exists(…)}`; `not used since 10.1RC1 because …` → `not used because …`.
* **The whole tag text was the version** — 28 of 464 sites (`@deprecated since 12.5RC1`), which become
  a bare `@deprecated` marker. Nothing is lost (the annotation carries it) and nothing objects: XWiki
  does **not** enable Checkstyle's `NonEmptyAtclauseDescription`, and `S1123` fires on a *missing* tag,
  not an empty one. Say so in the PR body and offer to write a sentence per site.

Three gotchas, all found by scanning the rewritten lines rather than by the build:

* **A phrase alone on a *continuation* line leaves a dangling ` * ` line** that must be deleted
  (`@deprecated replaced by {@link A} and {@link B}` / `since 2.2.1` — 2 sites in
  `DocumentAccessBridge`). Nothing in the build complains; it just looks wrong in the diff.
* **One surviving `since <number>` was correct** — `Velocity supports the same functionality natively
  since 1.6` is a *third-party* version. Re-scan every rewritten line for a leftover version and read
  the hits rather than assuming the regex was wrong.
* Also assert, per rewritten line: no leading punctuation, no double space, ≤120 columns. That scan
  (439 + 25 lines) found exactly the four real problems above and nothing else.
* **A removed phrase can leave prose that only made sense WITH it.** `already deprecated since 3.1M2,
  use {@link X}` strips to `already deprecated, use {@link X}` — grammatical, redundant, and the
  reviewer's second comment. Scan the remainder for a description that restates the deprecation
  (`^(already |now )?deprecated\b`) and delete that clause too; 2 of 464
  (`DefaultLogger`, `DocumentAccessBridge#getDocument`). The other remainder shapes the same scan
  flags — a leading `this`/`This` (`@deprecated this is now part of the official Velocity library,
  use …`) — read fine and must be left alone; ~20 of those.

Cost: one extra reactor. Second run over the same 77 modules, warm `~/.m2`: commons **3:53**/1177
tests, rendering **0:52**/524, platform **15:48**/3037 — **all green, ~21 min**, i.e. roughly half the
cold-ish first pass.

## Multi-version tags — NOT a judgement call, list them all

`@deprecated since 14.10.2, 15.0RC1 …` (also `7.4.5 and 8.2RC1`, `14.10.2 / 15.0RC1`, and 3-version
forms) — the API was deprecated on a stable branch *and* on master. `since` is a single `String`, which
is why a first sweep treated "which version?" as an open question and shipped the highest one in a
judgement PR. **That was wrong and review caught it**: the
[Java Code Style](https://dev.xwiki.org/xwiki/bin/view/Community/CodeStyle/JavaCodeStyle/#HDeprecation)
says *"If the deprecation is done in several branches, the since parameter should use a comma-separated
list of all versions in which the deprecation has been done"* — `@Deprecated(since = "15.5RC1,14.10.12")`.
So emit **every** version, comma-separated, no space, **in the order the Javadoc tag listed them** —
25 sites (platform 21, commons 1, rendering 3). Do not sort: the page prescribes no order, and sorting
newest-first (copying the format of its example) drew a second review comment — *"the best practice
page doesn't mention any order so please keep the order defined in the javadoc"*. Copying the source
order also needs no version comparator at all, which is the cheaper implementation as well as the
right one.

The same page settles the other two questions this rule raises: **never `forRemoval`** (XWiki does not
break APIs), and the `@deprecated` tag must state **WHY and WHAT INSTEAD** — which the 28 bare-marker
sites below do not, so expect to owe a follow-up there.

**Read that page (it is reachable — a stale note claiming Cloudflare 403 is what made a whole sweep
miss this) before treating anything about deprecation as a judgement call.**

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
* **Outcome: ALL 464 issues merged** — platform #6216 (349) + #6217 (21), commons #1920 (81) + #1921
  (1), rendering #409 (10) + #410 (3), plus the `legacy-oldcore` build fix (#6218) and the OKF
  correction (`xwiki-dev-llm#72`). The mechanical halves landed the same day; the three multi-version
  halves landed once the convention question was answered from the code-style page (platform's two
  days later). **So a "judgement" PR whose open question turns out to have a documented answer merges
  like any other** — the split still paid (it kept 440 clean fixes moving while the question was
  settled), but the question itself should have been a documentation lookup, not a PR.
* **The commons half merged the same day**, after two review comments —
  and both were about **text my strip pass mangled** (the truncated `4.4MA`, the dangling "already
  deprecated"), not about the rule, the version derivation, the 28 bare tags or the multi-version
  split. So the classification is not where this rule's risk lives; the prose transformation is.
  Budget the review scrutiny accordingly: scan every rewritten line before pushing, and expect the
  annotation half to pass without comment.
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
