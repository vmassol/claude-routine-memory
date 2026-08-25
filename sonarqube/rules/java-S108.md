# `java:S108` — "Nested blocks of code should not be left empty"

**Comment-only remediation, exactly like `java:S1186`** — Sonar's own rule text says *"The rule
ignores code blocks that contain comments unless they are synchronized blocks"*. So the fix is one
`//` line inside the block saying why it is empty, and the unit of judgement is the SHAPE, not the
site. Confirm the exception with `api/rules/show?organization=xwiki&key=java:S108` before batching:
it is the whole reason the rule is cheap.

## The pool

Platform-only (commons and rendering have **zero**), and heavily clustered: 83 issues in 20 files,
61 of them in four oldcore legacy files (`XWiki` 19, `XWikiRightServiceImpl` 16,
`XWikiHibernateStore` 11, `XWikiPluginManager` 10). Nearly all are empty `catch` blocks in
pre-component-era code. 82/83 shipped, ~35 distinct sentences.

## The shapes and what to write

Group by *why the exception is swallowed*; each group needs one sentence, reused verbatim:

- **A cleanup call in a `finally`** (`endTransaction`, `statement.close()`, `is.close()`) —
  `// Ignore: a failure to close the transaction must not hide the original error.` This is the
  densest shape (16 of the 82) and the least contestable.
- **A per-element loop over independent handlers** (every `XWikiPluginManager` hook, a `flushCache`
  over each class) — `// Ignore: a plugin failing must not prevent the other plugins from being
  called.`
- **A domain exception that MEANS "not found here"** — `catch (XWikiRightNotFoundException e) {}` in
  the rights cascade is the model case: `// No right found at this level, continue with the next
  check.` These are the sites where the comment adds real value.
- **A fallback chain** (a preference/locale/language read whose failure just means the next source
  is tried) — say which fallback happens: *"fall back to reading the resource from the file system
  below"*, *"this language source is then simply not taken into account"*.
- **An empty non-`catch` block**: an empty `else`/`finally` is pure syntax with no effect — **remove
  it** rather than commenting it (2 of 83, both in `XWiki`). An empty branch that is really a filter
  (`if (character < 0x20) { }` dropping control characters before an XML escape) keeps its block and
  gets the best comment of the batch.
- **An empty `default -> { }`** of an arrow `switch` — `// The other properties don't contribute to
  the where clause.`

## Drop condition — truthfulness, not safety

Nothing here can break at runtime, so the only reason to drop is **not being able to write a true
sentence**. One drop in 83: a *test helper* (`SyndEntryDocumentSourceTest#source`) that swallows the
exception of the call under test — it may be hiding a failing assertion rather than ignoring an
expected one, and only the test's author knows. Suspect every empty `catch` in `src/test` for the
same reason; `src/main` legacy fallbacks are self-evident from their surroundings.

## Mechanics

- The flagged line is the block OPENER (`} catch (X e) {`), and the next non-blank line closes it —
  but that closing line is not always `}`: it is `} catch (…) {` when a second catch follows
  (chained catches), so assert `strip() == '}' or strip().startswith('} ')`.
- Insert at the closing brace's indent + 4, and check the resulting line against 120 columns: the
  deep-nested `finally` sites in the Hibernate stores sit at 24 spaces, which forced the transaction
  sentence to be shortened. Write the sentence for the DEEPEST site of its group.
- The block sometimes already contains a blank line — replacing the whole span between opener and
  closer (rather than inserting) removes it in the same edit.
- Afterwards, scan every changed file for *any* remaining empty block (opener → only blank lines →
  closer): Sonar under-reports, and a half-commented file is what a reviewer objects to. On this
  pool the scan came back at 0.

## Do NOT turn these into log statements

Tempting for the swallowed exceptions, but it changes behaviour and (in the plugin and rights loops)
output volume — that is a JIRA issue, not a Sonar cleanup. Say so in the PR body; it is the obvious
reviewer question.
