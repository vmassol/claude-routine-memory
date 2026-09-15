# `java:S1452` — "Remove usage of generic wildcard type"

Recorded in `dropped-issues.md` as one of four **whole-rule drops** (with `S1700`/`S9149`/`S2176`),
on the reason *"zero `private` sites; every one is `public`, `protected` or an interface method, i.e.
a published signature change"*. That reason is correct and it is an objection to the **remediation the
message names** (narrow the signature), not to the finding — so the rule is an **FP-suppression pool**,
the same escape as `S2176`/`S9149`/`S2065`/`S1214`. 35 of 36 open sites shipped in one sweep across all
three repos (platform #6388 28, commons #1977 4, rendering #439 3).

## Why the rule is wrong here

The rule's premise is that a wildcard in a *return* position is useless because the caller cannot
narrow it, and its remediation is to name an invariant type (`List<Animal>`, or a super/subtype).
That presupposes an invariant type **exists**. In XWiki it usually does not: the returned type is
generic in a parameter the API deliberately leaves open.

## The free classifier: is the wildcard FORCED?

Decidable from the flagged line plus the method body — no whole-file reads. Keep the site when one of
these holds, drop it otherwise:

1. **The value comes from an upstream API that is itself wildcard-typed** — `MacroManager#getMacro`
   returns `Macro<?>`, `BaseCollection#getSetValue` builds from `(Collection<?>) prop.getValue()`,
   `StatsService` forwards `XWikiStatsService`'s own `List<?>`. Narrowing needs an unchecked cast.
2. **An implementation returns a subtype-parameterized list**, which is *not assignable* to the
   invariant form — `HibernateDataMigrationManager#getAllMigrations` returns
   `List<HibernateDataMigration>` where the abstract parent declares `List<? extends DataMigration>`.
   This one is a compile error if you "fix" it, i.e. the strongest possible proof.
3. **The wildcard is the declared type of the field/constructor/setter the getter mirrors** —
   `PrepareMailQueueItem#getMessages`, `BoxMacroParameters#getBlockTitle` /
   `setBlockTitle(List<? extends Block>)`, `FilterElementDescriptor#getParameters`.
4. **The type argument is only known at runtime, or differs per element** — `Provider<?>` built for a
   role type carried by the descriptor, `Patch<?>` whose element type is decided by the passed
   `StringSplitter`, `BaseProperty<?>` whose subclass depends on the xproperty, a DML Hibernate
   `Query<?>` that has no result type at all.
5. **The Javadoc already documents the heterogeneity** — `RightsManager`/`XWikiGroupService` say
   *"a List of String … otherwise a List of XWikiDocument"* depending on a `withdetails` flag. The
   comment text is then already written.

**Drop condition (the truthfulness gate, same as `S1186`/`S1214`):** the wildcard is merely stylistic —
the body returns a call whose type argument is *inferred* from the target type, so both forms compile.
`SpaceTreeNode#getChildren` (`AW5-S85W1Yj5qvzeRoRl`) is the example: `return query.execute()` infers
whatever is asked of it, the class is in an `internal` (Revapi-excluded) package, and the honest
answer is that the site could be fixed. Dropped; see `dropped-issues.md`.

## Mechanics

* The fix is a **pure insert** above the declaration (a `//` reason line, then
  `@SuppressWarnings("java:S1452")`), so it is structurally immune to the `Analyze` gate's
  "the issues your own lines carry" rule and to the `checkstyle` metric rules. 68 inserted lines,
  0 removed, across 19 files in three repos.
* **Two keys can sit on one line**: `Map<?, ?>` is reported once per wildcard
  (`StatsService:198`, `XWikiStatsService:123`), so one annotation clears two issues — build the
  accept/PR count by KEY, not by edit.
* Write the reason **per method**, specific to that method, not one generic sentence copied over the
  pool — that is exactly what drew the review question on `S1214`.
