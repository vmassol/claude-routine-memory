# java:S6126 — replace String concatenation with a text block

> Load only when fixing S6126. Cross-cutting mechanics live in `learnings.md` → *General
> batch-fix techniques*. This is a JUDGMENT/CHURN rule → keep it in its OWN separate PR (Vincent's
> easy/hard split), never bundle into the safe-mechanical batch.

Message: "Replace this String concatenation with Text block." Fires on a multi-line
`"a\n" + "b\n" + "c"` concatenation that a Java text block (`"""..."""`, Java 15+; xwiki-platform
18.x is Java 17 — supported, and text blocks already exist in the tree) can represent. Deep pool
(~90 open), rarely PR-touched, but each site needs byte-identity verification, so it is NOT
mechanical-safe fodder.

**The pool is DOMINATED by TEST files** (expected-output strings passed to
`assertBlocks`/`assertEquals` in rendering/macro `*Test.java`). This is the IDEAL sub-pool: the test
compares the EXACT string, so a byte-wrong conversion FAILS the build — full verification for free.
Concentrate a batch in a few dense test modules (e.g. `xwiki-platform-rendering-macro-include` ~15,
`display-macro` ~8, `rendering-wikimacro-store` ~8) and run them with tests; passing tests prove
byte-identity. PREFER test sites; be far more cautious with production strings (SQL/log/HTML) whose
exact content is NOT asserted anywhere — there a subtle whitespace slip ships silently.

**Conversion rules (get byte-identity EXACTLY right):**
- Opening `"""` stays on the statement line (after `=`/`return`/method-arg), then a newline; content
  lines follow. Indent EVERY content line AND the closing `"""` to the SAME column → Java strips the
  common (incidental) leading whitespace, leaving no leading spaces in the result. Meaningful LEADING
  spaces in the string → indent that line FURTHER than the baseline.
- **Trailing-newline rule (the #1 correctness trap):** if the original ends WITHOUT a trailing `\n`
  (last fragment `"...endDocument"`) put the closing `"""` IMMEDIATELY after the last char
  (`endDocument"""`); if it ends WITH `\n` the closing `"""` goes on ITS OWN line (that newline is the
  trailing `\n`). A blank line in the middle (`"a\n\n" ...`) is just an empty content line.
- A leading fragment with NO `\n` (`"{{groovy}}" + "println...\n"`) MERGES onto the next line
  (`{{groovy}}println...`). Keep escapes (`\t`,`\\`); a lone `"` needs no escaping, never emit `"""`.
- **Remove `// @formatter:off` / `// @formatter:on`** ONLY when the pair wraps SOLELY the converted
  literal (they existed to protect the manual concatenation layout). KEEP them when they wrap a whole
  statement (e.g. a multi-arg `registerWikiMacro(...)` call).

**DROP conditions (expect ~30-40% drops; a drained-pool run netted 25/41 ≈ 61% converted).**
The >120 drops CLUSTER on predictable content — triage these fast: DOCTYPE declarations
(`<!DOCTYPE html PUBLIC "-//W3C//DTD XHTML 1.0 Strict//EN" "...dtd">` ≈ 109 chars → 121 at the usual
12-space text-block indent), Velocity `{{info}}$services.localization.render(...)` / `{{/info}}`
markup rows, `$services.model.resolveObject*('xwiki:...[0]')` reference lines, and any two no-`\n`
fragments the author deliberately split to stay ≤120 (merging them re-breaches). All-whitespace rows
(`"      \n"`) hit BOTH the trailing-whitespace AND >120 rules.
- **A resulting content line > 120 chars** — the #1 drop cause. Long `beginMetaData [[...]...]` event
  lines routinely exceed 120 once un-concatenated onto a single text-block line and cannot be split
  (splitting would insert a newline into the string). DROP.
- **Meaningful TRAILING whitespace on a content line** — text blocks UNCONDITIONALLY strip trailing
  whitespace per line, so a fragment like `"| row | 12 | 13 | 14 \n"` (space before `\n`) CANNOT be
  reproduced → not byte-identical → DROP. (Table/CSV/chart-data test strings are a whole-file drop for
  this reason.)
- A string whose exact content is not test-asserted AND you cannot prove byte-identity by inspection.

Delegate one file per parallel general-purpose subagent (disjoint), give the exact flagged lines, tell
them to process bottom-up and report per-site CONVERTED/DROPPED; then review the diff (grep added lines
>120, check `"""` count = 2×conversions) and let the module's tests be the byte-identity gate. Sonar
attributes the issue to the STATEMENT-START line.
