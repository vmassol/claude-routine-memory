# javascript:S6582 — `a && a.b` → `a?.b` (optional chaining)

Long listed as "changes behaviour in at least one shape", which is true and is also a **free
classifier**, not a drop condition. Platform: 36 workable sites, **0 drops**, split 31 mechanical +
5 judgement across two PRs (#6201 / #6202).

## The one difference, and what it implies

`&&` short-circuits on any falsy left operand and yields **that operand**; `?.` short-circuits only
on `null`/`undefined` and yields `undefined`. Prove it once per run with a throwaway `node` program
(cheap, and its output is the strongest thing you can put in the PR body):

```
(a && a.b) vs (a?.b): same truthiness for all 14 falsy/truthy shapes = true
   raw value differs only for falsy-but-not-nullish a: null, 0, -0, "", false, NaN, 0n
(a && a.f()) vs (a?.f()):  a=0/""/false  ->  old: 0/""/false   new: TypeError
```

The second line is the one that matters and is easy to miss: **`0?.f()` does not short-circuit, it
evaluates `(0).f()` and throws.** Same for a further member access after the `?.` —
`0?.memo.elements` throws where `0 && 0.memo.elements` yielded `0`.

## Three buckets, decidable from the flagged line plus one look at the receiver's producer

1. **A single property access whose result is used only for truthiness, or with an `||` fallback** —
   `if (a && a.b)`, `$((data && data.elements) || document)`, `return (r && r[1]) || ''`,
   `!o || !o.data` → `!o?.data`. **Unconditionally equivalent**, no receiver check needed. ~30% of a
   pool.
2. **A call, or a further member access, after the `?.`** — `this._x && this._x()` → `this._x?.()`,
   `el && el.up('.x').show()`, `(event && event.memo.elements) || […]`. Needs a one-line check that
   the receiver can only be an object or nullish. In the WAR scripts every receiver was: a Prototype
   `$()` / `.down()` / `.next()` result (`Element` or `null`/`undefined`), a `Class.create` method
   (function or absent), a DOM/Prototype event (the `init(event)` functions are called either with no
   argument or as an observer), `console`, `xhr.upload`, a jqXHR, or a promise assigned only inside an
   `if`. ~60% of a pool and still 0 drops — just not free.
3. **A string receiver** — `s && s.startsWith(x)`, `s && s.substr(0,6) === 'x'`. The only falsy string
   is `''`, and `''.startsWith(…)`/`''.substr(…) === 'x'`/`''.endsWith(…)` are all `false`, i.e. the
   same branch `&&` took. Safe, and worth saying so explicitly in the PR.

## Outcome datapoint — both halves merged

The split shipped as #6201 (43 mechanical) + #6202 (5 judgement). **#6202 merged uncommented** and
#6200 (the Java sibling) merged too; the mechanical #6201 was still awaiting the front-end reviewer at
that point, exactly the day-of-latency the JS-routing note in `learnings.md` predicts. So the
assigned-to-a-variable judgement half is NOT a coin toss for this rule — write it to be merged and keep
splitting only so the mechanical half is not held up, not because the judgement half is unwelcome.

## Where the judgement half sits

The result **assigned to a variable** (`var icon = option && option.icon;`) is the one shape whose
value, not just its truthiness, is observable — put those in the sibling judgement PR even when the
receiver is provably an object. Follow the variable's uses: a `typeof x === 'object'` / `'string'`
test or a plain boolean use treats every falsy value alike, which is what makes them correct.

## Mechanics

* `?.` is **already used** in `xwiki-platform-web-war` (`dashboard.js:117`, `autosave.js:48`,
  `actionButtons.js:30-32,326,752`), so the toolchain is proven — grep for it rather than worrying
  about `closure-compiler`. `yuicompressor` in that pom only aggregates CSS.
* Sonar's `textRange` covers the whole `&&` expression, so an exact-string edit keyed to
  `(file, line, old_substring)` is unique; one site per line, several sites per file.
* Never write `??` — the original test was truthiness, not nullishness (same trap as
  [javascript-S6644](javascript-S6644.md)).
