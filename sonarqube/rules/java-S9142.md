# `java:S9142` — "Expensive compilation … should not be performed inside loops"

The XWiki pool is entirely **a regex compiled per iteration** because it was passed as a `String` to
`String#matches()` / `String#replaceAll()` inside a loop. Fix: a `private static final Pattern`
constant plus `PATTERN.matcher(s).matches()` / `.replaceAll(x)`.

## Why it is behaviour-preserving (put this in the PR body)

The JDK *defines* `s.matches(r)` as `Pattern.compile(r).matcher(s).matches()` and
`s.replaceAll(r, x)` as `Pattern.compile(r).matcher(s).replaceAll(x)`. Same regex, same replacement
semantics (`$0`, `$1`, backslash escapes), same result — only the compilation moves. No `node`-style
equivalence program is needed; quote the javadoc contract instead.

A regex built from a runtime value converts too (`TransactionException.NEWLINE` is
`System.getProperty("line.separator")`): the constant is then compiled at class-init from the same
value. Declare it AFTER the constant it reads — static initializers run in source order.

## The one real drop shape: `String#split` with a single-character separator

`String#split` has a fast path that avoids `Pattern` entirely when the separator is one
non-metacharacter, so the rule's premise is false there and `Pattern.split` would be *slower*.
Platform's `DefaultLiveDataEntriesResource:161` (`constraint.split(":", 2)`) is that shape — a
genuine false positive, dropped rather than pessimized. Check the flagged call before assuming the
whole pool converts.

## Pool

Never triaged before 2026-08: platform 10, rendering 1, commons 0 — thin-spread, one or two per
module over 8 modules, ~half in `src/test`. 10 of 11 shipped. Regenerates from ordinary code.

## Mechanics

- Anchor the new constant on an existing `private static final` declaration — but **if that
  declaration has a Javadoc block above it, insert above the COMMENT, not between the comment and
  the field**. Getting this wrong silently re-attributes the existing Javadoc to your constant, and
  it compiles.
- Give the constant a Javadoc line in files whose other fields have one; skip it where they don't.
- `import java.util.regex.Pattern;` sorts after every `java.util.X` import — and in a file with no
  `java.*` imports at all it opens a new first group with a blank line after it.
