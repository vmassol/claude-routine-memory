# `java:S2177` — "Rename this method; there is a `private` method in the parent class with the same name"

No OKF entry. The XWiki pool (platform 12, none in the siblings) is almost entirely **`private`
methods**, which makes the rename compiler-verified: every reference is in the same compilation unit.
Ten of the twelve shipped in one pass.

## Mechanics

- Rename with `(?<![\w$.])name(?![\w$])` over the whole file, asserting the occurrence count matches
  what the site survey predicted **and** that the new name occurs zero times in the file.
- **Grep `--include=*.aj` for the name first.** An AspectJ inter-type declaration in a `*-legacy-*`
  module is compiled as a member of its target class and can therefore call a `private` method that
  `javac` never sees from the main module. (None of the 12 names matched, but this is how a merged
  `S1172` batch once left `master` red.)
- **An overload is not a rename target.** `R1130040XWIKI16682DataMigration` has a private
  `setStore(List, Session)` *and* calls the inherited `setStore(Session, List, String)`; a
  word-boundary rename would rewrite both. Use exact-string edits for the declaration and its one
  call instead.

## Drop condition: the method's counterpart cannot be renamed

The rule is about readability, so a rename that leaves a class *less* readable is a drop even though
it compiles. The shape to watch for is a **verb pair split across visibilities**:
`MyPersistentLoginManager` has a `private decryptText` (flagged) next to a `public encryptText` (also
flagged, but a public API that cannot be renamed). Renaming half the pair is worse than leaving both.

## Why it is a judgement PR — and the outcome says it barely is

Nothing can break, but the rule buys clarity rather than correctness, so it was split out from the
mechanical batch on the assumption that "churn" was the likely objection. **It merged uncommented,
within minutes of the mechanical half** (`xwiki/xwiki-platform#6237`), so a private-method rename is
ordinary accepted work here, not a gamble — keep splitting (it costs nothing and lets the mechanical
half land first) but do not shrink the batch out of caution.

What to lead the PR body with: the one site whose confusion is *concrete* (the `setStore` overload
above), then the safety argument. Everywhere else the parent's method is `private` and therefore never
inherited at all, so the collision is purely cosmetic — say that plainly rather than overselling it.
