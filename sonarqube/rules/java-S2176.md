# `java:S2176` + `java:S9149` — deliberate name shadowing

Two rules, one pool, one argument: **`S2176`** ("Class names should not shadow interfaces or
superclasses") and **`S9149`** ("Static methods should not hide methods from superclasses"). They
fire on the same idiom seen at type level and at member level, so triage and ship them together.

**Both were recorded here as WHOLE-RULE drops and both are pools.** The recorded reason — *"the
rename is an API break and loses the intent"* — is true, and it is an objection to the **remediation
the message names**, not to the finding. Same escape axis as `S2386` (the eleventh denylist rescue):
ask whether the entry rejects the rule or one way of satisfying it. Here the shadowing is
deliberate, so the resolution is the false-positive one — `@SuppressWarnings("java:SXXXX")` plus a
`//` comment saying why, in the code (see [fp-suppressions.md](fp-suppressions.md)).

**Yield: 86 issues over 40 classes in all three repos on a day the never-mentioned-rule diff was
empty and every rule with a pool was a recorded drop** — platform #6376 (19), commons #1974 (63),
rendering #436 (4). It is one of the very few pools that exists in all three repos at once.

## Why it is cheap

* **The diff is insert-only** — three lines above a type declaration, nothing rewritten. So it
  cannot inherit a pre-existing finding on a line it wrote, and `Quality / Analyze` cannot fire.
* **The unit of judgement is the CLASS, and the classes collapse into four sentences** (below), so
  86 issues cost ~4 judgements plus a bucketing pass. `StringTool` alone is 42 issues, one
  annotation.
* **The classifier needs one line**: the type declaration itself (`X extends <pkg>.X`) already shows
  the shadowing, and the class Javadoc immediately above usually states the reason. No dataflow, no
  call-site reads.

## The four shapes, and the sentence each takes

1. **A deprecated class kept only to preserve the old name.** It extends the class that replaced it
   and carries `@deprecated use {@link <the same simple name in another package>}`. Renaming removes
   the backward compatibility it exists for. The `@deprecated` tag *is* the argument — write the
   comment from it. (11 of the 86.)
2. **A drop-in extension of a third-party type** (Commons Lang, dom4j, XStream, Velocity tools,
   SLF4J, SAX, Groovy). The name is shared so that moving to it is an import change, or — for the
   Velocity tools and XStream converters — so it can be registered in the library's place. Almost
   always stated in the class Javadoc ("Extends {@link X} with…", "Add support for Y to X"). (12.)
3. **The same role in a sibling package, paired by name**, told apart by package and component hint
   (`nestedspaces` vs `nestedpages`, `url.internal.standard` vs `url.internal.container`, an
   `internal` interface extending its public role, syntax `xwiki21` extending `xwiki20`). **This is
   the only shape the classes do not state themselves** — it is a reading of the package layout, so
   say so in the PR body and offer to drop those sites. (9.)
4. **A static that deliberately hides the inherited one** to change its defaults, selected
   explicitly through the subclass (`UsersClass.getListFromString`, `XWikiCoreContainer.createAndLoad`,
   `XStreamFileLoggerTail.exist`, `XWikiWikiParameters.newWikiParameters`), plus **test fixtures whose
   hiding is the thing under test** (`DefaultVelocityManagerTest.SubTestClass`). (54, of which
   `StringTool` is 42.)

## Applying it

* Anchor the insert on the **type declaration line** (`class`/`interface`), not on the flagged
  member: a class-level `@SuppressWarnings` covers every member, which is what makes `StringTool`'s
  42 one edit. For a nested class (`SubTestClass`) anchor on the nested declaration.
* Put the `//` comment **above** the annotation, which goes last, directly above the declaration and
  below any existing `@Component`/`@Named`/`@Deprecated`. That is the established in-repo style.
* A class flagged by both rules takes `@SuppressWarnings({"java:S2176", "java:S9149"})` and one
  comment (`LocaleUtils`).
* Verify the hidden member really exists before writing "deliberately hides": `S9149` requires an
  identical signature, so grep the superclass for it (`ListClass.getListFromString(String)` parses
  `key=value` items — the subclasses re-declare it to split a plain comma list, which *is* the
  reason).

## The drop condition is truthfulness

Nothing here was suppressed to clear a count. Shape 3 is the one to state as an open question rather
than assert; and a shadowing you cannot explain from the declaration, the Javadoc or the package
layout stays open.
