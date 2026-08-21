# javascript:S6582 — `a && a.b` → `a?.b` (optional chaining)

Long listed as "changes behaviour in at least one shape", which is true and is also a **free
classifier**, not a drop condition. Platform: **0 real drops in 67 shipped** — 36 in #6201/#6202,
then 26 mechanical + 5 judgement in #6210/#6211. The one site not converted was blocked by a
co-located `S7765` issue on the same line (see below), not by this rule.

The pool regenerates and the recorded "workable" count expires: the 36-site pass was constrained by
three concurrent agent PRs claiming files, and a later run with no open agent PR found 32 more.

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

## Outcome datapoint — all three PRs of the sweep merged the same day, uncommented

The split shipped as #6201 (43 mechanical) + #6202 (5 judgement) + #6200 (the Java sibling). **All three
merged within ~13.5 h of being opened, none with a single review comment on the change itself** — and
#6201 merged *14 minutes after* #6202, so the judgement half did not trail the mechanical half either.
Two conclusions:

- The assigned-to-a-variable judgement half is **not** a coin toss for this rule. Write it to be merged;
  keep splitting only so a disagreement cannot hold up the mechanical half, not because the judgement
  half is unwelcome.
- **The JS half was NOT routed to a front-end reviewer** this time, unlike the earlier `S7773`/`S6353`
  batch. See the corrected routing note in `learnings.md`: the day-of-latency is not a property of "a JS
  batch", so do not pre-emptively budget for it.

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
* **A `while` condition is a drop**, unlike an `if`. `while (r && r.type != type)` exits when `r` is
  nullish; `while (r?.type != type)` compares `undefined != type`, which is *true*, so the loop never
  ends. Sonar does not flag these, and the intra-file-consistency sweep must not "fix" them either —
  three such sites sit in `entityReference.js` next to a converted one.
* **A same-line `S7765` issue makes the S6582 fix impossible, not merely awkward** —
  `a && a.indexOf(k) == -1` has no equivalent `?.` form; see
  [javascript-S7765](javascript-S7765.md). Drop the S6582 key there.

## The judgement/mechanical split axis, generalised

The recorded axis (result assigned to a variable → judgement) held again at 5 sites. A second,
unrelated axis can share the same judgement PR as its own commit: **provenance** — sites inside a
vendored block, where the transform is provably safe but touching the code is a policy call. Keep
them as separate commits so either group can be dropped alone, and say so in the body.
