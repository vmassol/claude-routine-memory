# `java:S1130` — "Remove the declaration of thrown exception X, as it cannot be thrown"

The OKF documents the rule in `dead-code-rules` and this repo long recorded platform's remainder as
*"the permanent `src/main` residue"*. **That is a visibility split, not a residue.** Bucketing the 58
open platform issues by the modifier on the declaration gave **19 `private` `src/main` sites, all
fresh**, and all 19 shipped green. Re-bucket before believing any "residue" line about this rule.

- `private` → clean: every caller is in the same compilation unit, so `javac` is the whole
  verification (`src/main` is irrelevant).
- `protected` / `public` / package-private → drop: removing a checked exception from the signature is
  a source break for any caller that catches it.

## Bucket on the DECLARATION line, not the flagged line

Sonar attributes the issue to the line holding the `throws`, which for XWiki's brace style is very
often a **continuation** line (`        throws WikiManagerException, ComponentLookupException`) with
no modifier on it. A naive `'private' in flaggedLine` test buckets those as package-private and
silently drops the best sites — walk back to the nearest line containing
`public|protected|private` first. (Same trap on `java:S1172`, whose issues sit on a parameter line.)

## The fix is three edits, not one

1. Remove the exception from the `throws` list. When it was the only one and it sat alone on a
   continuation line, the line disappears and the declaration merges back into the previous line —
   assert the merged line is ≤120.
2. **Remove the matching `@throws` Javadoc tag** (7 of 19 sites had one). Find it by walking up from
   the declaration over annotations into the `/** … */` block; XWiki's tags are one-liners.
3. **Remove the import the change orphans** (4 of 19), or the next scan reports a fresh `java:S1128`.

## The cascade: an enclosing `catch` can become unreachable

Java rejects a `catch` for a checked exception the `try` can no longer throw. Narrowing a private
method therefore breaks its *callers*, in the same file, at compile time. Pre-check for free before
the build: per site, grep the file for `catch (…<Exc>…)` and, where one exists, check whether any
*other* call inside that `try` still declares it.

It fired once in 19: `DefaultNotificationParametersFactory#createNotificationParameters` wrapped
`useUserPreferences` / `dontUseUserPreferences` in `try { … } catch (EventStreamException e) { throw
new NotificationException(…, e); }`, and the sibling only declares `NotificationException` — which
the enclosing method already declares. So the whole `try` went and the body was dedented; the
exception propagates exactly as before. A `catch (Exception …)` is never affected (it also catches
`RuntimeException`), and a `catch` around a `try` that holds other throwers is fine.

## Outcome

**All 19 sites merged (`xwiki/xwiki-platform#6236`), within hours, with no review recorded and no
comment** — the ninth "the recorded drop condition was a proxy" rescue to land that way. Lead the PR
body with the two things a reviewer would otherwise ask: that every site is `private` so `javac` is
the verification, and that the non-`private` half is deliberately left open.

## Regeneration

Sonar reports only the frontier of the call graph, so narrowing these sites re-flags their private
callers on the next scan. Re-query this rule next run in the very files this PR touched.
