# `java:S1214` — "Move constants defined in this interface to another class or enum"

**OKF-denylisted (with `S115`) as a "cross-module rename, breaking". That is an objection to the
*message's remediation*, not to the rule — so it is an FP-suppression pool, and the rationale
sentence is ALREADY WRITTEN IN THE FLAGGED FILES.** Same escape axis as `S2386` and `S2176`.

Pool on 2026-09-14: **platform 8, commons 3, rendering 3 — 14 issues, all three repos, none claimed
by an open agent PR.** Every site is a published constant holder (`Constants`, `XWikiConstants`,
`WikiMacroConstants`, `RefactoringJobs`, `DocumentEventType`, `FilterStreamConstants`, `Resources`,
`HTMLConstants`, `IWemConstants`, `XWikiWikiModelHandler`, …), so moving the constants out breaks
every extension that references them.

## The lever, and it generalises far beyond this rule

**Checkstyle's `InterfaceIsType` is the same finding as `java:S1214` under another name, and XWiki
already suppresses it — with the reason — in 16 files:**

```java
// Old interface not describing a type, hard to remove for backward-compatibility reasons.
@SuppressWarnings("checkstyle:InterfaceIsType")
```

10 of the 14 flagged sites carry exactly that, so the fix is to add the Sonar key to the annotation
that is already there:

```java
-@SuppressWarnings("checkstyle:InterfaceIsType")
+@SuppressWarnings({"checkstyle:InterfaceIsType", "java:S1214"})
```

The 4 without one (`WikiMacroConstants` + its `@Deprecated` legacy twin `LegacyWikiMacroConstants`,
`IWemConstants`, and the nested `protected interface IBlockTypes` of `InternalWikiScannerContext`)
get a fresh `@SuppressWarnings("java:S1214")` carrying the same sentence, which keeps the codebase
speaking one language about the idiom.

**Generic form of the lever — ask it of every denylisted rule**: *does a linter XWiki already runs
report the same finding under its own name, and has the team already suppressed it?* An existing
`@SuppressWarnings("checkstyle:…")` / `// CHECKSTYLE:OFF` / `@SuppressFBWarnings` on the flagged
declaration is a **decision already taken and already justified in the code** — the Sonar issue is
then not a judgement call at all, it is a missing key. One grep
(`grep -rn 'checkstyle:<RuleName>' --include=*.java`) settles it, costs nothing, and produces the
strongest possible PR body: *"the only thing missing was the Sonar key."*

## Mechanics

* Sonar flags the `interface` declaration line itself, so the class/interface declaration is both
  the site and the guard. For the 10 rewrite sites the `old` is the whole
  `@SuppressWarnings("checkstyle:InterfaceIsType")` line, asserted to occur once per file.
* Use the array form `{"checkstyle:InterfaceIsType", "java:S1214"}` — the no-space form is the more
  common in-repo spelling (`@SuppressWarnings({"checkstyle:ParameterNumber", "checkstyle:JavadocMethod"})`).
* A nested interface needs the 4-space-indented variant; `InternalWikiScannerContext` is a **class**
  whose flagged line is a nested `protected interface`, so a "the flagged line starts with
  `public interface`" assertion would skip it.
* Insert-only / one-line-rewrite, so no Checkstyle metric, no Revapi, no behaviour exposure.

## Drop condition

The interface really does describe a type and merely happens to hold a constant — then the rule is
right and the constant should move. None of the 14 sites was that; each one's own name
(`*Constants`, `Resources`, `RefactoringJobs`, `DocumentEventType`) or its existing Checkstyle
suppression says otherwise. Do not suppress a site whose name gives you nothing to say.

## Outcome

Shipped 2026-09-14 as platform #6379 (8), commons #1976 (3), rendering #438 (3) — the only rule of
that day with a pool in all three repos, on a run where the never-mentioned-rule diff was empty for
the second day running and 24 open agent PRs held 1388 issues.
