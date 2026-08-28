# `java:S8786` — "Regular expressions should not cause non-linear backtracking"

**Whole-rule drop, all three repos** (platform 17, commons 2, rendering 3 — the only rule with fresh
keys in commons/rendering on the day it was triaged, which is why it looks tempting). Never
attempted; the triage below is one line read per site and settles it.

## Why it is a dead end

The rule's own *How to fix it* lists four strategies (negated character classes instead of `.`,
bounded quantifiers, unambiguous alternation, **possessive quantifiers / atomic groups**) and names
two causes, one of which is *"polynomial when unanchored: without a start anchor the engine retries
the pattern at every position"*.

**The decisive evidence: one of the flagged platform sites already carries the recommended fix.**
`TagPlugin:479` is `tags.trim().split("\\s*+,\\s*+")` — fully possessive — and Sonar flags it anyway.
So the mechanical remediation does not reliably clear the issue, and a fix that does not clear the
issue is worthless to this routine.

## And the other strategies are not equivalence-preserving

The rest of the pool is `Pattern` constants whose "simplification" changes what they match:

- `Pattern.compile("(.*)/(.*)")` (rendering `Syntax`, `DefaultSyntaxRegistry`) → the greedy `(.*)`
  splits on the **last** `/`; the suggested `([^/]*)/(.*)` splits on the **first** one. Different
  behaviour for any input with two separators, silently.
- `"\\s*,\\s*"` / `"\\s*;\\s*"` used with `String#split` (5 sites) — anchoring is what the rule wants
  and is exactly what a `split` pattern must not have.
- The URI/parser constants (`DefaultURLSecurityManager`'s RFC-3986 pattern, `EditForm`'s
  `^((?:[\S ]+\.)+[\S ]+?)_(\d+)_(.+)$`, `BaseObjectReference`'s `(\\*)\[(\d*)\]$`) are
  specification-shaped: rewriting them is a parser change needing its own tests, well past the
  15-minute mechanical bar.

## Generic lesson worth keeping

Before committing to a never-triaged rule, **check whether one of its own flagged sites already
contains the remediation the rule recommends**. It costs one line read per site, and a hit means the
analyzer will keep the issue open after the fix — i.e. the rule is unfixable *for this codebase*, not
merely expensive. Same shape of check as reading a rule's *Exceptions* section to learn that a comment
suffices (`S108`).
