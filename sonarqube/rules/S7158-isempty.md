# java:S7158 — "Use isEmpty() to check whether a StringBuilder is empty or not"

A length comparison against zero is replaced by `isEmpty()`. Purely mechanical, single-line,
behaviour-neutral, ZERO dataflow — one of the safest rules there is (on par with S1155).

- `X.length() == 0` → `X.isEmpty()`
- `X.length() > 0` (or `!= 0`) → `!X.isEmpty()`

## Key facts

- **SonarCloud's S7158 flags `String` receivers too, not only `StringBuilder`/`StringBuffer`.** The
  rule message always says "StringBuilder", but issues land on plain `String` locals/fields as well
  (e.g. `sourceSyntax.length() == 0`, `selection.trim().length() == 0`). Do NOT reject a site because
  the receiver turns out to be a `String` — `isEmpty()` is defined on `String` (since Java 6),
  `CharSequence` (default method since Java 15), `StringBuilder` and `StringBuffer`, so the transform
  is correct for every receiver that uses `.length()` (a *method*). XWiki targets Java 17, so the
  `CharSequence.isEmpty()` default is available for the builder receivers. (S1155 is the String-only
  sibling and its pool reads 0 — String `.length()` empty-checks are filed under S7158 here.)
- **Only `.length()` — never `.length`.** The array-length *field* (`arr.length`) has no `isEmpty()`;
  the regex `\.length\(\)` (with parens) excludes it automatically.

## Applying the batch

- **Locate sites by per-file MATCH COUNT, not by line number** — this is drift-proof and needs no
  `issue_snippets` call even when the checkout is days ahead of the scan. Per flagged file: count
  regex matches in the working copy and compare with that file's issue count. Equal ⇒ transform
  EVERY match in the file (that is exactly the flagged set). Unequal ⇒ inspect (causes below).
  Across 114 sites in 3 repos this matched on all but 2 files.
- Match `((?:[A-Za-z_$][\w$]*(?:\(\s*\))?)(?:\.[A-Za-z_$][\w$]*(?:\(\s*\))?)*)\.length\(\)\s*(==|!=|>)\s*0`
  → `('' if op=='==' else '!') + receiver + '.isEmpty()'`. The receiver group handles chained/field
  receivers: `selection.trim()`, `getPrinter().where`, `this.path`, `dependency.getSystemPath()`.
- **Count MATCHES, not LINES** — two issues on ONE line is common:
  `if (stb.length() > 0 && (resolvePath.length() == 0 || …))` is 2 issues / 1 line. A `re.sub` over
  the line fixes both at once (no stale-`old` stacking problem).
- **Two receiver shapes the chain regex misses** — both safe, both surface as a count shortfall;
  handle them as explicit literal (assert-unique) replacements:
  - cast-parenthesized: `((StringBuffer) getStackParameter(LIST_STYLES)).length() == 0`
    → `((StringBuffer) getStackParameter(LIST_STYLES)).isEmpty()`
  - redundant parens around the call: `if ((number.length()) == 0)` → `if (number.isEmpty())`
- Also worth a pre-scan: Yoda form (`0 == x.length()`) — not seen in practice, but the regex would
  silently miss it, and the count check is what catches it.
- **Compound conditions are common and safe**: only the flagged comparison changes, e.g.
  `if (buffer.length() > 0 && buffer.charAt(buffer.length() - 1) == ' ')` →
  `if (!buffer.isEmpty() && buffer.charAt(buffer.length() - 1) == ' ')` — the `buffer.length() - 1`
  (a `- 1`, not `OP 0`) is left untouched by the regex. Also seen: `x == null || x.length() == 0`,
  ternaries (`x.length() > 0 ? a : b` → `!x.isEmpty() ? a : b`), multi-clause `||`.
- **No line-length risk**: `isEmpty()` is shorter than `length() == 0` / `length() > 0`, so edits
  only shrink the line (confirmed: 0 breaches over 114 sites). The added `!` is one char — still net
  shorter — and `!x.isEmpty()` is a unary, so it never needs parens in `&&`/`||`/`if`/ternary/`return`.

## Pool

**Drained in all three repos** by one multi-repo run: platform 37 (the oldcore-heavy remainder),
commons 37, rendering 40 — 114 issues, **zero drops** (nothing about this rule disqualifies a site).
It will slowly REGENERATE as new code lands; re-query rather than assuming it is still empty.
Where it clusters when it comes back: commons `xml`/`extension-api`/`repository-api`; rendering
`wikimodel` whitespace filters + `syntax-xwiki20` printers; platform oldcore.

## Verification

Standard `-Plegacy,quality` with tests. No JaCoCo/Revapi impact (no instructions removed, no API
change). A clean build + green tests is expected; a mechanical `isEmpty()` cannot break a test.
