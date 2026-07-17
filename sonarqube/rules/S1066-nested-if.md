# java:S1066 — merge collapsible nested `if`

> Load only when fixing S1066. Cross-cutting mechanics live in `learnings.md` → *General
> batch-fix techniques*.

Structural; reliable pivot when all else drained. Reliable when
syntax/simplification/unused/test pools are all drained — STRUCTURAL, so those cleanup waves never
touch it, and it regenerates. Density NOT reliably oldcore-concentrated (sometimes ~70 in oldcore =
one batch; other runs thin-spread across ~16 modules). A wide reactor clears it in one `install`.
Sonar flags the INNER `if`. Fix: `if (A) { if (B) { BODY } }` → `if (A && B) { BODY }` — merge with
`&&`, delete the inner `if` line, DEDENT the body by 4, remove ONE trailing brace. Wrap an operand
containing top-level `||` in parens. NOT a pure line-keyed edit → DELEGATE to parallel subagents,
each file bottom-up. **A triple-nest `if(A){if(B){if(C){}}}` collapses to `if (A && B && C)` and
resolves TWO keys** — count accepted issues by KEY.

**Fixable ~75%.** DROP: merged condition >120 that can't cleanly two-line-wrap (+4 continuation
indent); the inner `if` is not the sole statement of the outer body (siblings/an `else`); the OUTER
`if`/`else if` has its own trailing `else`/`else if` (merging changes when that `else` fires). An
`else if` outer with NO trailing `else` IS mergeable (`} else if (A) { if (B) {..} }` →
`} else if (A && B) {..}`). A residual `X != null && X instanceof Y` is harmless. **A comment BETWEEN
the two `if`s is usually recoverable** (not an auto-drop): a single-line `//` describing the inner
condition MOVES above the merged `if` (same indent); DROP only a multi-line/block comment or one
documenting the OUTER condition.

**Subagent brace-surgery gotcha:** a subagent can leave a STRAY `}` (cascade `illegal start of type`).
ALWAYS build after structural subagent edits (and verify the file set per General techniques); one bad
file doesn't condemn the batch. **Cheap pre-build check:** per file, `open-Δ = #{(HEAD) - #{(working)`
and `close-Δ` similarly; a correct merge removes exactly one `{` and one `}` per issue, so
`open-Δ == close-Δ ==` the file's issue count. Any mismatch = stray/missing brace, inspect first.
