# `java:S4144` — "methods should not have identical implementations"

Pool: platform 11, rendering 6, commons 2 — one of the very few rules with sites in all three repos
on a day the mechanical allowlist is dry. First swept 2026-09: rendering **5 fixed** (#428), commons
**0 fixed / 2 reported**.

## Triage it as a BUG DETECTOR first, a duplication cleanup second

Half the pool is not duplication worth factoring out — it is a copy-paste slip that the rule happens
to notice. Three of the eight non-platform sites were real defects:

* `WikiEntityUtil#getHtmlSymbol(char)` (rendering) returns `entity.fWikiSymbol`, while its own
  `String` overload returns `fHtmlSymbol` and `getWikiSymbol(char)` returns `fWikiSymbol`. The
  `char` overload is wrong.
* `VelocityParser#getSpaces(...)` (commons) is documented *"Match a group of space characters
  (ASCII 32)"* and its body is byte-identical to `getWhiteSpaces` above it, i.e. it matches
  `Character.isWhitespace`. Either the Javadoc or the body is wrong; both are `public`.
* `OutputTargetConverterTest#convertFromInputStream()` (commons) is a copy of `convertFromFile()` —
  it builds a `File` and asserts a `FileOutputTarget`. It does not test what its name says.

**The classifier is the pair's NAMES and Javadoc, and it costs two method bodies to read**: when the
two names describe the *same* operation (`endDefinitionDescription` / `endDefinitionTerm`,
`printBeginItalic` / `printEndItalic`) the duplication is deliberate and extracting is right; when
they describe *different* operations (`getHtmlSymbol` / `getWikiSymbol`, `convertFromFile` /
`convertFromInputStream`, `getSpaces` / `getWhiteSpaces`) the identical body IS the defect and the
fix is a behaviour change that needs an owner. Report those in the PR body; do not clear them.

## The safe subset: extract one private helper per group

Give the shared body a name that states what it does and have every entry point call it. No
signature, visibility or behaviour changes, and the compiler verifies it. Sonar reports one issue
per *extra* copy, so 8 methods in 3 groups cleared 5 issues.

Two mechanics:

* **Assert the shared body's occurrence count before rewriting** (3 / 2 / 3 here). It is the whole
  guard against a fourth copy being left behind — and it also proves you are not converting a
  neighbour that merely looks similar.
* **Order the edits: replace the bodies FIRST, insert the helper second.** The helper's body is
  byte-identical to the string you are replacing, so an insert-first script rewrites the helper too.

## What a reviewer is likely to ask

That these are distinct listener/printer entry points and the duplication is fine. The answer is in
the PR body up front: every public entry point stays exactly where it was, only the shared body is
named, and the extraction is Sonar's own recommended remediation.
