# `javascript:S6644` — "Unnecessary use of conditional expression for default assignment"

`(x) ? x : y` → **`x ?? y`**. This is a reviewer style call from `@manuelleduc` on #6197, so apply it
from the start: *"while `||` is valid, `??` is preferred as it does not implicitly apply type
conversion — `||` accepts `0`, `''` or `false`, while `??` only accepts `null` and `undefined`."*

**An earlier version of this file said "`||`, never `??`". That was wrong as a default** — it
optimised for matching the original ternary's truthiness test, but the codebase prefers the operator
that says what is actually meant. Do not revert to `||` on the grounds of literal equivalence.

## But `??` IS a behaviour change — check the operand per site

The two forms agree **only** on `null` and `undefined`; they differ for `false`, `0`, `-0`, `NaN`,
`''` and `0n`. Since the flagged code tested *truthiness*, switching to `??` changes behaviour at any
site where a falsy-but-not-nullish value can reach the operand. So:

- **Optional object/map argument** (`opt`, `newParams`, an options bag) → `??` is equivalent in
  practice and states the intent. This is most of the pool.
- **The operand can legitimately be `0`** → do not silently switch. Read it: this is usually a
  *finding*, not a blocker. In `entityReference.js` `parentType` comes from `typeSetup[currentChar]`
  guarded by `typeof … === 'number'` (a guard written to admit `0`) and `EntityType.WIKI` **is** `0`,
  so `||` was discarding a valid parent type and falling through to `DEFAULT_PARENT[currentType]` —
  correct today only because the sole `0`-valued entry's default happens to be `WIKI` as well. `??`
  removed a latent trap, and saying so is what turned a style nit into an accepted fix.
- **Watch the consumer when switching a defaulted object.** `opt = opt ?? {}` followed by
  `'target' in opt` means a caller passing a falsy primitive now throws a `TypeError` where `||` gave
  it `{}`. No real caller does that, but flag it in the reply rather than letting the reviewer find it.

`??` is ES2020 and had no precedent in the WAR scripts; `mvn install -Plegacy,quality -pl
…/xwiki-platform-web-war` came back `BUILD SUCCESS` (3:52 warm) with the `.min.js` files regenerated,
so closure-compiler parses and minifies it. Never mix `??` with `||`/`&&` in one expression without
parentheses — that is a `SyntaxError`.

Drop only if the two branches are not the same expression (then it is not this rule's shape) or if
the condition has a side effect, which would then run twice. Neither has appeared yet.
