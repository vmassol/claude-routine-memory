# java:S3457 — "printf-style format strings should be used correctly"

**This one rule key bundles five unrelated defects. Triage it by `message`, never by rule** — one
`issues/search` grouped by `message[:60]` splits the pool in a single call and the five shapes have
five different verdicts. (Platform's 41: 18 + 12 + 8 + 2 + 1 → 7 fixed, 34 dropped.)

| Message | Verdict |
|---|---|
| `String contains no format specifiers.` on a `String.format(…)` **with an argument** | **FIX.** The format string uses the SLF4J `{}` placeholder inside a `String.format`, so the argument is silently dropped and the message renders a literal `[{}]`. `{}` → `%s`, same length so no 120-column risk. A real defect, and the intent is unambiguous. |
| `Nth argument is not used.` | **FIX.** Usually the caught exception passed as an extra `String.format` argument instead of being chained — move it to the exception constructor (`new X(String.format(…), e)`), message unchanged. Sometimes just a stray argument to delete. |
| `String contains no format specifiers.` on a `Formatter.format("<literal>")` | **DROP.** There is no cheap equivalent: `Formatter` has no `append`, so removing the call means going through `f.out()`, which throws `IOException`. Platform's whole 9-site cluster is `DatabaseKeywordSearchSource` building HQL this way. |
| `%n should be used in place of \n` | **DROP** (already recorded). A behaviour change on Windows, and every XWiki site's string is asserted or compared. |
| `No need to call "toString()" …` | **Owned by [`okf/conventions/logging.md`](https://github.com/xwiki/xwiki-dev-llm/blob/master/xwiki/okf/conventions/logging.md), read it first.** Mostly a DROP: a log argument is XStream-serialized into the job log, so the eager String is load-bearing. The clean subset is the convention's own "pass the object" list — a `File`, an **enum**, a model reference, `ExtensionId`/`Version`, `String`/`Number`/`Boolean`, a `@Component`, a `Collection`. Anything else (an `Object[]` row, a `URI`, a Hibernate `Session`, a `StringBuilder`, a job `Request`, a `Mail`/`MailConfiguration` whose `toString()` is deliberately narrower) stays. **Cheapest classifier: 10 of platform's 12 drops already carried a call-site comment saying why** — the universal "a comment explains it" drop condition catches most of the pool for free. |
| `Format specifiers should be used instead of string concatenation.` | **DROP on a log call whose raw message is asserted.** `warn("[DEPRECATED] " + message)` → `warn("[DEPRECATED] {}", message)` renders identically but changes `LogEvent.getMessage()`, and `LoggingScriptServiceTest#deprecate` asserts exactly that string. A test asserting the *unformatted* message is the codebase stating the flattened form is the contract — revert rather than adapt the test. This cost one build round; the general lesson is under *A logging fix can be contradicted by a test on the RAW message* in `learnings.md`. |

## Outcome datapoint — the `toString()` half is merged but UNDER REVIEW

`#6200` (7 fixes) **merged**, but Vincent explicitly flagged the two `toString()` removals for a second
opinion before merging: *"@tmortagne could you double check that this is ok please? … I'm merging the PR
but will fix if you think the `toString()` need to be kept (and update the logging best practice)."*
So the "pass a `File` / an enum as the object" reading of `okf/conventions/logging.md` is **provisionally**
accepted, not settled. Two consequences for a future run:

- If those two sites (`FileSystemURLFactory` L204, `DefaultSolrIndexer` L759) have had their `toString()`
  restored on master, that is the reviewer's answer — treat the whole `toString()` shape as a 100% drop
  pool and do not re-remove them. Check the current source before touching this shape again.
- The other four shapes of the rule drew **no comment at all**, so the `{}`-inside-`String.format` and
  unused-argument fixes are clear. Prefer them; the `toString()` half is worth ~2 issues and carries the
  only review risk in the rule.

Verification note: nothing about the two clean shapes is caught by the compiler — they change only
string contents — so the module's own tests are the whole gate, and a test asserting an exception
message is exactly what you want to run.
