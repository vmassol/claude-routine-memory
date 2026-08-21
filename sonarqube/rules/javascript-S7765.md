# javascript:S7765 — `x.indexOf(y) !== -1` → `x.includes(y)`

Platform: **0 drops in 73 shipped** (12 in #6201, then 27 + 34 in #6210/#6211). The "12 workable"
figure of the first pass was **PR-constrained, not rule-constrained** — on a later day with no open
agent PR the same rule read 61 non-vendored sites. Re-derive the workable count every run.

## The only semantic difference

`Array#includes` uses SameValueZero, `indexOf` uses strict equality, so they differ **for `NaN`
only** — `[NaN].indexOf(NaN)` is `-1` while `[NaN].includes(NaN)` is `true`. `node` proof over 63
array/probe combinations returns exactly that one differing pair, and for strings the two are
identical for every substring. So the per-site question is just: **can the probe be `NaN`?** In XWiki
the receivers are URLs, option labels, MIME types, literal state arrays, `String#split` results and
xobject-number arrays — never `NaN`, so the whole pool converts.

## Receiver check (the real drop condition)

`includes` must exist on the receiver. Safe: a `String`, and a real `Array` (including a Prototype-
extended one — Prototype adds `include()`, it does not remove `includes`). **Not** safe on an
array-*like*: `arguments`, a `NodeList`, an `HTMLCollection`, a jQuery object, a Prototype
`Enumerable` that is not an Array. One look at where the receiver was produced settles it — a literal
`[…]`, a `split()`, or a variable only ever `push`ed to is an Array.

## Comparison forms, all four of which fire

| Flagged | Replacement |
|---|---|
| `x.indexOf(y) !== -1` / `> -1` / `>= 0` | `x.includes(y)` |
| `x.indexOf(y) === -1` / `== -1` / `< 0` | `!x.includes(y)` |

Watch the **ternary** shape: `url.indexOf('?') < 0 ? '?' : '&'` becomes
`url.includes('?') ? '&' : '?'` — the branches swap with the negation. Getting that backwards
compiles, minifies and silently builds a wrong query string, so eyeball those sites in the dry-run
diff.

## What the rule does NOT fire on, and must not be "made consistent"

After a batch, the residual `indexOf(…)` comparisons in the touched files read like missed sites but
are not: **`indexOf(x) > 0` and `indexOf(x) == 0` are POSITION tests** ("found, but not at the
start" / "starts with"), which `includes` cannot express. Sonar is right not to flag them; leave
them and say so in *Clarifications*, or the intra-file-consistency sweep breaks the file.

## The densest platform cluster is vendored-inside-an-owned-file

34 of the 61 non-vendored-file sites sat in the `BrowserDetect` block of `xwiki.js` (lines
~1055-1130), a third-party snippet with its own header (`Version: 2.1.6`, Chris Nott, CC-BY 1.0)
embedded in an otherwise XWiki-owned file. The transform is provably safe there (`ua` is
`navigator.userAgent.toLowerCase()`), so this is a *policy* question, not a correctness one — see
the vendored-block note in `learnings.md`. Shipped as the judgement half of the split.

## Co-location with javascript:S6582

Often co-located with the surrounding `&&` guard flagged by
[javascript:S6582](javascript-S6582.md) — check the `(file, line)` intersection, but note the two
rules usually sit on *different* lines of the same file rather than the same line, so they are two
edits, not one.

**When they DO share a line, fix only S7765 and drop the S6582 key.** The shape is
`a && a.indexOf(k) == -1` ("`a` exists AND `k` is not in it"). The S7765 fix keeps the guard
(`a && !a.includes(k)`), and there is no readable `?.` form: `!a?.includes(k)` is `true` when `a` is
absent — the exact opposite of the original — and `a?.includes(k) === false` reads worse than the
guard it replaces. This is the "two issues on the same line" hazard in its unrecoverable form: the
two fixes are not merely order-dependent, they are mutually exclusive.
