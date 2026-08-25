# `java:S9355` — "Comments containing Javadoc or HTML tags should use Javadoc syntax"

Fires on a `/* … */` block comment that carries `@see` / `@return` / `@param` / HTML — i.e. someone
wrote Javadoc and lost the second asterisk. **Comment-only, and the safest rule in the corpus after
`S7476`.** Never triaged before 2026-08: platform 9, rendering 2, commons 0.

## Two shapes, two different fixes

1. **A real Javadoc comment missing its asterisk** — `/*\n * @see #getState()\n */` above a field
   (all 8 platform sites are `MailStatus` fields, and the file's FIRST field already uses `/**`,
   which is the giveaway). Fix: `/*` → `/**`. One-character diff, zero risk.
2. **An IDE-generated `(non-Javadoc)` header** on an `@Override` method (`/* (non-Javadoc) @see
   Iface#method(…) */`). Do **not** promote it: it says nothing the `@Override` below it does not,
   and its `@see` points at the very method being overridden. **Delete the comment** — removing the
   tags resolves the issue just as well as adding an asterisk.

A third variant needs a rewrite rather than a swap: a prose comment with a tag buried mid-sentence
(`XWiki#getUserTimeZone`: *"…with this timezone @return the timezone"*). Reflow it into a proper
Javadoc — first sentence ending in a period, blank `*` line, then the `@return` tag on its own line.
That is the only site of the pool where Checkstyle has an opinion, and it passed.

## Drop conditions

None found. The rule cannot fire on anything load-bearing — the comment is already not Javadoc, so
promoting or deleting it changes no generated documentation that anyone depends on today.
