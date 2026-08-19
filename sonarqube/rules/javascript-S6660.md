# `javascript:S6660` — "'If' statement should not be the only statement in 'else' block"

Purely structural (`} else { if (c) { … } }` → `} else if (c) { … }`), so there is no behaviour
question at all. The only decision is **how big the resulting diff is**.

## The drop condition is diff size, not correctness

Merging dedents the whole `else` body by one level. When that body is a short `if` (2-4 lines) the
diff is 3 lines and the change is obviously good. When it is a long block — a 30-line `if/else if`
chain, or an `if` whose branches span dozens of lines — every line of it moves, producing a diff far
larger than the fix for what is a cosmetic rule. **Drop those** and say so under *Clarifications*;
splitting the difference (merging and re-indenting 40 lines) is churn a reviewer has to read.

Two of four platform sites were dropped on exactly this basis (`suggest.js` lines 902 and 987, both
long chains); the two short ones shipped.

## A comment between `else {` and the `if` moves INSIDE the merged branch

```js
} else {
  // In mono-source mode, reset the list if present
  if (this.resultContainer.down("ul")) {
```
becomes
```js
} else if (this.resultContainer.down("ul")) {
  // In mono-source mode, reset the list if present
```
The comment cannot stay above the `} else if` line (it would land inside the *previous* branch), so it
goes to the first line of the new body. Check that the comment still reads correctly there — if it
describes the `else` case as a whole rather than the guarded action, prefer dropping the site.

## Mechanics

Feed the whole block verbatim into an `assert count == 1` edit table — brace surgery is exactly the
case the skill's structural-edit advice is for. Verify with `node --check` afterwards; a mismatched
brace is the only way to get this wrong and it fails loudly.
