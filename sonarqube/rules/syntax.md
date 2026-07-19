# Pure-syntax/annotation group — S1128 / S1197 / S1116 / S1161 / S1611 / S1124 / S3878

> Load only when fixing one of these. Cross-cutting mechanics live in `learnings.md` → *General
> batch-fix techniques*. Safest fodder — zero dataflow, deep regenerating pools.

A wide reactor cleanly satisfies the override. Apply by line number in one assert-guarded script.
- `S1128` unused import: delete the flagged `import ...;` (assert it starts `import`, ends `;`). TRUST
  Sonar (it keeps `{@link}`-only imports). A delegating subclass can legitimately have 10-14 removable.
- `S1197` array designator: `TYPE NAME[]`→`TYPE[] NAME`. **Regex gotcha:** a naive `\b(\w+)\s+(\w+)\[\]`
  wrongly grabs a return type already in `[]` form when a modifier precedes it. Use negative lookahead
  `\b(\w+)(\s+)(\w+)\[\](?!\s*\w)` → `\1[]\2\3` (a `[]` followed by ` identifier` is already correct).
- `S1116` empty statement: lone `;` (delete line); trailing `;;` (strip one); `};` where `}` closes a
  block/method (strip the `;`). NEVER strip `;` from `new Foo(){...};` / `Type x = ...{...};` (required;
  Sonar won't flag those). Distinguish by exact line number.
- `S1161` missing `@Override`: insert a line `<indent>@Override` ABOVE each flagged signature (purely
  additive). Deep pool, often concentrated in TEST files (anonymous-class methods). TRUST Sonar; assert
  the line has `(` and neither it nor its predecessor is already `@Override`. Interface methods
  redeclaring a super-interface method legitimately take `@Override` (Java 6+).
- `S1611` redundant lambda-param parens: `(x) -> ...` → `x -> ...` (single untyped param only). Thin-
  spread across modules; ideal filler to bundle into any batch. Match a UNIQUE token — `(x) -> {` or
  `(x) -> body`, not the bare `(x)` (which recurs); a `.thenAnswer((invocation) -> ...)` shape is common
  in Mockito test setup. Assert the per-file count (a file can hold >1 flagged lambda with the same body).
- `S1124` modifier order: reorder the LEADING modifier run to canonical JLS order (public/protected/
  private → abstract → static → final → transient → volatile → synchronized → native → strictfp). Almost
  always `final static`→`static final` or `static public`→`public static`. FULLY scriptable: regex-consume
  the leading run of modifier keywords, sort by canonical index, keep the type+rest verbatim. Zero
  behaviour/visibility change → NO `@since` even on public constants. Usually ZERO open PRs (cleanup waves
  skip it); dense in oldcore + spread across many leaf modules — a solid unclaimed backbone batch.
- `S3878` arrays created for varargs: remove the `new T[]{...}` wrapper and pass the elements. TWO message
  variants, both usually clean spreads: "Remove this array creation / and simply pass the elements" AND
  "Disambiguate by casting as Object/Object[]" (the latter is typically `MessageFormat.format`, SLF4J
  `LOGGER.x`, `Arrays.asList`, or reflection `getMethod`/`getConstructor`/`newInstance` — spreading is
  equivalent and preferred). DROP the genuinely ambiguous ones: an EMPTY-array delegation `foo(x, new
  Object[]{})` and a single `new Object[]{y}` where `y` could itself be an array. **Empty-array nuance:**
  reducing `new T[]{}` to no-args is SAFE for external-class reflection (`getConstructor()`/
  `newInstance()` have unambiguous no-arg forms — a clean fix); it is a DROP only when a fixed-arity
  overload of the SAME method name exists, because `serialize(x, new Object[]{})`→`serialize(x)` then
  binds the non-varargs `serialize(x)` overload — often the enclosing method itself → INFINITE RECURSION
  (grep the class for a same-name overload before reducing an empty-array self-call). Multi-line arrays: edit the open line (drop `new T[]{`) and the close line (drop one
  `}`), then normalize any over-indented continuation to +4. Concentrated in oldcore + legacy.
