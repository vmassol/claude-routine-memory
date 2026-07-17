# Other clean rules — S6204/S6211 / S2093 / S2119 / S1143+S1163 / S5361 / S2147 / S3626

> Load only when fixing one of these. Cross-cutting mechanics live in `learnings.md` → *General
> batch-fix techniques*. Re-query each run.

- **`S6204`/`S6211` `Stream.collect(Collectors.toList()/toSet())` → `Stream.toList()`/`.toSet()`** — a
  deep (~100 open) rarely-PR-touched pool; the GO-TO pivot when the mechanical/simplification/unused/
  S1066/S6201 families are ALL simultaneously PR-drained to 0 (a common concurrent-session state — verify
  with a facet query). Thin-spread across modules (a wide ~20-25-module reactor is fine — 59 sites in
  one green build, excluding the slow Solr/`-index` modules); or aggregate a dense same-family reactor
  (e.g. the 3 livedata modules held 25). CAVEAT: `.toList()` is UNMODIFIABLE, so the check is not just
  "is it mutated here" but "**can it ESCAPE into a public API where an external caller could mutate
  it**" (a reviewer WILL push back on this — it is the #1 objection). Convert only when the result stays
  confined: returned/iterated/`isEmpty`/`size`/`get`/`toArray` locally, or used as an `addAll` SOURCE
  (elements copied out, the unmodifiable list discarded); a test-code list built only for
  `assertEquals`/iteration is likewise safe (near-0 drops in tests). **"Passed to a setter/ctor" is NOT
  automatically safe** — if the setter stores the list BY REFERENCE (`this.x = x;`) on a non-`internal`
  public model class whose getter returns it directly (`return x;`), the unmodifiable list becomes the
  live backing list of a public getter and an extension doing `obj.getX().add(...)` breaks at runtime →
  DROP, OR (reviewer-preferred when the model class is xwiki-owned) keep the `.toList()` and make the
  SETTER defensively copy: `this.x = x == null ? null : new ArrayList<>(x);` — clears the Sonar issue
  AND preserves the mutable-getter contract, so the escaping list no longer matters. READ the target
  setter+getter to confirm copy-vs-by-reference before trusting it. A REST JAXB
  response DTO (`*.rest.model.jaxb`) built once and only serialized stays safe. Also DROP if the result
  is later `add`/`set`/`remove`/`sort`/`removeIf`-ed or assigned to an `ArrayList`-typed target. Delegate
  the per-site dataflow read to subagents (general-purpose, not Explore, and verify their edits landed —
  General techniques), telling them to trace the FULL escape path (setter → field → public getter), not
  just the immediate use.
  `.toList()` is 19 chars shorter than the original so line length never breaches. **Auto-derive the
  orphaned-import removal:** after converting a file's flagged lines, drop `import
  java.util.stream.Collectors;` (or the `import static java.util.stream.Collectors.toList;` variant,
  which also shows up as `.collect(toList())`) iff `Collectors.`/`toList(` no longer appears in the
  body (KEEP when a sibling `Collectors.toSet/joining/toMap/groupingBy` survives) — reproduces the
  correct per-file REMOVE/KEEP with no manual bookkeeping. **PITFALL — the "keep import" substring
  check is fooled by a surviving `import static java.util.stream.Collectors.joining;` (or any
  `Collectors.X` static import): that line literally contains `Collectors.`, so a naive
  `"Collectors." in content` test wrongly KEEPS the now-unused plain `import
  java.util.stream.Collectors;` → Checkstyle `UnusedImports` FAILS the build.** Test for `Collectors.`
  usage OUTSIDE `import` lines (strip/skip lines whose lstrip starts with `import`), not anywhere in
  the file. A plain `import java.util.stream.Collectors;` and a `import static
  ...Collectors.joining;` can COEXIST — the static one being used does NOT keep the plain one alive.
- `S2093` try-with-resources: `R r = new ...(); try {...} finally { r.close() }` → `try (R r = new
  ...()) {...}`. ~half of hits are NOT real closes (push/pop, `reset()`, semaphore release, resource
  created mid-body) — verify the finally actually CLOSES an `AutoCloseable` declared just before `try`.
  Implicit `close()` throws `IOException`; if the surrounding catch is narrower and the method doesn't
  declare it, add a `catch (IOException)`. Removing `IOUtils.closeQuietly` may orphan the `IOUtils` import.
- `S2119` reuse Random: extract `new Random()`/`new SecureRandom()` to a `private static final` field.
- `S1143`+`S1163` (a `finally` throws, masking the try's exception): fire as a PAIR. Replace the `throw`
  in the finally with `logger.warn("Failed to close ...: [{}]", ExceptionUtils.getRootCauseMessage(e))`.
  Wrap EVERY placeholder in `[{}]`. If an XWiki `@Component` with no logger, add `@Inject private Logger
  logger;` (org.slf4j, NOT static) + the ExceptionUtils import (import order: java/javax, then org.*
  alphabetical — slf4j between apache and xwiki). Grep a sibling for the existing message style.
- `S5361` replaceAll→replace: convert only when the first arg reduces to a literal (no regex metachars,
  or a single escaped one `"\\+"`→`"+"`) AND the replacement has no `$`/`\`.
- `S2147` combine catch clauses with identical bodies: `catch (A e){B} catch (B e){B}` → `catch (A | B
  e){B}`. Gotchas: (1) if the types are in a SUBTYPE relationship, multi-catch is a COMPILE ERROR —
  DELETE the redundant subclass catch instead. (2) Deleting a catch can ORPHAN that exception's import.
  (3) if the merged/removed catches had DIFFERENT comments, MERGE both into the surviving block — a
  reviewer will flag a dropped comment.
- `S3626` remove redundant jump — clean ONLY for a TRULY-trailing jump (last statement of a void
  method; a branch-final `continue`/`return` when the if-else chain is the WHOLE loop body). DROP: the
  jump is the branch's only statement (removing → EMPTY block, checkstyle `EmptyBlock`); complex nested
  try/catch/finally.
