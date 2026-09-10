# `java:S2386` — mutable fields should not be `public static`

**OKF-denylisted for the wrong reason, and the reason is a NEW escape axis.** The denylist entry
reads *"make this member `protected`" — reduces the visibility of a public static member → Revapi
`java.field.visibilityReduced`, the same break as `S5993`*. That is true of the fix **the message
suggests**, and the message is not the rule. `S2386` fires on a `public static` member whose *value*
is mutable (array, `Collection`, `Date`, `awt.Point`), so the second compliant outcome — make the
value immutable — clears it with **no declaration change at all**.

Generalises beyond this rule: when a denylist reason objects to a *specific remediation*, check
whether the rule has another one. Sonar's `message` names one fix; the rule description usually
allows several.

## Proving the alternative fix actually clears the issue

Do this before batching (the `S8786` lesson: a fix that does not clear the issue is worthless).
`api/rules/show` did **not** settle it here — S2386's description shows only non-compliant examples.
What settled it in one grep: **find a constant in the scanned source that is already written in the
candidate form and confirm it carries no open issue.**

```
grep -rn --include=*.java -E "public static final (List|Set|Map|Collection)<[^=]*=\s*(Collections\.unmodifiable|List\.of|Set\.of|Map\.of)" <repo>
```

Two hits, both public static collection constants, both unflagged: platform
`PasswordClass.SUPPORTED_ALGORITHMS = List.of(…)` and rendering
`ImageBlockParser.IMAGE_ALIGNMENT = Map.of(…)`. So the `X.of(…)` family is accepted by the analyzer.
`Collections.unmodifiable*` was **not** proven this way (no in-repo precedent) — don't assume it.

## The fix

`Arrays.asList(<literals>)` → `List.of(<literals>)`; `SetUtils.hashSet(…)`/`new HashSet<>(…)` inline
→ `Set.of(…)`. Remove the `java.util.Arrays` / `org.apache.commons.collections4.SetUtils` import when
the swap orphans it (check the whole file — `Arrays.stream` elsewhere keeps it alive; that happened in
1 of 7 platform sites).

## It is a judgement PR, not a mechanical one

`Arrays.asList` is already fixed-size, so `add`/`remove` already threw; the one behaviour difference
is `set(i, v)`, which now throws `UnsupportedOperationException`. A caller mutating a shared public
constant is exactly the defect the rule reports, but it is still a behaviour change on published API
→ its own PR. Also state that `List.of`/`Set.of` reject `null` elements (and `Set.of` duplicates) and
that neither is reachable when every element is a literal or a `static final String`.

## Drop shapes (all decidable from the flagged line plus the lines just below it)

- **Empty collection + `static` block population** (`= new HashMap<>()` then a `static { … put … }`).
  Making it immutable needs a private backing field plus a public unmodifiable view — a refactor, not
  a cleanup. This was the single biggest drop group: platform `SyndEntryDocumentSource` (4),
  `XARDocumentParameters`, `AbstractEntityReferenceResolver`, `XWikiRepositoryModel`; commons
  `VelocityParser` (5), and it is why **commons yielded 0** on this rule.
- **An array** (`public static final String[] X = {…}`) — cannot be made immutable at all.
- **`EnumSet.of(…)`** — `Set.of` changes the concrete type and iteration order; and one of the
  platform sites is literally `= null`.
- **An interface field** — the message changes to *"Move X to a class and lower its visibility"*,
  which is the design change the denylist was right about.
- **`int[]` / a mutable domain object** (commons `ExtensionUtils.STANDARD_DELIMITERS`,
  `FilterEventParameters.EMPTY`).

The free classifier is the **initializer expression on the flagged line**: an inline
`Arrays.asList`/`SetUtils.hashSet`/`List.of`-able literal list converts; anything else drops.

## Outcome

**The judgement half MERGED, uncommented** — rendering [#424](https://github.com/xwiki/xwiki-rendering/pull/424)
landed ~8 h after opening with no review comment at all, alongside the sweep's mechanical commons PR.
So the immutable-value remediation is not merely *accepted by the analyzer*, it is accepted by
reviewers: an `Arrays.asList`/`SetUtils.hashSet` → `List.of`/`Set.of` swap on a read-only public
constant does not read as a backward-compatibility risk to a maintainer. Twelfth denylist rescue to
merge, and it keeps the record at "a rule rescued from a mis-scoped denylist reason merges
uncommented".

Ship it as its own PR anyway: the `set()`-throws difference is a real behaviour change on published
API and belongs in front of a maintainer as one, not buried in a mechanical batch.

## Pool shape (2026-08)

Platform 21, commons 8, rendering 3. Converted 7 + 0 + 3. It is one of the very few rules that is
*present in all three repos* while the mechanical allowlist reads dry.

## When the constant is BUILT rather than listed: the analyzer only reads JDK/Guava factories

`VelocityParser`'s five `public static final Set<String> VELOCITYDIRECTIVE_*` (commons) are the other
shape of this rule: not `Arrays.asList(…)` but `new HashSet<>()` filled by a `static {}` block, with
three of the five *derived* from the other two. Three lessons, in the order they cost a round:

1. **Do not spell the derived sets out as literals.** Writing each of them as an explicit `Set.of(…)`
   duplicates the base sets' elements and `checkstyle:check` fails on `MultipleStringLiterals` —
   13 violations, *after* 213 tests had gone green. Compose instead.
2. **The composition's outer call must be one SonarJava recognises.** Its check is syntactic and its
   list of immutable factories is JDK/Guava only — `Collections.unmodifiable*`, `Set.of`/`List.of`/
   `Map.of`, `ImmutableSet.of`. A third-party call it cannot resolve reads as mutable, so
   `Collectors.toUnmodifiableSet()`, a private helper, or a bare `SetUtils.union(…)` all risk leaving
   the issue open — the `S8786` hazard ("does the remediation actually clear the issue?") applied to
   this rule. Keep `Collections.unmodifiableSet(<whatever computes it>)` as the outermost expression.
3. **Reviewer datapoint (commons #1962): use the repo's own helper inside that wrapper.**
   `tmortagne` asked for commons-collections4's `SetUtils#union` instead of a hand-rolled varargs
   helper — check the module pom first (it *is* a declared dependency there) — which deletes 20 lines
   and keeps the recognised wrapper. Worth saying in the reply that the wrapper looks redundant
   (`SetUtils.union` already returns an unmodifiable `SetView`) and *why* it stays, plus an offer to
   drop it: there is no `public static final` field initialized through `SetUtils`/`ListUtils`
   anywhere in the three repos, so the in-repo-precedent check cannot settle it.
