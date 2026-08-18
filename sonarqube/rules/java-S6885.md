# java:S6885 — use `Math.clamp` instead of nested `Math.min`/`Math.max`

Java 21 (XWiki's source level), so the rule is never disqualified for the JDK version. Four
overloads exist: `clamp(long,int,int)`, `clamp(long,long,long)`, `clamp(double,double,double)`,
`clamp(float,float,float)` — a `float` value with `int` bounds resolves to the `float` one and
returns `float`.

Two things to check per site, both decidable from the flagged line:

1. **Which argument is the value.** `Math.max(lo, Math.min(hi, v))` and `Math.min(Math.max(v, lo), hi)`
   both become `Math.clamp(v, lo, hi)` — read the nesting, do not pattern-match the outer call.
2. **`Math.clamp` throws `IllegalArgumentException` when `min > max`; the nested form silently
   returns a bound.** So the fix is only equivalent when `lo <= hi` is guaranteed. Usually it is
   obvious from the line above (`Math.clamp(start + limit, start, list.size())` is safe because
   `start` was itself clamped into `[0, list.size()]` on the previous line).

NaN is *not* a difference: `Math.clamp(NaN, 0, 1)` and `Math.max(0, Math.min(1, NaN))` both give NaN
(verified with a throwaway `java` file).

Reviewed and merged in xwiki-platform with no objection to the transform itself — this rule is
welcome here, so it does not need to ride in a judgement-call PR next time.
