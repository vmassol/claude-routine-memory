# `java:S2629` — "Invoke method(s) only conditionally"

OKF-denylisted, and rightly: the corpus's own `dropped-issues.md` entry records a **withdrawn PR**
(`xwiki-commons#1888`) that deleted an eager `toString()` on the "SLF4J calls it itself" reasoning.
In XWiki a log argument is stored as an object and XStream-serialised into the job log, so
rewriting the call changes what is persisted — `okf/conventions/logging.md` is the authority.

**But the entry's conclusion ("the remaining sites all need an `isXxxEnabled()` guard, which is a
judgement call") is false for a large, free subset — and the classifier is one token on the flagged
line: THE LOG LEVEL.**

## The free classifier: `warn` / `error` are always enabled

`okf/conventions/logging.md` states it outright: *"`warn` and `error` are always enabled with the
default configuration"*. So on a `warn`/`error` call an `isWarnEnabled()` guard **can never skip
anything** — there is nothing to defer, and the rule's premise (the level may be off) is simply
false. That is a complete, checkable false-positive argument, identical for every such site, and it
needs no per-site dataflow reasoning at all:

```java
    // warn/error are always enabled in XWiki's default logging configuration, so guarding this call
    // could never skip the evaluation of its arguments.
    @SuppressWarnings("java:S2629")
    protected void logWarning(String explanation, Throwable e)
```

`@SuppressWarnings("java:S2629")` on a **method** is already the established in-repo resolution —
`JGroupsNetworkAdapter`, `XWikiHibernateBaseStore`, `MailSenderPlugin`, `WorkspacesMigration`
(platform), `AbstractInstallPlanJob`, `DefaultJobProgress` (commons). Cite them; that precedent is
what makes the batch uncontroversial.

Measured 2026-09-14 over the 37 unclaimed sites: **17 shipped** (platform 10, commons 7), the rest
are `debug`/`info`.

## Mechanics

* The annotation goes on the **enclosing method**, so one annotation clears every S2629 key in it —
  `RightsFilterListener#cancel` held 3 and `FailingTestDebuggingTestExecutionListener#executionFinished`
  held 6. Count by key, not by edit.
* Put the `//` reason immediately above the `@SuppressWarnings`, after any `@Override`. That is the
  form the `checkstyle:InterfaceIsType` suppressions use and it survived review.
* Pure insert above a declaration ⇒ it cannot inherit a pre-existing finding, so the `Quality /
  Analyze` collision pre-check is trivially clean.
* Locate the enclosing declaration by walking up from the flagged line to the nearest `{` alone on
  its line, then up over annotations/Javadoc — Sonar flags the **log call**, not the method.

## Two site-specific reasons worth reusing

* **The flattened message is the contract.** `LoggingScriptService#deprecate` concatenates on
  purpose: `LoggingScriptServiceTest#deprecate` asserts `LogEvent#getMessage()` equals the
  concatenated string. (Same fact the corpus records as "a logging fix can be contradicted by a test
  asserting the RAW log message" — here it is the *suppression's* justification rather than a drop.)
* **Running the call IS the point.** `FailingTestDebuggingTestExecutionListener#executionFinished`
  runs `top`/`lsof`/`docker ps` in order to log their output, and the block already only executes
  under `isInCI()`. `dropped-issues.md` had recorded exactly that sentence as the reason to DROP the
  6 issues — which is the recurring lesson: *a drop reason that describes a deliberate idiom is the
  finished suppression comment.*

## What stays open

The `debug`/`info` sites. There a level guard really would skip work, so whether to add one is a
judgement about the surrounding method, not a false positive — keys are in `dropped-issues.md`.
Do not suppress them with a hand-waved reason; the gate on this pool is that the comment must be
*true and checkable*.

## Outcome

Shipped 2026-09-14 as platform #6379 (10 keys over 8 methods) and commons #1976 (7 keys over 2
methods), alongside the `java:S1214` half of the same sweep. 21 `debug`/`info` keys stay open and
are listed in `dropped-issues.md`.
