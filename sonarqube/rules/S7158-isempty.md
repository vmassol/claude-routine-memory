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

- Match `([\w.]+(?:\(\))?(?:\.[\w]+)*)\.length\(\)\s*(==|!=|>)\s*0` and substitute once per flagged
  line (`count=1`). The receiver group handles chained/field receivers: `selection.trim()`,
  `getPrinter().where`, `filterContext.wsGroup`, `printer.where` — all captured correctly.
- **Compound conditions are common and safe**: only the flagged comparison changes, e.g.
  `if (buffer.length() > 0 && buffer.charAt(buffer.length() - 1) == ' ')` →
  `if (!buffer.isEmpty() && buffer.charAt(buffer.length() - 1) == ' ')` — the `buffer.length() - 1`
  (a `- 1`, not `OP 0`) is left untouched by the regex. Also seen: `x == null || x.length() == 0`,
  ternaries, multi-clause `||`.
- **No line-length risk**: `isEmpty()` is shorter than `length() == 0` / `length() > 0`, so edits
  only shrink the line. (The added `!` for the `> 0` case is one char — still net shorter.)
- Edit by line number is safe when the checkout is at/near the scan commit (verify per core
  learnings). Confirm no drift by spot-checking one `issue_snippets` call if the checkout is ahead.

## Pool (as of a mid-2026 run)

~72 project-wide. Spread across many leaf modules (annotation, model-api, query-xwql, rendering
async/macros, rest, url, user, xar, zipexplorer) PLUS ~30 in oldcore. A ~17-module non-oldcore
reactor cleared 35 in one green `-Plegacy,quality` build (~19 min with tests) — enough to hit a
30-fix target on its own. **The remaining ~37 (oldcore-heavy, plus feed-api/search-solr/tools) are
still VALID, safe fixes deferred only for build-ROI (oldcore's huge test suite)** — pick them up on a
future run; they are NOT in `dropped-issues.md` because nothing disqualifies them.

## Verification

Standard `-Plegacy,quality` with tests. No JaCoCo/Revapi impact (no instructions removed, no API
change). A clean build + green tests is expected; a mechanical `isEmpty()` cannot break a test.
