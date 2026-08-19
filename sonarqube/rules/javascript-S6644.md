# `javascript:S6644` — "Unnecessary use of conditional expression for default assignment"

`(x) ? x : y` → **`x || y`**, never `x ?? y`. The original test was *truthiness*, so `||` is the exact
translation; `??` only rejects `null`/`undefined` and would change behaviour for `0`, `''`, `false`,
`NaN` and `-0`. Sonar's message does not distinguish them — pick from the code, not the rule.

Equivalence is total and worth proving once with a `node` loop over
`undefined, null, false, 0, -0, NaN, '', 0n` plus a few truthy values (compare with a `Number.isNaN`
escape hatch, since `NaN !== NaN`).

Three shapes seen in the WAR scripts, all one-liners with no drops:

- `opt = (opt) ? opt : {};` → `opt = opt || {};`
- `currentType = parentType ? parentType : DEFAULT_PARENT[currentType];` → `parentType || …`
- inside a call: `$H((newParams) ? newParams : this.params)` → `$H(newParams || this.params)`

Drop only if the two branches are not the same expression (then it is not this rule's shape) or if the
condition has a side effect, which would then run twice. Neither has appeared yet.
