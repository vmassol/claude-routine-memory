# `java:S6880` — "Replace the chain of if/else with a switch expression"

Fires on an `if`/`else if` chain of `instanceof` **pattern** tests over one selector. XWiki targets
**Java 21** (`xwiki.java.version` in the root pom), so pattern matching for `switch` is available and
this is a legitimate modernization, not a preview feature. No OKF entry.

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

## Drop condition

**A branch that is not a single expression** — a `try`/`catch`, or a nested `if` producing two
different values. The arm then needs a block body with `yield`, and the result reads worse than the
chain it replaces, which is a review comment waiting to happen. A nested `if` that is a plain
either/or *does* fit, as a ternary inside the arm (`StAXUtils#getXMLStreamWriter`).

Also drop a chain that passes a **raw** generic (`case Map map`): the `if`-chain's raw
`instanceof Map map` compiles with an unchecked warning, but a raw type pattern in a `switch` is
worse, and narrowing it to `Map<?, ?>` breaks the constructor call it feeds.

## Outcome

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
