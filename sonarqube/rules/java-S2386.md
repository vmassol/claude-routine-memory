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

## Pool shape (2026-08)

Platform 21, commons 8, rendering 3. Converted 7 + 0 + 3. It is one of the very few rules that is
*present in all three repos* while the mechanical allowlist reads dry.
