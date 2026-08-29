# `java:S9354` — subtraction in `compare()` / `compareTo()`

> *"Subtracting numeric values in compare/compareTo can overflow; use Integer.compare instead."*
> MAJOR, type **BUG**. Part of the 2026 `S93xx` generation of Sonar rules (see also `S9357`,
> `S9358`, `S9365`), none of which had ever been triaged here.

## The fix

`a - b` → `Integer.compare(a, b)` (`Long.compare` when the operands are `long`). One line, no import.
Only the **sign** of the result is part of the `Comparable`/`Comparator` contract, so the swap is
behaviour-preserving everywhere except the overflow case the rule exists for.

The pool is uniform: XWiki's sites are all `getPriority()`-style `int` accessors or `int` fields
inside a `compareTo`/`compare`/comparator lambda. Verify the operand type once per site by reading
the accessor's declaration (`grep "int getX"`), because `Long.compare` is the fix for a `long` and
`Integer.compare(long, long)` does not compile.

## The one drop condition, and it is invisible until the tests run

**A test may assert the MAGNITUDE of `compareTo`.** `Integer.compare` returns −1/0/1 where the
subtraction returned the difference, so any assertion on the numeric value fails:

```java
// xwiki-platform-resource-api, ResourceReferenceHandlerTest#priority
assertEquals(300, handler1.compareTo(handler2));   // 500 - 200
```

That assertion is over-specified — `Comparable#compareTo` promises only "a negative integer, zero,
or a positive integer" — so adapting it (`assertTrue(handler1.compareTo(handler2) > 0)`) is the
right change, **but it is a judgement call about someone else's test**, so it belongs in the
judgement PR, not in the mechanical batch.

Cheap pre-check, worth doing before the reactor: for each flagged class, grep its module's tests for
`compareTo(` / `compare(` and look for an `assertEquals(<number>,`. That is a grep, versus a build
round (this one cost a 12-module recovery reactor because the failing module was early in the
reactor and `-fae` does not save the modules downstream of it).

## Line length

`Integer.compare(a, b)` is ~14 characters longer than `a - b`. Two commons sites landed at 114 and
one platform pair needed the statement re-wrapped (`this.x =\n    loadConverters(…)` became
`this.x = loadConverters(…,\n    lambda)`). Assert ≤120 on the modified lines in the apply script.
