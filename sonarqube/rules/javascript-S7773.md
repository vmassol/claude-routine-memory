# javascript:S7773 — prefer `Number.parseInt` / `parseFloat` / `isNaN` / `isFinite`

**Split the pool by which global is flagged — only two of the four are free.**

- **`parseInt` and `parseFloat` are provably identical to their `Number.*` twins.** ECMA-262 defines
  `Number.parseInt` as *the same function object* as the global (`%parseInt%`); `node -e
  'console.log(Number.parseInt===parseInt)'` prints `true`. Zero-risk, no reasoning needed per site.
- **`isNaN` and `isFinite` are NOT equivalent** — the globals coerce their argument, the `Number.*`
  versions do not (`isNaN("x")` is `true`, `Number.isNaN("x")` is `false`). Convert **only** when the
  argument is provably already a `Number`. The recoverable shapes are `isNaN(parseInt(…))` and an
  `isNaN(x)` whose immediately preceding statement is `x = parseInt(x)`; everything else is a drop.
  This split saved 3 of 11 `isNaN`/`isFinite` sites on the platform pool.

Drive the edit off the issue's `textRange` and assert the char before the token is not `[\w$.]` and
the char after is `(` — that rules out a property access or a longer identifier.

`Number.parseInt` & co. are ES2015, so browser support is a non-issue for XWiki.

**Merged in xwiki-platform (#6186), approved by the front-end reviewer with no change requested** — so the `parseInt`/`parseFloat` subset (and the 3 provable `isNaN` sites) is confirmed acceptable as-is; a future JS run does not need to re-argue it.
