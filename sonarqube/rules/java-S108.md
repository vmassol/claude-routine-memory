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

## What the comment must SAY — a `// TODO:`, not a rationale (review verdict)

**This is the whole review of the 82-site PR, and it inverts the naive fix.** In XWiki a `catch` that
neither rethrows nor logs is *a bug to be fixed later*, not a decision to be documented — so a comment
that only explains why the exception is swallowed reads as blessing it. Vincent: *"any place which has
a catch and doesn't either rethrow an exception, or log a message is a bug that will need to be fixed.
The minimum is to log a warning… please add TODO comments (`// TODO:` prefix) so that we remember."*

Write the TODO line, then the sentence describing today's behaviour:

```java
} catch (Exception e) {
    // TODO: log a warning instead of ignoring this exception.
    // A plugin failing must not prevent the other plugins from being called.
}
```

Two forms, and the split is the same one the shapes below already give you:

- **`// TODO: log a warning instead of ignoring this exception.`** — the default (64 of 78 here): any
  fallback, and any cleanup in a `finally`.
- **`// TODO: change the logic so this case is not signalled by an exception.`** — where the exception
  IS the expected outcome (14 here): a "not found" domain exception used as a signal
  (`XWikiRightNotFoundException`), a missing constructor detected by catching `Throwable`, a null check
  written as a `catch`. Expecting an exception as a normal case is the anti-pattern the reviewer names.

Do **not** add the logging itself in the sweep: it changes behaviour and, in the plugin and rights
loops, output volume on a hot path. Say so in the PR, and flag the consequence — N TODOs become N new
INFO-level `java:S1135` issues, which is precisely the reminder being asked for.

The rule now lives in the plugin OKF at `okf/conventions/code-comments.md`.

## The shapes — they pick the TODO form and the second line

Group by *why the exception is swallowed*. The group gives you both the TODO form above and the
explanation line that follows it (one sentence per group, reused verbatim; 78 sites came out as ~33
sentences):

- **A cleanup call in a `finally`** (`endTransaction`, `statement.close()`, `is.close()`) — *log a
  warning*; "A failure to close the transaction must not hide the original error." The densest shape
  (16 of 82) and the least contestable.
- **A per-element loop over independent handlers** (every `XWikiPluginManager` hook, a `flushCache`
  over each class) — *log a warning*; "A plugin failing must not prevent the other plugins from being
  called."
- **A fallback chain** (a preference/locale/language read whose failure just means the next source is
  tried) — *log a warning*, and say which fallback happens: "The resource is then read from the file
  system below."
- **A domain exception that MEANS "not found here"** — `catch (XWikiRightNotFoundException e) {}` in
  the rights cascade — *change the logic*; "No right found at this level, continue with the next
  check." Same for a missing constructor found by catching `Throwable`, or a null check written as a
  `catch`.
- **An empty non-`catch` block**: an empty `else`/`finally` is pure syntax with no effect — **remove
  it** rather than commenting it (2 of 83, both in `XWiki`). A filter branch (`if (character < 0x20)
  { }` dropping control characters before an XML escape) and an empty `default -> { }` keep a plain
  explanatory comment with **no** TODO: they are not exception handling.

## Drop condition — truthfulness, not safety

Nothing here can break at runtime, so the only reason to drop is **not being able to write a true
sentence** (and with the TODO form above, even that shrinks: a TODO asking for a log is true of any
swallowed exception). One drop in 83: a *test helper* (`SyndEntryDocumentSourceTest#source`) that swallows the
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
