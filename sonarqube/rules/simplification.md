# Pure-simplification rules — S1125 / S1488 / S1858 / S2864 / S1612 / S1155 / S1126

> Load only when fixing one of these. Cross-cutting mechanics live in `learnings.md` → *General
> batch-fix techniques*. Best batch fodder — no dataflow check.

Behaviour-preserving with NO use-verification. Oldcore often holds 40-90; else thin-spread across leaf
modules (a ~10-module reactor of 3-5 each clears the target).
- `S1125` redundant boolean literal: `x == true`→`x`, `x == false`→`!x`; ternary shapes `cond ? x :
  true`→`!cond || x`, `cond ? x : false`→`cond && x`, `cond ? true : y`→`cond || y`, `cond ? false :
  y`→`!cond && y` (operand may be a boxed Boolean that autounboxes — fine).
- `S1488` inline an immediately-returned local.
- `S1858` drop `toString()` on a String receiver (TRUST the rule).
- `S2864` `keySet()`+`get(k)` → `entrySet()`; prefer `values().forEach(...)` when key unused, else the
  `entrySet()` enhanced-for (required when the key IS used, or the body throws checked / uses
  `continue`/`break`/mutates an outer local). `Map.Entry` needs no import.
- `S1612` `x -> obj.foo(x)` → `obj::foo`; also block-body `() -> { obj.foo(); }`, ctor `s -> new
  Foo(s)`→`Foo::new`, `x -> x instanceof Foo`→`Foo.class::isInstance`, enum `v -> v.name()`→`Enum::name`,
  qualified super. **Import gotcha (build-breaker):** a method ref names its target TYPE, which the
  lambda never needed imported — if that type is a NESTED class or the stream element type and isn't
  imported, build fails `cannot find symbol`; add the import. (`Type.class::isInstance`/`::cast` need NO
  new import.)
- `S1155` `size()>0`/`==0` → `!isEmpty()`/`isEmpty()`.
- `S1126` if-then-else returning boolean literals → single return: `if (c) {return true;} else
  {return false;}` → `return c;`; the `false`/`true` shape → `return !c;`; the equals-style tail
  `if (!c) {return false;} ... return true;` also collapses to `return c;`. When the flagged condition
  returns `false` you NEGATE it — De Morgan a multi-part `||` (`!(A||B||C)` → `A' && B' && C'`), wrapping
  a >120 result onto a `+4` continuation line. STRUCTURAL (multi-line old-string) → use the assert-guarded
  script with the exact block, never `replace_all`. A `// comment` between the `if` and the final `return`
  survives above the merged return (same as S1066). Concentrated in oldcore (equals()/boolean getters) —
  a DIFFERENT rule from any open oldcore S6201 PR, so oldcore stays fair game for it.
