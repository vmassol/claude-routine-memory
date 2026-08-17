# java:S6916 — "replace this `if` with a pattern match guard"

**False positive whenever the enclosing `switch` selects on enum constants.** Java 21 allows a `when`
guard only on a *pattern* case label; `case BLOCKED, BLOCKED_BY_ANCESTOR when cond ->` does not
compile (`: or -> expected`). Confirmed with a five-line `javac 21` throwaway — cheaper than
reasoning about the JLS.

So the rule is only actionable on `case Type t ->` labels. In XWiki the pool so far has been enum
switches, i.e. all drops.
