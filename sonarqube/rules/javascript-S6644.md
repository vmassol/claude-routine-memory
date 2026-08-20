# `javascript:S6644` — "Unnecessary use of conditional expression for default assignment"

**Use `x || y`, not `x ?? y`.** Settled on #6197 after a full round trip in both directions, so do not
re-litigate it.

The reasoning is structural to the rule and is what a future run must carry: **`S6644` fires on
`(x) ? x : y`, a *truthiness* test.** `||` is therefore the exact translation, while `??` narrows the
fallback to nullish only and so changes behaviour for `false`, `0`, `-0`, `NaN`, `''` and `0n`.

`@manuelleduc` prefers `??` **as a general JavaScript style** ("`||` accepts `0`, `''` or `false`,
while `??` only accepts `null` and `undefined`") — and that preference is real, so expect to be told
it on any PR touching a default assignment. But he closed the loop himself once the consequence was
visible: *"don't replace `||` with `??` if it's a behavioral change."* For this rule it essentially
always is one, because of what the rule fires on. `??` would be right only if the original code had
tested `!= null`.

**Do not try to rescue `??` with a per-site proof.** Bounding the operand is not worth it in a cleanup
PR, and two of the three platform sites cannot be bounded at all (`livetable.js#serializeParams` and
`xwiki.js#shortcut.add` are both reachable from wiki-page and extension code). `xwiki.js` is the
cautionary one: `opt` is consumed as `'target' in opt`, so `opt = opt ?? {}` turns a caller passing
`false` into a `TypeError` where `||` quietly supplied `{}` — the "more precise" operator is the less
robust one there.

## A `0`-valued enum is worth noticing, but it is not this rule's business

`entityReference.js:453` has `parentType || DEFAULT_PARENT[currentType]` where `parentType` can be
`EntityType.WIKI`, which **is `0`** — the guard above it (`typeof … === 'number'`) exists precisely to
admit `0`. So `||` discards a legitimate value and leans on `DEFAULT_PARENT` to return the same thing.
It is provably harmless today (`parentType` is `0` only when `currentType === SPACE`, and
`DEFAULT_PARENT[SPACE]` is `WIKI` too), but that is a property of the lookup tables, not of the code.
Report it as an observation for a possible JIRA issue; do **not** "fix" it inside a Sonar sweep — the
reviewer will read the operator swap as the behaviour change it is.

`||` clears the issue just as well as `??`, so nothing is lost. Drop the site only if the two branches
are not the same expression (then it is not this rule's shape) or if the condition has a side effect,
which would run twice. Neither has appeared yet.
