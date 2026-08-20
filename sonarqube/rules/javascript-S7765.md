# javascript:S7765 — `x.indexOf(y) !== -1` → `x.includes(y)`

Platform: 12 workable sites, **0 drops** (shipped in #6201). The pool is ~90 project-wide but most of
it sits in the vendored scripts and in files other agent PRs claim.

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

Often co-located with the surrounding `&&` guard flagged by
[javascript:S6582](javascript-S6582.md) — check the `(file, line)` intersection, but note the two
rules usually sit on *different* lines of the same file rather than the same line, so they are two
edits, not one.
