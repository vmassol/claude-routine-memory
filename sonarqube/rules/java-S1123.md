# `java:S1123` — "deprecated elements should have both the annotation and the Javadoc tag"

OKF-denylisted, and [rules/java-S6355.md](java-S6355.md) records it as *analyzed, not attempted*.
Both halves of the rule have since been shipped: the pool splits cleanly by `message`, and each
half splits again on a free classifier — the annotation half on *is the version derivable*, the
tag half on *is the site an `@Override`*.

## Triage by `message` — one query, no source reads

* **"Add the missing `@deprecated` Javadoc tag"** — **CORRECTION: not a permanent drop.** The
  recorded reason ("the tag has to say *why* and *what to use instead*") is true only of the sites
  that declare their own API. **Split the pool on `@Override`**: an override of an
  already-deprecated parent member inherits the answer, because the parent interface/superclass
  usually documents it with its own `@deprecated` tag — so the fix *copies* the text instead of
  inventing it. On the 2026-09 pool that was 102 of 190 sites `@Override`, 76 of them with a
  signature-matching parent tag. See *The `@Override` half* below. A bare
  `@deprecated since X` still clears the rule while leaving worse documentation than it found —
  that shape is what the original verdict was really about, and it stays banned.
  **Then split the REMAINDER on "does the member's own BODY name the replacement" — that is a
  second, independent lever and it is worth 33 more sites.** See *The self-declaring half* below.
* **"Add the missing `@Deprecated` annotation"** (platform 16, commons 3, rendering 0) — mechanical
  *if* the version is derivable, and a **judgement PR**, never the mechanical batch: the annotation
  makes every existing call site emit a deprecation warning and changes what tooling reports about a
  published API.

## Three checks decide each annotation-shape site

1. **Is the deprecation version in the `@deprecated` tag?** Reuse the `S6355` classifier
   ([java-S6355.md](java-S6355.md)) verbatim — same regex, same multi-version comma rule, same
   "strip the version out of the tag rather than duplicating it". **A bare `@Deprecated` is not an
   option**: it clears `S1123` and raises a fresh `java:S6355` on the next scan, which is trading one
   issue for another. 4 of 16 platform sites failed this
   (`LegacyOfficeImporterScriptService`, whose tags read `use {@link #officeToXHTML(…)} instead`).
2. **Does a comment above the declaration already answer the rule?** In XWiki this is not
   hypothetical — **7 of the 16 platform sites carry a commented-out annotation**:
   ```java
   // TODO: uncomment the annotation when XWiki Standard scripts are fully migrated to the new API
   // @Deprecated(since = "17.0.0RC1")
   public class ScriptXWikiServletRequest extends WrappingXWikiRequest
   ```
   That is the universal "a comment explains why the code is the way it is" drop condition in its
   most literal possible form, and it is the single biggest bucket of the pool
   (`Script*`/`Wrapping*`/`*Stub` servlet wrappers in `oldcore`). Grep the two lines above the
   flagged line for `@Deprecated` before anything else.
3. **Is the element a copy of somebody else's API?** commons'
   `jakarta.servlet.http.HttpSessionContext` says `@deprecated deleted in Servlet 6` — a *servlet
   spec* fact, not an XWiki version, and the `jakartabridge` modules deliberately mirror the spec.

Net on the 2026-09 pool: 16 platform → 5 shippable, 3 commons → 2, rendering → 0.

**Outcome: BOTH judgement PRs MERGED, uncommented** — commons #1946 (2 sites) and platform #6304
(5 sites) landed the same day, with no question raised about the deprecation-warning cost the rule
is denylisted for, and none about the multi-version `since = "10.2,9.11.4"` form. So
the annotation shape is welcome where the version is derivable. Keep shipping it as its own PR
anyway — the split is what let it merge on its own schedule.

## Revapi does not object

`revapi:check` under `-Plegacy,quality` passed on `oldcore`, `legacy-oldcore`, `refactoring-api` and
`rest-server` with `@Deprecated` **added** to a `public` class, a `public` constructor and a `public`
static method, and on commons `extension-api` with it added to two `public` interface fields. So the
break to argue about is not `java.annotation.added` — it is only the deprecation warnings the
annotation now emits at call sites, i.e. a product decision. `java-S6355.md` had verified the
narrower `java.annotation.attributeAdded`; this extends it to the annotation itself.

## The `@Override` half — the tag is a COPY, not prose

The whole batch is one sentence a reviewer can check: *every site is an `@Override` of a member
whose parent already carries an `@deprecated` tag, and the tag text is that parent's text.* Deriving
it needs no snippet reads beyond the flagged declaration:

1. Index every `(methodName, paramCount) → @deprecated tag` in **all three repos** (one walk over
   `*.java`, regex a Javadoc block followed by optional annotations and a declaration). Do it across
   repos: an oldcore override's parent interface often lives in commons or rendering.
2. Per site, rebuild the declaration from the flagged line **forward** until the closing `)` — XWiki
   wraps long signatures, so the flagged line alone gives the wrong param count.
3. Keep a candidate only if its class simple name occurs in the site file (import / `extends` /
   `implements`) — that filter is what stops a same-named method in an unrelated class matching.
4. One distinct tag ⇒ apply. Several ⇒ pick by hand. None ⇒ **drop**, the text would be invented.

Three normalisations, all needed:

* **Strip the version from the copied tag** (`since 4.0M1 use {@link #getRoleType()} instead` →
  `use {@link #getRoleType()} instead`); the version belongs in `@Deprecated(since = …)` per the
  Java Code Style, and duplicating it is exactly what the `S6355` sweep was asked to undo.
* **Qualify a `{@link Type#…}` the subclass does not import** (same-package and imported types are
  fine, everything else needs the FQN) — the parent's tag was written against the *parent's*
  imports.
* **A parent tag can be WRONG; do not copy a typo.** `GroupFilter#endGroup`'s own tag points at
  `beginGroupContainer`; the override on `endGroup` must say `endGroupContainer`. Read each tag
  against the member it is going onto — the copy is mechanical, the *check* is not.

Emit a Javadoc holding only the block tag; no `{@inheritDoc}` (Javadoc inherits the main
description automatically when the subclass comment has none):

```java
    /**
     * @deprecated use {@link #getExtensionFeatures()} instead
     */
    @Deprecated
    @Override
```

### Why the minimal comment is safe and sufficient

* **It clears the rule** — the in-repo-precedent check settles the `S8786` hazard: commons+platform
  already hold **202** tag-only Javadoc comments above a `@Deprecated` member, and a 40-site sample
  carries **zero** open `S1123`. (`api/rules/show` cannot answer this; the precedent grep can.)
* **Checkstyle is satisfied by construction** — `JavadocMethod`'s `allowedAnnotations` defaults to
  `Override`, so an override with a parameter list needs no `@param` tags, and XWiki enables none of
  `SummaryJavadoc` / `JavadocParagraph` / `RequireEmptyLineBeforeBlockTagGroup` /
  `NonEmptyAtclauseDescription`. **A non-`@Override` public method with no Javadoc is a different
  matter** — adding one there turns `JavadocMethod` on and it then demands `@param`/`@return`, so
  either write the full comment or drop the site. (It should not arise: `MissingJavadocMethod`
  means such a method already has Javadoc unless its file is Checkstyle-excluded.)
* The diff is insert-only, so nothing can be a behaviour change and no line the PR writes can
  carry a pre-existing finding.

### Drop shapes seen on the 102-site override pool (26 drops)

* **A pure delegate to a third-party API** — `QueryImplementorDelegate` (18) forwards Hibernate's own
  deprecated `Query` methods under the same name. The deprecation is Hibernate's, its Javadoc is not
  in these repos, and there is no XWiki replacement to name. Biggest single bucket; recognise it from
  the body being `return this.delegate.<sameName>(…)`.
* **The parent declares the member deprecated but documents nothing** (`ServletContainerInitializer`
  javax overloads, `DelegateComponentManager#getComponentDescriptorList`).
* **The parent tag is empty** (`DefaultWikiTemplateManager#applyTemplate`).

## The self-declaring half — read the BODY, not the Javadoc

The 88-site residue the `@Override` sweep left was recorded here as *deferred, not dropped* because
"the tag must state why and what instead, and only the API author knows". That is true of the site's
*documentation* and false of its *implementation*: a deprecated member almost always still works, and
the way it still works is by delegating to its replacement. **So the classifier is the first line of
the body, not the comment above it.** 33 of the 101 residue sites shipped on that basis (platform
[#6334](https://github.com/xwiki/xwiki-platform/pull/6334) 29, commons
[#1958](https://github.com/xwiki/xwiki-commons/pull/1958) 4). **Outcome: commons #1958 MERGED
uncommented within hours** — the lever's first verdict, and it matches every denylist rescue before
it, so "the tag is copied, not invented" is a principle a reviewer accepts without argument. Three
convertible shapes:

* **A jakarta overload of a flagged `javax` method, in the same type** — the body is
  `return getSourceURL(JakartaServletBridge.toJakarta(servletRequest));`, so the replacement is the
  same-named jakarta overload. `HttpServletUtils`, `BrowserTab`.
* **A non-deprecated counterpart in the same class** — the deprecated constructor delegates
  (`this(propertyName, new DocumentReference(wiki, space, page))` ⇒
  `{@link #ClassPropertyReference(String, DocumentReference)}`), the deprecated constant simply *is*
  it (`ELEMENT_FILES_FILES = ELEMENT_FILES_FILE`), or the existing Javadoc already points at it
  (`See {@link #CFGPROP_STATS_EXCLUDEDUSERSANDGROUPS_REQUEST}.` ⇒ the same link as the tag). Biggest
  group: 24 sites.
* **An in-file note** — `IncludeMacroParameters#setContext` carries
  `// Marked deprecated since there's now a Display macro instead.` directly above the annotation.

Two free bonuses worth looking for: an **in-file precedent for the wording** (`BrowserTab:87` already
says `@deprecated use {@link #navigate(URL, Cookie[], boolean, int)} instead`, so the new tags read
like the old ones), and the fact that the whole diff is **insert-only** — which is the sentence that
answers the `Quality / Analyze` risk in the PR body.

### Drop shapes of the residue (68 of 101, keyed in `dropped-issues.md`)

* **A delegate to a THIRD-PARTY deprecated API** — `QueryImplementorDelegate` (16), Hibernate's own
  `Query` methods. The body names a replacement that is not XWiki's to recommend.
* **A wholesale-deprecated legacy class stating nothing** — the ~21 `xwiki-platform-legacy-oldcore`
  types (`XWikiCache*`, `com.xpn.xwiki.notify.*`, `XWikiCriteria`/`XWikiQuery`/`OrderClause`,
  `i18n`). You will *think* you know the replacement (`org.xwiki.cache.Cache`, the observation
  system, `QueryManager`) and the file will not say so. Truthfulness, not safety, is the bar.
* **A `javax` overload with NO same-named jakarta counterpart** — and telling this apart from the
  convertible jakarta shape above is the whole check. `ServletContainerInitializer` (4) +
  `DefaultServletContainerInitializer` (3): the only jakarta method on that interface is
  `initializeRequest(HttpServletRequest, HttpServletResponse)`, which folds `initializeResponse` and
  `initializeSession` into one call, so naming it as *the* replacement for each is a claim about
  intent. Same for `XWikiAction#initializeXWikiContext` and the oldcore `XWikiRequest` helpers
  (`Util#getObject`, `Utils#getRedirect`/`getPage`/`prepareContext`, `XWikiForm`): grep the file for
  a same-named overload *before* assuming the jakarta bridge gives you the answer — 8 of the 21
  `javax`-shaped sites failed that grep.

### Checkstyle, when the tag needs a whole new Javadoc block

Adding a comment where there was none turns checks on, so decide per file before editing:

* `xwiki-platform-legacy-*` sets `<xwiki.checkstyle.skip>true</xwiki.checkstyle.skip>` in
  `xwiki-platform-legacy/pom.xml` — nothing is enforced there at all.
* Otherwise the **module pom's `maven-checkstyle-plugin` `<excludes>` list** decides; grep it for the
  file's basename (oldcore's list is long and holds `AttachmentDiff`, `XWikiServletRequest`,
  `XWikiServletResponse` but *not* `StatsUtil`).
* In a non-excluded file, `JavadocMethod` (`accessModifiers=public`) then demands `@param`/`@return`
  and `JavadocType` (`versionFormat=\$Id.*\$`) demands `@version` — so **write the complete comment**
  rather than a tag-only one, and do not create a class-level Javadoc from scratch (every `$Id` in
  the three repos is an expanded `$Id: <hash> $`; a fresh bare `$Id$` would be the only one).

## Outcome (tag half)

Shipped as one mechanical PR per repo — platform #6327 (53), commons #1955 (21), rendering #430 (2)
— verified by a three-repo reactor: rendering 517 tests, platform 9 modules / 1526 tests (oldcore
1209, legacy-oldcore 48), commons 5 modules / 442 tests, all green with `revapi:check` and
`checkstyle:check`. No safe/unsure split was needed: every site satisfies the one principle the PR
body states, so there was nothing left to put in a judgement PR.
