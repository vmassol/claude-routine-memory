# Utility-class / dead-code family — S1118 / S1144 / S1185

> Load only when fixing one of these. Cross-cutting mechanics live in `learnings.md` → *General
> batch-fix techniques*. Fresh pivot when S6201/S6204 are saturated.

When the small mechanical pools AND S6201/S6204/S1066 are all simultaneously drained or PR-saturated
(the common concurrent-session state), this trio is a reliable unclaimed pivot — deep MAJOR pools,
cleanup waves rarely touch them, they regenerate, and they're near-always zero open PRs. **oldcore is
the densest source and is FAIR GAME** for them even when an open oldcore PR exists, as long as that PR
is a *different* rule (the recurring open oldcore PR is S6201-only). All three are additive/removal, so
they bundle into one multi-type PR over an oldcore-dominant reactor (one build). Verify each per below —
delegate the per-site reading to parallel general-purpose subagents over DISJOINT files, then cross-check
`git diff --name-only` against the expected set and grep each removed name is gone (General techniques).

- **`S1118`** add a `private` constructor to a utility class. Two variants: "Add a private constructor
  to hide the implicit public one" (class has NO ctor → insert one after fields/before first method,
  respecting DeclarationOrder; a short Javadoc is safe, private ctors don't strictly need it) and
  "Hide this public constructor" (class HAS a `public` ctor → change `public`→`private`). **The body
  MUST contain a comment** (`{ // Utility class }`, matching the `XarUtils`/`TikaUtils` idiom) — an
  empty body just trades S1118 for a fresh `S1186` (empty method) and a reviewer WILL flag it.
  **DROP the "hide" variant when the ctor is actually instantiated** (`grep "new Foo("` across the
  reactor) — e.g. factory classes exposing instance methods, `new XxxFactory()` called from a service.
  **`FinalClass` follow-on (build-breaker, not optional):** a class whose only ctor is now `private`
  trips checkstyle `FinalClass` ("Class X should be declared as final") → `-Pquality` FAILS. Always add
  `final` to the class in the SAME edit, after confirming it has no subclasses (`grep "extends Foo" == 0`);
  applies to inner holder classes too (`static class Holder` → `static final class Holder`, the
  init-on-demand idiom). Do this pre-emptively for every "add" fix.
  **NOT near-0 drops on leaf modules** — expect heavy drops: (a) **abstract base classes** that hold only
  static members but are DESIGNED FOR EXTENSION (`public abstract class AbstractX` with subclasses) — a
  private ctor breaks the subclasses' `super()`, and any ctor on a *public* abstract class risks revapi →
  DROP (it's a false positive; the class is already non-instantiable). (b) **public-API holder classes in
  a NON-`internal` package** (e.g. an enum's `public static class Constants`) — adding a private ctor
  REMOVES the implicit-public ctor → revapi `java.method.visibilityReduced` → DROP unless you add the
  ignore. PREFER S1118 sites in `internal` packages / package-private inner holders (revapi ignores
  `internal`, so those are the clean ones). The purely-additive "add private ctor + final" on an
  internal/inner utility class is the safe subset.
  **Revapi gotcha:** reducing a previously-public (or implicit-public) ctor to `private` on an API class
  is a breaking change → `-Pquality` fails with `java.method.visibilityReduced`, needing a revapi ignore.
  Legacy modules RE-EXPORT oldcore's public classes, so the SAME change trips the `*-legacy-*` module's
  revapi too — add the ignore in BOTH that module and oldcore. An S1118 PR that skips this leaves the
  affected module (esp. `xwiki-platform-legacy-oldcore`) failing revapi on a clean build for everyone —
  a still-present master-wide breakage: when your batch touches `legacy-oldcore` for an UNRELATED rule and
  its revapi fails on `<init>()` of classes you never edited (XWikiConstant/Utils/TOCGenerator/i18n),
  that's this pre-existing debt — drop your `legacy-oldcore` file and ship the rest (per learnings.md).
- **`S1185`** remove an override whose body is ONLY `super.x(sameArgs)` (optionally `return`ed). Delete
  the whole method incl. Javadoc + `@Override`. DROP if it does anything else, changes
  return/throws/visibility meaningfully, or adds a behaviour-bearing annotation. Removing the sole
  method of a class ORPHANS its imports → clean them (word-boundary rule). **CRITICAL false-positive —
  reflective declared-method dispatch:** a pure super-call override IS needed (not redundant) when a
  framework registers behaviour by scanning the concrete class's `getDeclaredMethods()` (inherited
  methods excluded). In XWiki this is **`XWikiPluginManager.initPlugin()`**: it maps a plugin to a
  function name only if that method is DECLARED on the plugin class, so any `com.xpn.xwiki.plugin.*`
  class (extends `XWikiDefaultPlugin` / implements `XWikiPluginInterface`, incl. all `skinx` plugins)
  must redeclare `endParsing`/`virtualInit`/… even to just call super, or the callback stops firing.
  **DROP every S1185 hit in a plugin class.** More generally: **an explicit "we must override…"/"do not
  remove" Javadoc on the method is a HARD STOP — never delete a method whose own comment explains why it
  exists**; treat that comment as authoritative and drop the issue (Sonar can't see runtime dispatch,
  and the build/tests won't catch a lost reflective callback). Non-plugin classes with no such dispatch
  (plain POJOs, component impls called via their interface) are safe to remove.
- **`S1144`** remove an unused `private` method (+ orphaned fields/imports). GOTCHAS: (1) **Hibernate/JPA
  reflective accessors** — a `getX`/`setX` on a persistent entity (e.g. `XWikiDocument`) may be mapped by
  property name in `*.hbm.xml`; grep the `.hbm.xml` mappings before removing any getter/setter, DROP if
  mapped. (2) skip serialization hooks (`writeObject`/`readObject`/`readResolve`/`writeReplace`). (3)
  **Removal CASCADES** — deleting the method can orphan a private helper it was the sole caller of (e.g.
  removing `localizePlainOrKey` orphaned `getLocalization()` + its field); trace and delete the whole
  dead chain, else you leave a fresh S1144. Process multiple methods in the SAME file highest-line-first.
  (4) **Check the SIBLING/twin class** — a legacy `Deprecated*` class often mirrors a non-deprecated
  twin (in a different, often non-legacy module outside your batch's module set) carrying the SAME dead
  method; re-query the rule project-wide by the method name so both get fixed in one PR, not a review
  round-trip.
