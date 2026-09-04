# `java:S1123` — "deprecated elements should have both the annotation and the Javadoc tag"

OKF-denylisted, and [rules/java-S6355.md](java-S6355.md) records it as *analyzed, not attempted*.
That is still right for the rule as a whole, but the pool splits cleanly by `message` and the
smaller half is worth a judgement PR.

## Triage by `message` — one query, no source reads

* **"Add the missing `@deprecated` Javadoc tag"** (platform 155, commons 32, rendering 3) — a
  permanent drop. The tag has to say *why* and *what to use instead*; a bare `@deprecated since X`
  clears the rule and leaves worse documentation than it found.
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
the annotation shape is welcome where the version is derivable; it is the *tag* shape (writing the
prose) that stays a permanent drop. Keep shipping it as its own PR anyway — the split is what let it
merge on its own schedule.

## Revapi does not object

`revapi:check` under `-Plegacy,quality` passed on `oldcore`, `legacy-oldcore`, `refactoring-api` and
`rest-server` with `@Deprecated` **added** to a `public` class, a `public` constructor and a `public`
static method, and on commons `extension-api` with it added to two `public` interface fields. So the
break to argue about is not `java.annotation.added` — it is only the deprecation warnings the
annotation now emits at call sites, i.e. a product decision. `java-S6355.md` had verified the
narrower `java.annotation.attributeAdded`; this extends it to the annotation itself.
