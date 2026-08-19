# `javascript:S7781` — "Prefer `String#replaceAll()` over `String#replace()`"

## Why the flagged set is small and clean

The rule fires only where the regex is a **single literal character** (plus `/g`), so the fix is
`replaceAll("<char>", r)` with a **string** pattern, not `replaceAll(/<char>/g, r)`. That is why a
transliteration table of ~35 `replace(/[…]/g, …)` lines yields only the handful whose class holds one
character — the multi-character classes are not flagged at all. Read the `message`: a second shape
says *"This pattern can be replaced with '\\\\'"*, which is the same instruction.

Safe because none of the flagged characters is a regex metacharacter. Prove it anyway with a `node`
loop over: the empty string, one occurrence, a repeat, adjacent occurrences, and a haystack
containing `.*+?[]()|\` — the switch from regex to literal string is exactly what that last case
tests.

## It PAIRS with `javascript:S6397` — one edit, two issues

`temp.replace(/[Ĳ]/g,"IJ")` carries **both** an S7781 (regex → `replaceAll`) and an S6397
("replace this character class by the character itself") on the same line. Going straight to
`temp.replaceAll("Ĳ","IJ")` clears both, because the regex is gone entirely. Seven such lines in
`xwiki.js` gave **14 issues from 7 edits** — the best issues-per-edit ratio of that run. Intersect the
`(file, line)` sets of the two rules before planning the batch.

## Drops

- **`js/xwiki/panelwizard/ieemu.js` (4 keys) is third-party** (IE Emu, © Erik Arvidsson / WebFX) —
  permanent drop, like the other vendored WAR scripts.
- Anything in a file an open agent PR already touches.

## No baseline concern

`replaceAll` is ES2021 but is **already used** in these same hand-written WAR scripts
(`js/xwiki/table/livetable.js`, `js/xwiki/editors/dataeditors.js`), so it introduces no new browser
requirement. Say so in the PR body rather than leaving a reviewer to wonder. Keep the file's own
spacing style (`,"IJ"` with no space after the comma, in `xwiki.js`).
