# Test-code rules — S5786 / S5785 / S3415

> Load only when fixing one of these. Cross-cutting mechanics live in `learnings.md` → *General
> batch-fix techniques*. Deep clean pool once syntax/simplification/unused are drained.

Pure test-code edits (production untouched, low review risk). The module's tests now RUN (no
`-DskipTests`), so these edited tests are exercised directly; Checkstyle also runs in `install`.
- **`S5786` JUnit5 test class/method should be package-private — the best test-rule batch.** Two
  message variants (method-level "Remove this 'public' modifier", class-level "Remove redundant
  visibility modifiers..."). Do NOT infer scope from the message/line (the method-level message often
  points at the CLASS decl). Key the fix BY FILE: strip the leading `public ` (`re.sub(r'\bpublic\s+',
  '', line, count=1)`) from (a) the class-decl line (incl. nested/`@Nested`) and (b) every method whose
  immediately-preceding contiguous annotation block contains a REAL JUnit annotation (`@Test`/
  `@BeforeEach`/`@AfterEach`/`@BeforeAll`/`@AfterAll`/`@ParameterizedTest`/`@RepeatedTest`/`@TestFactory`/
  `@TestTemplate`/`@Nested`). Keep other modifiers (`@BeforeAll public static void`→`static void`). Do
  NOT touch fields, unannotated helpers, or methods with an XWiki-specific (non-JUnit) lifecycle
  annotation (`@BeforeComponent`/`@AfterComponent` stay public). A class-level flag makes the script
  strip ALL that file's test methods, so a dense file yields far more removals than its issue count —
  expected. **Cross-module compile check** (oldcore publishes a widely-used test-jar): for each class
  made package-private, `grep -rl "extends <Class>" --include=*.java xwiki-platform-core | grep -v
  <thisModule>` — the risk is only `abstract`/base test classes (a class NAMED `Abstract*Test` is often
  not abstract — read the decl). Open S5786 PRs usually sit on SIBLING modules (distinct, no file
  overlap) — scope off-limits by exact module path.
- **`S5785` use assertEquals/assertNotEquals/assertNull/assertSame — fully SCRIPTABLE by line number**
  (the message names the exact target). **Default RECEIVER-FIRST for `.equals()` shapes (universally
  safe):** `assertTrue(a.equals(b))`→`assertEquals(a, b)`, `assertFalse(a.equals(b))`→`assertNotEquals(a,
  b)` — JUnit's `assertEquals(expected, actual)` calls `expected.equals(actual)`, so keeping the
  original receiver first reproduces the exact call (correct even for custom/asymmetric equals or a
  different-typed `b`). Never flip operands (a flip caused a real test failure). Other shapes:
  `assertTrue(LIT == x)`→`assertEquals(LIT, x)`; `assertTrue(x != LIT)`→`assertNotEquals(LIT, x)` (covers
  `hashCode() != 0`); `assertTrue(null == x)`→`assertNull(x)`; degenerate `assertFalse(x.equals(null))`/
  `assertTrue(x.equals(x))` convert too (JUnit uses `Objects.equals`). **`==`/`!=` between REFERENCES** is
  a distinct message → `assertSame`/`assertNotSame` (order cosmetic; TRUST the message). Only convert
  FLAGGED lines (`assertTrue(x instanceof Y)` siblings stay). A message arg moves to the end. **Imports:**
  add the new static imports (alphabetical), remove `assertTrue`/`assertFalse` only when the file no
  longer uses them (grep after). Duplicate-line gotcha: identical assert lines can recur — a
  `count==1` assert then trips, diagnose. Already-half-fixed site: a flagged line directly above an
  existing equivalent → DELETE the redundant flagged line. **Run the changed test classes as the
  operand-correctness net:** `install ... -Dtest=<flagged classes> -DfailIfNoTests=false`. Since ONLY
  test code changed, the flagged classes are exactly the tests that can break — this RUNS them, it does
  not skip tests (Checkstyle + full test-compile still run, ~5s even in oldcore). For production-code
  rules never narrow like this: run the whole edited module (see Building / verifying).
- **`S3415` "swap expected/actual arguments" — usually UNSAFE, default DROP.** The rule assumes
  operand order is cosmetic, but many flagged asserts DEPEND on it (same root cause as S5785's "never
  flip operands"): (a) an `assertEquals`/`assertNotEquals` on a type with ASYMMETRIC `equals` — e.g.
  `RegexEntityReference.equals` does regex matching, so `regexRef.equals(plain)` ≠ `plain.equals(regexRef)`;
  swapping flips the result and BREAKS the test; (b) `assertNotEquals(obj, null)` deliberately exercises
  `obj.equals(null)` — swapping to `(null, obj)` short-circuits via `Objects.equals` and no longer tests
  that contract. Only swap when BOTH operands are plain values with symmetric `equals` and neither is
  `null` (e.g. a bare literal genuinely in the actual slot); otherwise DROP. Read the asserted type's
  `equals` before trusting Sonar.
