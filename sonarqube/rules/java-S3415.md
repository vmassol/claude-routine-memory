# `java:S3415` — "Swap these 2 arguments so they are in the correct order: expected value, actual value"

The OKF's `test-code-rules` entry says **"usually unsafe — default to dropping it"**, and
`pool-state.md` used to say **"Default DROP"**. Both mis-set the prior: a real platform sweep went
**15 fixed / 3 dropped (83%)**, green first try. The drop conditions in the OKF are correct; what is
wrong is treating them as the common case.

## The free classifier: does either operand read `null`?

Decidable from the one flagged line, with no whole-file read and no `equals()` reading:

- **`null` on either side → DROP.** `assertNotEquals(new X(...), null)` deliberately exercises
  `x.equals(null)`; swapping to `(null, x)` short-circuits inside `Objects.equals` and stops testing
  the contract. A file that asserts *both* directions on consecutive lines
  (`assertNotEquals(event1, null); assertNotEquals(null, event1);`) is saying so explicitly.
- **An asymmetric `equals()` → DROP.** `RegexEntityReference.equals` does regex matching, so the
  operands are not interchangeable. This is rare and the type name usually gives it away
  (`Regex*`, a matcher, a pattern-holder).
- **Everything else swaps.** In practice the pool is: an actual-side getter against a constant
  (`assertEquals(this.sp1.getScopeReference(), WIKI_REFERENCE)`), a `List` against a `List`
  (`assertEquals(range.subList(x), x)`), a computed `String` against a `CONTENT_*` constant, a
  numeric expression against a literal (`assertEquals(Math.signum(a.compareTo(b)), -1.0)`).

## Convert the whole FILE, not just the flagged lines

Sonar flags only some of the reversed assertions in a file, apparently arbitrarily — `RangeTest` had
**10 reversed assertions and 2 flagged**, `PropertyClassTest` 4 and 2 (the `-1.0` sibling of a flagged
`1.0` line went unflagged). Shipping only the flagged half leaves the file reading two ways, which is
what a reviewer objects to. Swap all of them and count only the flagged keys as fixed.

## Mechanics

- Overload resolution is unchanged by the swap, but check it when the operands have *different*
  numeric types. `Math.signum(int)` returns `float` (the `float` overload is more specific), so both
  `assertEquals(float, double)` and `assertEquals(double, float)` resolve to
  `assertEquals(double, double)` — same method before and after.
- Identical lines repeat inside one file, so an `assert count == 1` fails. Either extend the `old`
  with the unique preceding line (`Range range = new Range(0, 0);`) or assert the exact expected
  count for a global replace (6 × `CONTENT_VERSION1`, 2 × `CONTENT_VERSION2` in one file).
- Line length never grows: the swap moves characters, it does not add any.
- Test-only, so the module's own suite is the whole verification. Zero coverage risk.

## Where the pool is

Ordinary tests, spread thin: 8 in one `*RunnableTest`, 2-3 each elsewhere. The `equals()`/`hashCode()`
contract tests of the *same* modules supply nearly all the drops — see
`okf/sonarqube/test-code-rules.md` on the sibling `S5785`, whose suppressions exist precisely to stop
an S3415 sweep from breaking them.
