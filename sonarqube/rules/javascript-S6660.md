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
goes on its own line at the top of the merged branch's body. **That is the settled house style** — it
is what the rest of the code base does, and both reviewers converged on it on #6197 ("the original
version was good enough for me … something we do in lots of other places"; "leading comment is fine as
long as the line length is within the rules").

Two things NOT to do, both tried and rejected on that PR:

- **A trailing comment on the `else if` line.** It is the only placement that syntactically binds the
  comment to the condition, which is why it looks like the answer when a reviewer asks whether the
  comment should apply to the `else if` — but trailing comments on a condition line are not wanted here.
- **Dropping the site to preserve the original commented `else`.** Also rejected: the `else if` merge
  is worth having, and the comment placement is not a reason to keep the nested shape.

So the earlier advice in this file — "prefer dropping the site if the comment describes the `else` case
as a whole" — was wrong; keep the merge and leave the comment in the body.

## Mechanics

Feed the whole block verbatim into an `assert count == 1` edit table — brace surgery is exactly the
case the skill's structural-edit advice is for. Verify with `node --check` afterwards; a mismatched
brace is the only way to get this wrong and it fails loudly.
