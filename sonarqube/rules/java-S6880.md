# `java:S6880` — "Replace the chain of if/else with a switch expression"

Fires on an `if`/`else if` chain over one selector. XWiki targets **Java 21**
(`xwiki.java.version` in the root pom), so pattern matching for `switch` is available and this is a
legitimate modernization, not a preview feature. No OKF entry.

**The pool is NOT only `instanceof` chains** — that assumption would have written off a fifth of it.
Of platform's 25 sites, 5 tested `==` against **`int` or enum constants** (`if (value == 1) … else if
(value == 0)`, `if (contentType == ContentType.PURE_TEXT) …`) and one used several constants per
branch (`a == X || a == Y`), which becomes a comma-separated case label. Those convert with *less*
risk than the pattern ones: an `int` selector has no `null` question at all. Read the flagged line's
condition before assuming the rule's shape.

## The transform

```java
T x;
if (o instanceof A a)      { x = f(a); }
else if (o instanceof B b) { x = g(b); }
else                       { throw new E(…); }
```
becomes
```java
T x = switch (o) {
    case A a -> f(a);
    case B b -> g(b);
    default -> throw new E(…);
};
```
It also deletes the bare local declaration the chain was assigning into, so the diff is strongly
net-negative (89 removed / 51 added over 8 sites). When the chain's value is returned immediately,
collapse it further to `return switch (o) { … };`.

## `case null` is the one real behaviour difference

A `switch` whose labels are patterns throws **`NullPointerException` on a `null` selector** unless a
`case null` label is present — the `if`-chain sent `null` to the `else`. So bucket every site by what
its `else` branch does with `null`:

- the `else` **accepts** `null` (returns `false`, wraps it, passes it on) → `case null, default -> …`;
- the `else` **already dereferences** the selector (`throw new E("…" + o.getClass() + "]")`, the usual
  XWiki shape) → a plain `default` is equivalent, both forms NPE.

Say which of the two each site got, in the PR body; it is the only question a reviewer can have.

## The compile error is a FINDING, not a script bug

`switch` enforces a **dominance** rule that an `if`-chain does not: `case S` after `case T` where
`S <: T` is a compile error (*"this case label is dominated by a preceding case label"*), whereas the
same order in an `if`-chain is merely **dead code**. So when the conversion refuses to compile, the
original chain has an unreachable branch.

`ExtensionUtils#wrap` is the worked example: `IndexedExtension extends RatingExtension extends
RemoteExtension`, and both supertypes are tested before it, so
`else if (extension instanceof IndexedExtension)` can never run and an `IndexedExtension` is wrapped
as a `WrappingRatingExtension`. **Do not "fix" it inside the sweep** — deleting the branch or moving
it first are both product decisions (the second changes what the method returns). Revert that one
site, leave the issue open, and report the dead branch in the PR body: that is worth more than the
issue count.

## Drop condition 1: the enclosing method's Checkstyle CyclomaticComplexity

**A `case` label counts towards `CyclomaticComplexity` where an `else if` did not**, so a chain of N
branches in a method already at the cap (10) pushes it over and `checkstyle:check` fails the module
*after* its tests pass — a whole build round. It fired on 1 of platform's 20 applied sites
(`R160300000XWIKI17243DataMigration#getValues`, 10 → 11). There is no way to write around it: the only
form that keeps the count is a plain `default`, which is a behaviour change wherever the `else`
accepted `null`. **Drop the site**, and say in the PR body that the codebase's own metric rejected the
merged form. Pre-check it in the apply script when the enclosing method is long: count the existing
`if`/`for`/`while`/`case`/`catch`/`&&`/`||`/`?` in it.

## Drop condition 2: a branch that is not a single expression

A `try`/`catch`, or a nested `if` producing two
different values. The arm then needs a block body with `yield`, and the result reads worse than the
chain it replaces, which is a review comment waiting to happen. A nested `if` that is a plain
either/or *does* fit, as a ternary inside the arm (`StAXUtils#getXMLStreamWriter`).

Also drop a chain that passes a **raw** generic (`case Map map`): the `if`-chain's raw
`instanceof Map map` compiles with an unchecked warning, but a raw type pattern in a `switch` is
worse, and narrowing it to `Map<?, ?>` breaks the constructor call it feeds.

## Outcome

**Platform: 19 of 25 shipped** in one PR (`xwiki/xwiki-platform#6247`, 17 files / 22 modules) — 6
drops: 4 non-single-expression arms, 1 nested-`if`-plus-`return`, 1 CyclomaticComplexity rejection.
The `case null, default` bucketing covered 14 of the 19; a plain `default` covered 2 (the `else`
already dereferenced the selector); 3 were primitive-`int` selectors with no null question.

**The commons batch (8 sites, `xwiki/xwiki-commons#1927`) merged within hours, with no review at all
recorded and no comment** — so the `switch`-expression form is uncontroversial in XWiki and this rule
is *not* the coin-toss a "judgement" label suggests. Ship it as its own PR anyway (that is what let
it land while the platform batch was still waiting), but write it to be merged: the branch-order and
`case null` bucketing table above is the whole review story, and stating it up front is what leaves a
reviewer nothing to ask.

## Verification

The compiler is most of it (dominance, exhaustiveness, arm types are all checked), and the module's
own tests cover the rest. Nothing here is invisible to `javac` except the `null` question, which is
why the bucketing above is the whole review story.
