# SonarQube fix — learnings

Generic, reusable playbook for the "fix one (or a batch of) SonarQube issues in xwiki-platform"
routine. Keep this compact and generic: record techniques and gotchas, NOT run history. When you
learn something, merge it into the right section and trim — don't append dated anecdotes or PR logs.

## Picking a target rule (find phase)

- Get the rule distribution cheaply FIRST (no issue bodies), restricted to a mechanical allowlist:
  `.../api/issues/search?...&issueStatuses=OPEN&severities=BLOCKER,CRITICAL&rules=java:S1192,java:S5361,java:S2093,java:S1481,java:S1854,java:S2147&facets=rules&ps=1`
  The `rules` facet IGNORES the `rules=` filter and returns the WHOLE project rule distribution — one
  call to see the entire landscape; for an exact per-rule count read the response `total`, not a facet.
- **Counts shift every run as PRs land, and clean rules get EXHAUSTED then slowly REGENERATE — always
  re-query, never assume a rule still has convertible issues.** If a rule's remaining issues are all
  the non-convertible residue, pivot to another rule (skill rule: "if a fix is hard, drop it and pick
  another"). `java:S1192` (~12) and `java:S1488` (~12, inline immediately-returned local — pure,
  no-dataflow, in practice zero-drop) are the reliable small ZERO-PR perennials: when EVERY other
  family (syntax S1128/S1197/S1116/S1161, simplification S1125/S1858/S2864/S1612, unused
  S1068/S1481/S1854, S1066, and even the test rules S5786/S5785) is simultaneously drained-to-zero OR
  already covered by open PRs — a common concurrent-session state — MIX S1488 + S1192 into ONE wide
  reactor to reach the 20-50 target (a clean two-rule PR). Their small pools each sit below 20 alone,
  so neither is a standalone batch; together they clear it.
- **The BLOCKER/CRITICAL mechanical pool is frequently exhausted** — S1192 tiny (see feasibility
  check), S2093 often ALL residue. When it is, drop the severity filter: there is a deep MAJOR-severity
  clean pool for single-issue fixes. Reliable MAJOR winners (re-query counts): `java:S1155`
  (`size()>0`/`==0` → `!isEmpty()`/`isEmpty()`), `java:S1068`/`java:S1481`/`java:S1854` (unused
  field/var, dead store), `java:S2864` (iterate `entrySet()` not `keySet()`), `java:S1858` (pointless
  `toString()` on a String), `java:S1612` (lambda → method reference), `java:S1488` (inline
  return-of-temp), `java:S1125` (redundant boolean literal), `java:S1066` (merge collapsible nested
  `if` — a DEEP oldcore pool, own section below), `java:S2147` (combine identical `catch` bodies) and
  `java:S3626` (remove a trailing redundant `return`/`continue`) — both structural, clean, see "Other
  clean rules". Starting BLOCKER/CRITICAL is the skill's
  guidance but is not a hard gate — a clean MAJOR fix beats forcing a risky higher-severity one.
- **A recent open agent PR can drain a WHOLE rule family, not just single issues.** Before committing
  to a rule, read the open `llm-agent` PRs' titles/bodies — if one already batched e.g. S1488/S1125/
  S1612/S2864/S1858, those issues are still OPEN in Sonar (PR unmerged) but OFF-LIMITS; pivot to an
  untouched family. BUT a per-MODULE batch PR only claims the files it touched: if a PR fixed rule X in
  ONLY one module (e.g. S5786 in model-api), the SAME rule X in OTHER modules is fully fair game — no file
  overlap. Scope the off-limits check by (rule + module), not by rule alone. **SEVERAL concurrent
  sessions can saturate the single "best" rule at once** — e.g. 4 open S5786 PRs covering oldcore,
  model-api + ~15 modules in one day. When the rule you planned already has multiple open PRs, don't try
  to thread the gaps; PIVOT to a rule family with ZERO open PRs. `java:S1066` (merge nested `if`, deep
  oldcore pool) is a reliable such pivot — structural, so the S5786/syntax cleanup waves never touch it.
  The safest untouched fodder is the **pure-syntax/annotation group** (own section
  below): `java:S1128` (unused import), `java:S1197` (array designator), `java:S1116` (empty statement),
  `java:S1161` (missing `@Override`) — zero-dataflow, even safer than the simplification rules, and they
  regenerate. **When EVERY small pool (syntax + simplification + unused + S1066 + S1192 + S2093 + test
  rules) is simultaneously below 20**, fall back to `java:S6201` (pattern matching for instanceof,
  hundreds open — own section below), the deepest untouched pool.
- **Denylist — skip these** (bad ROI / risky / not one-liners): `java:S3776` (cognitive complexity),
  `java:S3252`/`java:S1845` (API/backward-compat), `java:S1186` (empty methods), `java:S115` (naming),
  `java:S2447` (null from Boolean method — in XWiki *script services* null is a deliberate
  "check getLastError()" contract; changing it alters behavior), `java:S1214` (constants-in-interface,
  cross-module), `java:S1113` (finalize → needs Cleaner/PhantomRef), `java:S1215`
  (`System.gc()` — the enclosing method may be a deliberately-exposed API, e.g. `$xwiki.gc()` in
  Velocity), `java:S2696` (static field written from instance method — usually lazy-init needing
  synchronization), `java:S2157` (add `clone()`), `java:S5845` (assert dissimilar types — erasure can
  make the assertion actually correct). Verify before "fixing" any of these.

## java:S1192 — define a constant for a duplicated literal (the go-to)

The most reliable clean fix, and the target for Vincent's "fix 20-50 in one PR" override. Query
`&rules=java:S1192&ps=500`, group by `component`/module. **Feasibility check FIRST:** the S1192 OPEN
pool fluctuates and is sometimes small (seen as low as 13 total project-wide). If it holds < 20, the
20-50 batch is IMPOSSIBLE — do NOT pick S1192; instead batch a MIX of the pure-simplification rules
in one module (see that section) — the easiest way to satisfy the 20-50 override. Don't do a sub-20
S1192 batch to "sort of" satisfy it. If you DO have enough S1192, two ways to hit 20-50: (a) combine several
files across a few SMALL leaf modules in one reactor build (cheapest per-file); or (b) when the rule's
TOTAL open pool is small (e.g. ~33) but one module already holds ≥20 (oldcore often does — 15-ish files
of 1-6 dups each), fix that ONE module for a single-module build — simpler, and it alone clears the
target. Note oldcore fixes span heterogeneous files (1-6 dups each), not one fat file.

**Fix mechanic:** add `private static final String NAME = "literal";` and replace EVERY occurrence in
that ONE file. The rule fires at ≥3 occurrences, so leaving ≤2 copies also resolves it.

**Count-verify each literal (line numbers drift — never trust Sonar's line blindly):** count the
quote-bounded token, `content.count('"admin"')` — quote-bounding avoids substring hits (`"admin"` ≠
`"administrator"`, `"login"` ≠ `"loginerror"`). When the count != Sonar's stated N, **diagnose before
dropping**:
- **Excess is in COMMENTS/javadoc** (whole token in prose, e.g. `status "Paused"`, or a `"..."` inside
  a `* #error(...)` javadoc line) → FIXABLE: replace only on non-comment lines (skip lines whose
  `lstrip` starts with `*` / `//` / `/*`) and assert the CODE-occurrence count == N.
- **Substring/fragment mismatch** (grep==0 because Sonar counts a concatenation fragment that isn't a
  standalone `"..."` token; or grep>N because your token is a substring of a longer literal Sonar
  doesn't count) → NOT fixable, DROP that literal.
- **STALE issue** (the literal is absent from the file entirely — grep==0 everywhere, not just a
  drifted line): already fixed on master but not yet rescanned by Sonar. DROP it; you can't fix what
  isn't there. Always confirm each literal actually exists at ~N occurrences BEFORE planning edits.

Aim for a few more than the target so the batch still clears after these drops.

**Scripting the edits (per-line Python, not sed):** replace on original content asserting each count,
THEN insert the constant block — if you insert first, the replace pass rewrites your own new
declarations (`ADMIN = ADMIN`). Delete the helper script before committing. After the batch, run
`git diff | grep '^+' | awk '{if(length>120)print}'` — a constant NAME longer than the quoted literal
(especially a fully-qualified `XWiki.SYSTEM_SPACE` swapped for `"XWiki"`) can push a line past
checkstyle's 120-char limit; rewrap those lines before building, or the build fails.

**Where to declare the constant:**
- **Forward-reference gotcha:** Java forbids referencing a static field by simple name *before* its
  textual declaration inside a static-field initializer (incl. static array/collection initializers,
  e.g. `static final String[][] X = { {CONST, ...} }`) — an "illegal forward reference" compile error.
  So if the literal appears in a static initializer, declare the constant ABOVE it. If it's used only
  in method bodies (the common case), order is irrelevant.
- XWiki checkstyle does NOT enforce public-before-private (or private-before-protected) field order,
  so a private constant may sit above public fields / a `protected` array. But when
  `DeclarationOrder` DOES bite in a class, place the constant block after public fields and before the
  first private field that uses it (e.g. near `LOGGER`).
- Consecutive `private static final String` decls with no blank line between them are fine; private
  constants need no javadoc.
- The **"use already-defined constant" variant** is FIXABLE when the named constant is declared
  *before* the duplicating line — common winners: method-body usages (constant near the class top) and
  a public constant used a few lines below its decl (`SYSTEM_SPACE`, `DEFAULT_ENCODING`,
  `DEFAULT_VALUE`, `EMAIL_CHECKED_PROPERTY`). UNFIXABLE only when the constant sits *after* the line —
  typically a static field initializer near the top referencing a constant declared lower (forward
  reference); skip those. Also DROP when the value match is COINCIDENTAL and the semantics differ (a
  `DefaultPluginName="package"` constant vs. an XML root-element literal `"package"`) — reuse there is
  misleading and draws reviewer pushback.

**Reviewer preferences (apply pre-emptively to avoid churn):**
1. A literal used ONLY in a log-message concatenation (`LOGGER.x(PREFIX + var + " ...")`) — do NOT
   introduce a constant; convert to slf4j parameterized syntax `LOGGER.x("Prefix [{}] ...", var)`
   (bracket the placeholder — XWiki convention). Eliminates the duplicate entirely. If the resulting
   message string is itself duplicated across ≥2 calls, extract ONE constant for the WHOLE message
   (reviewers flag even a 2× duplicate below Sonar's ≥3 threshold).
2. If a literal is a **domain property/field name** (an XWiki class property), don't make a local
   constant — reuse/add a **public** constant on the owning class (the `*DocumentInitializer`; grep its
   `add*Field("...")` calls to identify which literals are its properties; reuse an existing public
   constant if one holds that value; the `"XWiki"` system-space literal already lives at
   `com.xpn.xwiki.XWiki.SYSTEM_SPACE`). **BUT keep it a LOCAL private constant when** (i) the owning
   constant is `private` in an `internal` package (making it public to reuse exposes internal API), or
   (ii) the fully-qualified reference would blow the 120-char line limit — a `LongClass.FIELD` ref adds
   ~40 chars. A class often already has a local constant despite the initializer's public one, so local
   is an accepted pattern; local names are usually SHORTER than the quoted literal so lines shrink.
3. **Any newly-public API (incl. a field widened private→public) needs an `@since` tag** or it's
   flagged. Format = reactor `project.version` with `-SNAPSHOT`→`RC1` (grep `pom.xml` `<version>`,
   e.g. `18.6.0-SNAPSHOT` → `@since 18.6.0RC1`; confirm by grepping recent `@since 18.` tags). Last
   javadoc line, after a blank ` *` separator.

**Checkstyle-excludes trap:** legacy files rich in S1192 are often excluded from Checkstyle entirely
(file-level `<excludes>` in the module pom's `maven-checkstyle-plugin` `default` execution) — that
exclusion is *why* the duplicates accumulated (`MultipleStringLiterals` never ran). Sonar scans them
regardless, so the fix is valid. A reviewer may ask to "update the excludes accordingly" (un-exclude
the file) — **do NOT attempt it in an S1192 PR**: un-excluding enables the FULL ruleset and surfaces
massive unrelated legacy debt (missing javadoc, complexity, declaration order, and more
`MultipleStringLiterals` at Checkstyle's stricter ≥2 threshold). Reply with the concrete violation
count and offer a separate cleanup PR. (To measure: remove the file from `<excludes>` and run the
normal `install` build — NOT `mvn checkstyle:check` from CLI, which uses the default sun_checks config,
not XWiki's, and falsely passes.)

**Other caveats:** a class-level `@SuppressWarnings("checkstyle:MultipleStringLiterals")` suppresses
only the *checkstyle* check, NOT Sonar's S1192 — still fixable. Skip a second S1192 on the SAME line
for a different literal (fix only the reported one).

## Pure-simplification rules — the best batch fodder (no dataflow check)

`java:S1125` (redundant boolean literal: `x == true`/`== false` → `x`/`!x`; ALSO the ternary shapes
`cond ? x : true` → `!cond || x`, `cond ? x : false` → `cond && x`, `cond ? true : y` → `cond || y`,
`cond ? false : y` → `!cond && y` — the operand can be a boxed `Boolean` that autounboxes, still fine),
`java:S1488` (inline an immediately-returned local),
`java:S1858` (drop `toString()` on a value already a `String` — TRUST the rule, it only fires on String
receivers), `java:S2864` (`keySet()`+`get(k)` → `entrySet()`; prefer `values().forEach(v -> ...)` when
the key is unused, else the `entrySet()` enhanced-for `for (Map.Entry<K,V> e : m.entrySet())` — required
whenever the key IS used or the body throws a checked exception / uses `continue`/`break`/a mutated
outer local; `Map.Entry` needs no import), `java:S1612` (`x -> obj.foo(x)` → `obj::foo`; also fires on
block-body `() -> { obj.foo(); }`, constructor `s -> new Foo(s)` → `Foo::new`, `x -> x instanceof Foo`
→ `Foo.class::isInstance`, enum `v -> v.name()` → `Enum::name` (java.lang, no import), qualified super
`() -> Outer.super.foo()` → `Outer.super::foo`; overloaded
targets and `assertThrows(..., obj::method)` clusters in tests are fine). **Import gotcha (build-breaker):**
a method ref names its target TYPE (`Foo::getX`), which the lambda never needed imported — if that type is
a NESTED class (`LiveDataQuery.SortEntry` → `SortEntry::getProperty`) or the stream's element type and
isn't already imported, the build fails `cannot find symbol`; add the import at the right alphabetical
slot. (`Type.class::isInstance`/`::cast` from an `instanceof`/cast lambda need NO new import — the type was
already referenced.) These are behaviour-preserving
with NO use-verification, unlike the removal rules `java:S1481`/`java:S1854`/
`java:S1068` (own dedicated section below — a deeper pool and great batch fodder). For a large
batch, prefer the simplification rules. Oldcore OFTEN holds 40-90 of them (one single-module build
clears a 20-50 mix), but density is NOT guaranteed — some runs the pool is spread thin across many
small leaf modules with oldcore near-zero. **Thin-pool fallback:** query the mix of simplification
rules project-wide, group by module, pick a cluster of ~10 highest-density fast leaf modules (3-5
issues each) totalling 20-50, and build them ALL in ONE reactor `-pl m1,m2,...` (exit 0, a few
minutes) — avoid the heavy modules (feed-api ~5 min, oldcore). Cleanly satisfies the 20-50 override
even with no single dense module.

**`Edit` replace_all indentation gotcha:** the SAME pattern (e.g. `if (x == true) {`) recurs at
DIFFERENT nesting depths in one file; a single `replace_all` matches only the exact indentation you
typed and SILENTLY leaves the others. After any batch replace, grep for the residual pattern
(`git diff --name-only | xargs grep -n '== true\|== false'`) and fix stragglers with more context.

## Pure-syntax/annotation group — S1128 / S1197 / S1116 / S1161 (zero-dataflow, safest batch fodder)

Even safer than the simplification rules (no use/side-effect verification at all) and deep,
regenerating pools. A single batch across ~20 leaf modules (one reactor) cleanly satisfies
the 20-50 override; build ~13 min with oldcore in the set. Exclude a heavy single-issue module (e.g.
feed-api ~5 min for 1 issue) — fix "almost all", not literally all, for ROI. Apply in one
atomic Python script keyed BY LINE NUMBER (the repo is at the exact master commit Sonar scanned, so
lines don't drift); assert each target line's shape before editing and write nothing if any assert
fails; process each file's edits mapping over ORIGINAL indices (or insert bottom-up per file) so an
edit doesn't shift later ones.

- **`java:S1128` (unused import):** delete the flagged `import ...;` line (assert it starts with
  `import` and ends `;`). TRUST Sonar — it already accounts for javadoc `{@link}` refs (it will keep an
  import used only in a `{@link}`). A legacy *delegating subclass* can legitimately have 10-14 unused
  imports (copied wholesale from its parent, whose members it inherits) — all removable. Checkstyle
  `UnusedImports` + compile is the real gate.
- **`java:S1197` (array designator):** `TYPE NAME[]` → `TYPE[] NAME`. **Regex gotcha (always review the
  diff):** `\b(\w+)\s+(\w+)\[\]` WRONGLY grabs a return type already in `[]` form when a modifier
  precedes it — `protected byte[] createZipFile(... docs[] ...)` becomes `protected[] byte ...`. Fix
  with a negative lookahead `\b(\w+)(\s+)(\w+)\[\](?!\s*\w)` → `\1[]\2\3` (count=1): a `[]` followed by
  ` identifier` is an already-correct `Type[] name`, skip it; only a `[]` followed by `,`/`)`/`=`/`;`/`
  {` is a real `name[]` to convert. Handles params, locals and `int x[] = {...}`.
- **`java:S1116` (empty statement):** three shapes — a lone `;` line (delete the whole line); a trailing
  `;;` (strip ONE `;`); and `};` where `}` closes a block/enum/nested-class/method (strip the trailing
  `;`, keep `}`). NEVER strip the `;` from `new Foo(){...};` / `Type x = ... {...};` — that terminates a
  declaration/expression statement and is REQUIRED (Sonar won't flag those). In an anonymous-class field
  init the flagged one is the INNER method-body `};` (redundant); the OUTER field-terminator `};` is NOT
  flagged — distinguish by indentation / exact line number, so line-number-keyed editing is safe.
- **`java:S1161` (missing `@Override`):** deep pool (~66 seen), often CONCENTRATED in a few TEST files
  (anonymous-class methods overriding an interface/abstract, e.g. `new Runnable(){ void run(){...} }`)
  — so 2-3 dense files can be a whole 50-issue batch, but it also spreads 1-per-module as a great
  additive add-on to an unused-code batch. Fix = insert a line `<indent>@Override` (matching the
  flagged signature's leading whitespace) directly ABOVE each flagged method-signature line; purely
  additive, no dataflow. TRUST Sonar (the method genuinely overrides). Assert the flagged line contains
  `(` and is not already `@Override` and its predecessor line isn't `@Override`; insert bottom-up per
  file. When the preceding line is ANOTHER annotation (`@BeforeEach`, `@Unstable`, …), inserting above
  the signature puts `@Override` between it and the signature — fine, annotation order is irrelevant.
  An INTERFACE method that redeclares a super-interface method legitimately takes `@Override` (Java
  6+), so a target inside an interface is valid, not a bug. Behaviour-preserving — the single safest
  rule in this group.

## java:S1066 — merge collapsible nested `if` (structural; the reliable pivot when all else is drained)

The single most reliable batch source when the syntax / simplification / unused-code / S5785 / S5786
pools are ALL simultaneously drained-or-saturated (concurrent sessions frequently leave every one of
them near-zero or with multiple open PRs at once) — S1066 is STRUCTURAL, so those syntax/test cleanup
waves never touch it, and it regenerates. **Density is NOT reliably oldcore-concentrated** — sometimes
~70 in oldcore alone (one dense module = a whole 40-issue batch in ONE build), but other runs it is
THIN-SPREAD (e.g. 31 total across 16 modules, only ~4 in oldcore, no dense module). Then a WIDE
~17-module reactor still clears the whole rule in ONE `install -DskipTests` build (~10-15 min, dominated
by oldcore/packager/model-api). Re-query the module distribution each run; don't assume a single-module
batch. Sonar flags the INNER `if`. Fix: `if (A) { if (B) { BODY } }` → `if (A && B) { BODY }`
— merge with `&&`, delete the inner `if` line, DEDENT the body by 4, and remove ONE of the two trailing
braces. Wrap an operand containing top-level `||` in parens. NOT a pure line-keyed edit (needs reindent
+ brace surgery), so DELEGATE the reading/editing to subagents that work each file bottom-up — for a
wide spread, split the files across SEVERAL parallel subagents (e.g. 4 groups of ~7 files); disjoint
files never conflict, so parallel is safe and much faster than one subagent. **A triple-nested
`if (A){if(B){if(C){...}}}` collapses to `if (A && B && C)` and resolves TWO Sonar keys in one merge** —
so the fixed-ISSUE count can exceed the edited-SITE count; build the accept list by KEY (all-open-S1066
minus the dropped `(basename,line)` pairs), not by counting edits.
**Fixable rate is HIGH (~75%) once you recover comment-between sites.** The standard DROPs are: the merged condition would exceed 120 chars AND can't be cleanly two-line-wrapped
(+4 continuation indent); the outer is an `else if`; or the inner `if` is NOT the sole statement of the
outer body (sibling statements / an `else`) — merging any of these changes semantics. A residual
redundant `X != null && X instanceof Y` is harmless — leave it.
**A comment BETWEEN the outer and inner `if` is USUALLY recoverable, not an automatic drop:** if it is a
single-line `//` comment describing the inner condition/body, MOVE it directly above the merged
`if (A && B)` (same indent) and merge — the comment still describes the same logic, so it's clean and
NOT review churn. This roughly doubled the fix rate (14→25). DROP only when the comment is a multi-line
`//`/block comment, or clearly documents the OUTER condition rather than the inner.

**Subagent brace-surgery gotcha (generic):** when a subagent removes a brace level across many sites, a
single site can be left with a STRAY extra `}` (compiles-cascade error `illegal start of type`). The
`install` build catches it immediately; the fix is deleting the one leftover `}` and rebuilding. Lesson:
ALWAYS build after a subagent does structural (brace-removing) edits — never trust the subagent's
self-report of success; and one bad file among 24 does not condemn the batch (the other 23 compiled).
**Cheap pre-build stray-brace check (run across ALL edited files at once before building):** for each
file compute `open-Δ = #{(HEAD) - #{(working)` and `close-Δ = #}(HEAD) - #}(working)`. A correct merge
removes exactly one `{` and one `}` per issue, so for every file `open-Δ == close-Δ` AND both equal that
file's S1066 issue count (2 for a two-issue file or a resolved triple-nest). Any file where the two
deltas differ has a stray/missing brace — inspect before spending the build. (Also grep the diff for
added lines >120 chars — a wrapped merged condition can breach the checkstyle limit.)

## java:S6201 — pattern matching for instanceof (the DEEPEST clean pool; go-to when all small pools drained)

By far the largest clean pool (seen 460-566 open, ~90-143 in oldcore ALONE) and it barely regenerates
down — so it is the reliable batch source when the small mechanical rules (S1066/S1068/S1192/S2093/
syntax/simplification/test-rules) are ALL simultaneously drained below 20 (the common concurrent-session
state). Requires Java 16+; xwiki-platform 18.x is Java 17 and already uses `instanceof` patterns, so it compiles.
Message: "Replace this instanceof check and cast with 'instanceof Foo foo'". Fix = bind a pattern
variable and delete the redundant cast:
- Positive guard: `if (x instanceof Foo) { ((Foo) x).m(); }` → `if (x instanceof Foo foo) { foo.m(); }`
- Compound `&&` (pattern var scopes rightward + into the block): `x instanceof Foo && ((Foo) x).m()` →
  `x instanceof Foo foo && foo.m()`
- Ternary/return: `return x instanceof Foo ? ((Foo) x).m() : y;` → `return x instanceof Foo foo ? foo.m() : y;`
- Negated guard + early exit (flow scoping): `if (!(x instanceof Foo)) { return; } Foo foo = (Foo) x;` →
  `if (!(x instanceof Foo foo)) { return; }` (delete the redundant decl, use `foo`).
- Negated `||` short-circuit (VALID, don't drop it): `!(x instanceof Foo) || ((Foo) x).m()` →
  `!(x instanceof Foo foo) || foo.m()` — the `||` RHS runs only when the left is false, i.e. when the
  instanceof is TRUE, so `foo` is definitely assigned there. (Contrast the DROP case below: a negated
  instanceof in a `&&`/ternary whose cast is in the `:`/else branch is NOT in scope.)
- Existing explicit local: `if (x instanceof Foo) { Foo foo = (Foo) x; ... }` → `if (x instanceof Foo foo)
  { ... }` — REUSE that local's name and delete its declaration line.
`Object[]` patterns work too (`x instanceof Object[] arr`).

STRUCTURAL like S1066 (not a pure line-keyed edit) → DELEGATE reading+editing to PARALLEL
general-purpose subagents (NOT Explore — they must Edit), disjoint files, ~13 sites each; oldcore's
143 make a single-module batch (`-pl xwiki-platform-oldcore install -DskipTests`, ~6.5 min cold, clears 50).
When oldcore is already PR-claimed, the
next-densest single FEATURE module is a clean self-contained batch — reliable ones seen: notifications
~52, rendering ~41, extension ~34 (spread over 6-7 submodules, but 2 are Solr-based —
`-extension-index` + `-extension-security-index`, slow builds; DROP those two and the remaining 5 leaf
submodules — `-api`/`-cluster`/`-distribution`/`-handler-xar`/`-script` — give a 28/28 0-drop cheap
batch, incl. dense else-if safe-provider chains that convert cleanly), eventstream ~32,
security ~21 (concentrated in just 2 leaf modules — `-security-authorization-api` 16 + `-security-requiredrights-default` 5, 8 files, 20/21 0-drop cheap build), search ~21 (ALL in the one
`xwiki-platform-search-solr` submodule group: `-solr-api` + `-solr-query`, 6 files — the single most
concentrated 20+ batch, 21/21 0-drop first build). Prefer a module concentrated in FEW submodules
(security, search-solr) over one with the same issue count spread wide (extension) — fewer submodules = cheaper reactor build. Feature-module pools spread across 3-6 submodules /
6-18 files. **Concurrent sessions routinely leave SEVERAL S6201 feature-module PRs open at once** (seen NINE at
once: oldcore + refactoring + query-manager + security + search-solr + rendering + eventstream +
notifications + extension), so don't assume only oldcore is claimed — scan the WHOLE open `llm-agent`
PR list up front and pick modules with ZERO open PRs. Build all touched submodules in ONE
`-pl sub1,sub2,...` reactor (a 3-submodule feature module is a cheaper build than a 5-6 submodule one —
prefer the concentrated ones for ROI; note store submodules nest two levels deep, e.g.
`...-eventstream/...-eventstream-stores/...-eventstream-store-solr`). Split the files across 3-4 parallel
subagents BY MODULE/submodule (disjoint files never conflict — verify full coverage: `git diff
--name-only | wc -l` == expected file count). Fix rate is ~98-100% — 52/52, 45/45 and 32/32 seen 0-drop.
**When even the big feature modules are ALL claimed (the 9-10-at-once state — seen 10 open S6201 PRs
covering oldcore + notifications + rendering + eventstream + extension + security + search-solr +
query-manager/refactoring + the model/user/livedata/lesscss aggregate + a rest-server/chart-renderer/
filter-stream-xar/mail-send-default/mentions/office-importer aggregate), AGGREGATE untouched leaf
modules into one reactor to reach 20-50.** Trust NO fixed list — the "obvious" aggregate sets get
claimed too (both the model-api+user-default+livedata-livetable+lesscss set AND the rest-server+
chart-renderer+filter-stream-xar+mail-send+mentions+office-importer set were BOTH already-PR'd in one
run): re-query S6201 by module, subtract EVERY module named in an open S6201 PR title, and aggregate
whatever cheap untouched leaf modules remain. **The `xwiki-platform-legacy-*` modules are the reliable
untouched fallback** — the syntax/simplification/feature-module cleanup waves skip them, so they stay
unclaimed and can be DENSE (legacy-events-hibernate-api held 18, one file alone 16). Add `-Plegacy` to
the reactor (already in the standard command) and they build fast (~40s). A ~6-leaf-module mix built
around one legacy module + a handful of small `*-api`/leaf modules cleared 41/41 0-drop. The cleanest
~100% fodder is the internal
`*Reference`/`*Resolver`/`*Serializer`/colortheme/skin classes and equals-style
`if (!(o instanceof X)) return...; X x = (X) o;` / `return (X) ref;` guards, plus AST-visitor
converters (`node instanceof FooNode` chains) — all convert with no drops. Avoid feed-api (~5 min
build for a few issues) and Solr-based submodules.
- **Splitting files across subagents — verify FULL coverage.** When you partition the file list into
  N subagent groups, it is easy to drop a file from every group (missed one of 27). After the agents
  return, cross-check `git diff --name-only | wc -l` == the expected file count and fix any gap before
  building. Also note a pattern var can be a try-with-resources resource (`try (fooVar)`) — it's
  effectively final — so `instanceof Foo f` + `try ((Foo)x)` collapses cleanly.
- **Naming:** use idiomatic **camelCase**, NOT Sonar's all-lowercase suggestion (`alltablecolumns` →
  `allTableColumns`). Ensure no collision with an in-scope name.
- **Replace EVERY cast of that expression WITHIN the pattern var's scope.** A cast OUTSIDE that scope —
  the `else` branch, a later statement, or a cast to a DIFFERENT type (`(Object[]) result` beside
  `(String) result`) — is a separate/unflagged site; leave it.
- **DROP** when the cast can't reach the pattern var's flow scope: a negated `instanceof` with NO early
  exit whose only cast sits under a SEPARATE positive `instanceof`; OR a negated `instanceof` used as a
  TERNARY/`&&` CONDITION (not a guard with early exit) whose cast is in the `:`/else branch — a
  `x != null && !(x instanceof Y) ? ... : ((Y) x)...` short-circuits to that branch via `x == null` too,
  so the var isn't definitely assigned there. Also DROP instanceof+cast on unrelated expressions; name
  collision. Fix rate is high (~95-98%; e.g. 20/21 in the security modules, 52/53 in oldcore).
- **Line length is the #1 DROP cause in a feature module:** the rewritten line can breach 120 —
  first drop redundant parens to fit (`(x instanceof T v) && (!v.foo())` → `x instanceof T v && !v.foo()`),
  and pick a SHORTER in-scope pattern-var name (`reference` over `entityReference`) when that saves the line.
  But when the flagged site is a long pre-existing line with NO redundant parens — a lambda-field
  initializer (`BlockMatcher M = b -> ... instanceof RawBlock`) or an already-tight declaration — adding
  any idiomatic name overflows and it is an UNAVOIDABLE drop. This pulls a dense-feature-module fix rate
  down to ~90-93% (38/41 in rendering) vs oldcore's ~98%. Grep the diff for >120 after.

## java:S1068 / S1481 / S1854 — unused-code removal (deep MAJOR pool, best single-module batch)

The most reliable batch source once the simplification/syntax pools are drained: ~100+ open
project-wide, sometimes ~45 in oldcore ALONE (one dense module = a whole 20-50 batch in ONE build),
and untouched by the simplification/syntax cleanup PRs so rarely off-limits. **But density is NOT
guaranteed** — some runs the ~40-issue pool is thin-spread 1-2 per module across ~28 modules with
oldcore near-zero (or oldcore's few all DROPs). Then a WIDE ~24-module reactor of mostly leaf modules
builds green in one shot; and you can EXCLUDE the heaviest module (oldcore) entirely when ALL its
issues are DROPs — check that before adding it. **Bundle `java:S1161` (@Override, purely additive)
into the same batch for a clean multi-type PR** — Vincent's override explicitly allows mixed issue
types, and additive @Override adds no risk. Needs light dataflow judgement per site, so delegate the
reading to ONE Explore subagent that returns a precise action (DELETE range / STRIP_PREFIX /
INSERT_OVERRIDE / DROP+reason) per issue KEY; then apply BY LINE NUMBER in one atomic assert-guarded
Python script (repo is at the scan commit, lines don't drift). Expect to fix ~40 of ~45 after DROPs.

- **S1481 (unused local) + S1854 (dead store) fire as a PAIR on the same `Type x = expr;` line** —
  one edit clears both keys.
- **Pure RHS → delete the whole declaration line. Side-effecting RHS → KEEP the call as a bare
  statement, drop only the `Type name = ` prefix** (robust script: `indent + line.split(' = ',1)[1]`,
  preserves indentation). Must-keep side effects seen: in tests `doc.addAttachment(...)` /
  `newXObject(...)` (mutates the doc the asserts check), `registerMockComponent(...)`, a getter that
  lazily inits; in main code `velocityManager.getVelocityContext()` (initializes bindings).
- **`x = null` dead store in an `else` whose sibling `if` returns** → the whole `else` is redundant;
  delete the entire `} else { x = null; }` (collapse to the `if`'s closing `}`) rather than leaving an
  empty `else {}` block. A trailing dead `timer++` on a last use → drop just the `++`, keep the read.
- **Removing a private LOGGER/field usually ORPHANS its import** (`org.slf4j.Logger`/`LoggerFactory`,
  or the field's own type) → delete the now-unused `import` too, or Checkstyle `UnusedImports` fails.
  In the script, delete an import ONLY IF the type's SIMPLE NAME is absent from the FINAL post-edit
  content, matched with a WORD-BOUNDARY regex `\bLogger\b` — a plain substring check sees `Logger`
  inside `LoggerFactory` and wrongly keeps the orphan. This rule is safe both ways: name still present
  → keep the import (no checkstyle issue), absent → delete. (Removed local vars rarely orphan an
  import — the type is usually still used elsewhere; the same check confirms it.)
- **Clean up what the removal leaves behind (reviewers WILL flag both):** (a) any COMMENT that
  solely described the removed line — e.g. a `// Note: we use getRequestURI()...` above a deleted
  `requestUri` var, or a field's javadoc (remove the javadoc WITH the field). But KEEP a comment that
  describes a call you preserved (the strip-assignment case). (b) STRAY BLANK LINES: a statement
  surrounded by blanks leaves a DOUBLE blank; the last statement before `}` leaves a trailing blank;
  the first field after `{` leaves a leading blank. Collapse them. Checkstyle usually tolerates these
  so the build stays green — grep the diff context yourself, don't rely on the build to catch them.
  **ASSERT each blank you delete is truly blank (`line.strip()==''`) before removing it** — a
  subagent's "collapse this blank line N" hint can mis-point at a following member's `/**` javadoc
  opener (which looks like the start of the next block); deleting it would corrupt that member.
- **Removal CASCADES — delete the whole dead chain/block in one pass:** deleting `T x = other.getFoo()`
  can leave `other` (or a sibling local it read) newly unused; deleting a var can orphan a dead block (a
  `// comment` + a `Matcher`/`Pattern`/`baseClass` line that only fed it). Sonar flags only the outermost
  var, so the sibling looks "used" until you remove the first — then a follow-up scan re-flags it. Trace
  each removed RHS's inputs and delete the entire block (pure getters only; keep side-effecting calls).
- **A write-only field/var assigned in ONE place IS fixable:** delete the decl AND edit that single
  assignment — strip the `this.x = ` prefix if the RHS has a side effect to keep (`registerMockComponent`),
  else delete the assignment line too (a builder setter body collapses to just `return this;`; an
  `@Override` setter's now-unused param is NOT re-flagged — S1172 skips overrides).
- **DROP (don't fix):** a write-only field/var assigned in SEVERAL places (removing the decl alone
  breaks compile — would need deleting every assignment); a field exposed via a public setter (API); a
  dead store whose call must MOVE to a later line (coordinated multi-line change). These are the ~10-15%
  residue — leave them, the rest of the cluster still clears the batch.

## Other clean rules (verify they still have convertible issues — re-query each run)

- **`java:S2093` (try-with-resources):** convert `Resource r = new ...(); try { ... } finally {
  r.close()/IOUtils.closeQuietly(r) }` → `try (Resource r = new ...()) { ... }`. ~half of Sonar's hits
  are NOT real closes (push/pop, `stream.reset()`, semaphore release, resource created mid-body, state
  restoration) — verify the finally actually CLOSES an `AutoCloseable` declared just before `try`
  first. **Compile-safety:** the implicit `close()` throws `IOException`; if the surrounding catch is
  narrower and the method doesn't declare `throws IOException`, add a sensible `catch (IOException)`.
  **Removing `IOUtils.closeQuietly` may orphan the `IOUtils` import** → checkstyle failure; if it was
  the only use, delete the import. These issues are each in a different module (no cluster).
- **`java:S2119` (reuse Random):** extract `new Random()/new SecureRandom()` to a `private static
  final` field (SecureRandom is thread-safe; shared static is safe).
- **`java:S1143`+`java:S1163` (a `finally` block throws, masking the try's exception):** always fire as
  a PAIR on the same line — fix once, accept both keys. Replace the `throw` in the finally's inner
  catch with `logger.warn("Failed to close ...: [{}]", ExceptionUtils.getRootCauseMessage(e))`.
  **XWiki logging convention: wrap EVERY placeholder in square brackets `[{}]`, never bare `{}`.** If
  the class is an XWiki `@Component` and has no logger, add `@Inject private Logger logger;`
  (org.slf4j.Logger, NOT a static LOGGER) plus the `org.apache.commons.lang3.exception.ExceptionUtils`
  import (mind checkstyle import order: java/javax, then org.* alphabetical — slf4j between apache and
  xwiki). Grep the same module for a sibling that already does this rather than inventing the message.
- **`java:S5361` (replaceAll → replace):** convert only when the first arg reduces to a literal (no
  regex metachars `. * + ? [ ] ( ) { } | \ ^ $`, or a single escaped metachar `"\\+"`→`"+"`) AND the
  replacement has no `$`/`\`. Char classes, `\s+`, backrefs, `;jsessionid=.*?` are genuine regex —
  fix only the reported lines.
- **`java:S2147` (combine `catch` clauses with identical bodies) — clean, mechanical, safe** and
  clusters nicely (oldcore data-migration classes, REST resources each hold 1-2). `catch (A e){BODY}
  catch (B e){BODY}` → `catch (A | B e){BODY}`; comment-only body differences are fine to collapse.
  **Two gotchas:** (1) if the two types are in a SUBTYPE relationship (e.g. `UnsupportedEncodingException`
  extends `IOException`), a multi-catch is a COMPILE ERROR — instead DELETE the redundant subclass
  `catch` (identical body ⇒ behaviour-preserving). (2) Deleting a `catch` can ORPHAN that exception's
  import → checkstyle `UnusedImports` (a real build failure this run); remove the import if its simple
  name is now absent (same word-boundary check as the unused-code rules).
- **`java:S3626` (remove redundant jump) — clean ONLY for a TRULY-trailing jump.** SAFE: a trailing
  `return;` that is the last statement of a void method, or a branch-final `continue;`/`return;` when the
  if-else chain is the WHOLE loop body (so the jump lands exactly where fall-through would). DROP: (a)
  the jump is the branch's ONLY statement (or leaves only comments) — removing it makes an EMPTY /
  comment-only block → checkstyle `EmptyBlock`; (b) it sits in complex nested try/catch/finally where
  redundancy is hard to verify (e.g. a `return;` before a `finally` with code after) — too risky, skip.
  Line-length/import churn is nil; the only failure mode is the empty-block trap.

## Test-code rules — the deep clean pool once syntax/simplification/unused are drained

When the pure-syntax (S1128/S1197/S1116/S1161), simplification (S1125/S1488/S1858/S2864/S1612) and
unused-code (S1068/S1481/S1854) pools are all EXHAUSTED (re-query — they drain to single digits), the
biggest remaining CLEAN mechanical pools are the JUnit5 test rules `java:S5786` (hundreds) and
`java:S5785`. Pure test-code edits — production code untouched, so very low review risk. **S5786 is
frequently SATURATED by concurrent sessions (4+ open PRs at once covering oldcore, model-api and many
leaf modules) — when it is, S5785 is the go-to fallback: the S5786/syntax cleanup waves never touch it,
and it has dense SINGLE-MODULE clusters (seen: chart-macro 50, model-api 27, oldcore 22) that one cheap
build clears — check the densest module and just do that one.**

- **`java:S5786` (JUnit5 test class/method should be package-private) — the best test-rule batch.** Two
  message variants: **method-level** "Remove this 'public' modifier" and **class-level** "Remove
  redundant visibility modifiers from this test class and its methods". Do NOT infer scope from the
  message/line — the method-level "Remove this 'public' modifier" often points at the CLASS-decl line
  itself (class public, methods already package-private), NOT a method. So key the fix BY FILE, not by
  line: for every flagged file apply the same uniform treatment (strip class decl + `@Nested` +
  JUnit-annotated methods). Fix mechanically in one Python script: strip the leading `public ` token
  (`re.sub(r'\bpublic\s+', '', line, count=1)`) from (a) the flagged class-declaration line and (b)
  every method line whose immediately-preceding contiguous annotation block contains a JUnit annotation
  (`@Test`/`@BeforeEach`/`@AfterEach`/`@BeforeAll`/`@AfterAll`/`@ParameterizedTest`/`@RepeatedTest`/
  `@TestFactory`/`@TestTemplate`/`@Nested`). Keep other modifiers — `@BeforeAll public static void` →
  `static void`. Do NOT touch fields or unannotated helper methods: they aren't flagged and leaving them
  public won't re-flag the class. **Keep the allowlist to REAL JUnit annotations only** — a method
  annotated with an XWiki-specific (non-JUnit) lifecycle annotation such as `@BeforeComponent`/
  `@AfterComponent` is NOT flagged by S5786, so leave its `public` intact (a `@BeforeComponent public
  void configure(MockitoComponentManager cm)` stays public and won't re-flag; only its JUnit-annotated
  siblings like `@BeforeEach setup()` are stripped). It IS also safe to strip `public` from a NESTED helper/`@Nested` class
  in a test file (same package — won't re-flag, no cross-package caller in practice), so a simple
  `public (abstract|final)? (class|interface|enum)` match covering top-level AND nested decls is fine.
  Behaviour-preserving; `-DskipTests` still test-compiles + runs Checkstyle so it fully validates
  (oldcore's `install -DskipTests` build is ~4 min and clears a 32-issue batch cleanly).
  **Check oldcore FIRST** — like the other rules it is frequently a dense single-module batch (32 seen,
  each in its own file), which one `-pl xwiki-platform-oldcore` build clears with no reactor juggling.
  Otherwise the pool is a thin spread (~1-8 per module across dozens of leaf modules once the dense
  modules are PR'd): pick a CLUSTER of fast leaf `*-api`/leaf modules totalling ~30 and build them ALL
  in ONE reactor `-pl m1,m2,...` (a couple heavier `-api` modules ~2.5 min each dominate; leaf modules
  ~5-45s). **Ultra-thin case (~1/module):** when the spread is only 1-2 per module the ~30-issue batch
  needs ~30 modules — don't be deterred by the width, a 32-module reactor (one issue each, incl. a
  mix of `*-api`, xar `*-ui` page-test and legacy modules) still builds green in ONE shot, dominated by
  its one heaviest module. This is the correct play when S5786 is the ONLY pool ≥20 and every other rule
  is drained to single digits (S1066/unused/S1192 all <10, syntax/simplification/S5785 at 0 — a common
  concurrent-session state). **Open S5786 PRs usually sit on SIBLING modules, not yours:** a prior PR
  covering `annotation-core`/`store-merge`/`localization-source-jar`/`search-solr`/`livedata-livetable`
  leaves `annotation-reference`/`store-transaction`/`localization-source-wiki`/`search-ui`/`livetable-ui`
  fully fair game — sibling modules under one parent are DISTINCT modules with no file overlap, so scope
  the off-limits check by exact module path, and don't abandon S5786 just because 2-3 of its PRs are open. **Cross-module compile check** (matters most in oldcore, which publishes a widely-used
  test-jar): an oldcore-only build won't catch a subclass in ANOTHER module breaking when its base goes
  package-private, so `grep -rl "extends <Class>" --include=*.java xwiki-platform-core | grep -v <thisModule>`
  for each class made package-private. Concrete `*Test` classes are never extended cross-module (check
  comes back empty); the risk is only `abstract`/base test classes — and a class NAMED `Abstract*Test`
  is often NOT abstract and has no subclasses, so read the decl, don't trust the name.
  **When EVERY mechanical family is simultaneously thin**, PREFER a uniform single-rule S5786 wide
  reactor over a MIXED multi-rule reactor: one uniform per-file script keeps a wide reactor low-risk,
  whereas mixing 5+ rule mechanics multiplies edit-error surface for the same fix count.
  **BUT when even S5786 AND S5785 are BOTH multi-PR-saturated** (2–3 open `llm-agent` PRs each,
  their remaining OPEN issues sitting in modules those PRs already claim) AND no single other rule reaches
  20, the correct fallback IS a MIXED batch of the small ZERO-PR pure-mechanical rules — S1612 + S1125 +
  S2864 + S1155 + S1197 + S1128 together reached 24 across ~20 modules in one green reactor. The
  error-surface caution above applies to DATAFLOW rules; these six are all zero-dataflow single-line edits
  (method ref / boolean literal / entrySet / isEmpty / array designator / unused import), so a scripted
  mixed batch stays low-risk. Pivoting to zero-PR rules beats threading the gaps in a PR-saturated rule.
  A class-level flag makes the script strip ALL that file's `@Test`/lifecycle methods, so a
  dense test file yields far MORE `public` removals than its flagged-issue count (e.g. 40 removals in one
  file flagged with 2 issues) — expected, not an error.
- **`java:S5785` (use assertEquals/assertNotEquals/assertNull, not a boolean assert) — fully SCRIPTABLE
  per file by line number** (the earlier "prefer S5786, needs judgement" caution was too conservative).
  The issue MESSAGE names the exact target ("Use assertEquals/assertNotEquals/assertNull instead"), so no
  guessing. **Default to RECEIVER-FIRST for the `.equals()` shapes — it is universally safe and removes the
  need to hunt for asymmetric equals:** `assertTrue(a.equals(b))`→`assertEquals(a, b)`;
  `assertFalse(a.equals(b))`→`assertNotEquals(a, b)`. JUnit's `assertEquals(expected, actual)` calls
  `expected.equals(actual)`, so keeping the ORIGINAL receiver first reproduces the exact same
  `a.equals(b)` call — correct even when `a.equals` is custom/asymmetric or `b` is a DIFFERENT type
  (e.g. `event.equals(anotherTypeEvent)`, a regex/matcher reference). The only cost is cosmetic (the
  failure message's expected/actual labels are swapped when `b` is the "expected" literal); for symmetric
  value classes either order passes, so receiver-first is a safe single rule for the whole file — no
  per-site asymmetry analysis. (The old "flip to `assertEquals(b, a)`" caused a real test failure in
  model-api `RegexEntityReferenceTest`; receiver-first avoids it entirely.) Other shapes: `assertTrue(LIT
  == x)`/`assertTrue(x == LIT)`→`assertEquals(LIT, x)`; `assertTrue(x != LIT)`→`assertNotEquals(LIT, x)`
  (covers `hashCode() != 0`→`assertNotEquals(0, x.hashCode())` and `== 0`→`assertEquals(0, ...)`);
  `assertTrue(null == x)`→`assertNull(x)`. **Degenerate `.equals()` forms convert too — trust the
  message:** `assertFalse(x.equals(null))`→`assertNotEquals(x, null)` and self-equality
  `assertTrue(x.equals(x))`→`assertEquals(x, x)` both compile and pass (JUnit uses `Objects.equals`, so
  the null/self case never routes through a custom `equals`); no need to "improve" them to
  assertNotNull. **`==`/`!=` between two REFERENCES** (identity, not value) is
  a distinct message "Use assertSame/assertNotSame instead": `assertTrue(a == b)`→`assertSame(a, b)`,
  `assertFalse(a == b)`→`assertNotSame(a, b)` (identity is symmetric so operand order is cosmetic; TRUST
  the message to choose same-vs-equals — e.g. attachment identity, enum constants). Only convert the
  FLAGGED lines — sibling `assertTrue(x
  instanceof Y)` lines are NOT flagged and must stay (so `assertTrue` often survives; keep its import).
  A message-string arg (`assertTrue(cond, "msg")`) just moves to the end: `assertEquals(a, b, "msg")`.
  **Imports:** add the new static imports (`assertEquals`/`assertNotEquals`/`assertNull`), remove
  `assertTrue`/`assertFalse` ONLY when the file no longer uses them (grep after) or Checkstyle
  `UnusedImports` fails; insert alphabetically (`assertNotEquals` sits between `assertFalse` and
  `assertNull`). **Duplicate-line gotcha:** identical assert lines can recur in different test methods
  (e.g. same call + same message string in two tests) — a `content.count(old)==1` assert then trips;
  diagnose, and if both convert identically just replace with the real count. **Already-half-fixed site:**
  a flagged `assertTrue(x != Y)` sometimes sits directly ABOVE an already-present `assertNotSame(Y, x)`
  (a prior partial fix) — converting would DUPLICATE the next line, so just DELETE the redundant flagged
  line (and drop its now-unused import). Always eyeball the adjacent line before converting. **Compile
  note (not a risk):** `assertEquals(0, method())`/`assertNotEquals(0, x.hashCode())` resolve via `int`
  overload or autoboxing to `(Object,Object)` — both compile and preserve semantics. **Build WITH the
  flagged tests as the operand-correctness safety net, but you do NOT need the module's FULL suite:** run
  `install ... -Dtest=<the flagged test classes> -DfailIfNoTests=false` — Checkstyle + full test-compile
  still run (that's what `install` gates), only the flagged classes execute (~5s even in oldcore). This
  makes oldcore a perfectly cheap S5785 batch and generalises to any test-rule that "must run tests".
  **Module choice:** S5785 clusters in dense single files (seen: `XWikiDocumentMockitoTest` 15,
  `ActionExecutingEventTest` 14, `SimpleEventQueryTest` 12); oldcore is a solid dense single-module batch
  (22 seen across 4 files) even when the previously-dense modules (bridge/eventstream-api/model-api/
  chart-macro) are already PR'd — check oldcore FIRST. When no single module hits 20, COMBINE two small
  dense modules in one reactor (e.g. bridge + eventstream-api).

## Find-phase cost

- Inline `sed`/`Read` (offset/limit) of ONE candidate region is cheaper than an Explore subagent for
  mechanical rules; use a subagent only when you must read & reject several candidates. Always trim
  `issues/search` JSON through `python3`/`jq` (keep key,rule,component,line,message) — some rules
  attach huge `flows`/`locations`; never dump raw responses into context.
- Component key = `groupId:artifactId:path` — the projectKey itself contains a colon
  (`org.xwiki.platform:xwiki-platform:xwiki-platform-core/...`), so it has TWO colons. Get the path with
  `component.split(':')[-1]`, NOT `split(':',1)[1]` (that leaves `xwiki-platform:...` and every file
  open fails). Read locally at `/home/user/xwiki-platform/<path>`; never fetch file contents remotely.

## Batch mode ("fix ALL / 20-50 of rule X")

- One `issues/search?rules=<rule>&ps=500` gives every open issue + component + line. Group by
  file/module.
- **Apply a many-file mechanical batch in ONE Python script**, not dozens of `Edit` calls: for each
  `(file, old, new)` assert `content.count(old) == 1` FIRST (catches stale/drifted/ambiguous matches),
  and write NOTHING if any assertion fails. Atomic, cheap, and a failed count flags a bad target before
  it corrupts the tree. (Reading via `sed`/`grep` for the snippets is fine — you don't need the `Read`
  tool before this script; it edits files directly.)
- **Collecting the issue keys to accept:** re-query the rules, then filter issues by a substring of the
  full component PATH (`.../xwiki-platform-chart-macro/...`), NOT a guessed short module name — a
  short-name match silently returns 0.
- **Build ALL affected modules in ONE reactor invocation**, not one-by-one:
  `mvn clean install -B -ntp -pl m1,m2,m3,... -Plegacy,snapshot -Dxwiki.revapi.skip=true
  -Dxwiki.surefire.captureconsole.skip=true -DskipTests`. Maven sorts by dependency order; missing
  transitive deps resolve from the remote snapshot repo (`-Psnapshot`). `-Plegacy` is required to
  include `*-legacy-oldcore`.
- Accept all issues in one loop (per issue: `add_comment` + `do_transition` accept). Each issue's 2
  curls to sonarcloud run ~4s round-trip, so even ~30 issues blow a 2-min foreground timeout (seen: 29
  of 31 done at the cutoff) — run the accept loop as a BACKGROUND task for 20+ issues, or make it
  idempotent (query which keys are still OPEN and only accept those, so a re-run finishes the tail).
  **The `do_transition` response does NOT reliably contain an `issues` key** — don't
  `json.load(...)['issues']` (KeyError); the transition still applied. Confirm separately with one
  `issues/search?issues=<comma-keys>` → all ACCEPTED/RESOLVED. **A transition occasionally no-ops for
  ONE issue in a large batch (e.g. 1 of 35 stayed OPEN) — after the loop, count statuses and re-POST
  `accept` for any straggler.**

## Building / verifying

- Checkstyle + Spoon run in `install` (that's what catches issues); `-DskipTests` is fine.
- Pick the smallest leaf module(s). Rough datapoints: small plugin/notifier leaf modules ~10s each in
  a warm reactor; oldcore ~3.5 min warm / ~6.5 min cold; feed-api ~5 min.
- **Always run mvn from the repo root** (`cd /home/user/xwiki-platform && mvn ...` or `-f
  /home/user/xwiki-platform/pom.xml`): the shell cwd can silently reset to `/home/user` between turns,
  and a `-pl <relative>` build from the wrong cwd fails fast with `Could not find the selected project
  in the reactor` (a path error, not a code error — just relaunch from root).
- Run the build in the **background**, letting the run-in-background tool capture stdout to its own
  `tasks/<id>.output`. Do NOT add your own `> build.log` redirect (leaves the tool's file empty) and do
  NOT wrap in `nohup … &` (one layer of backgrounding only). NO `| tail` (breaks incremental grep).
  The completion notification carries the exit code; ONE grep for `BUILD SUCCESS` afterwards confirms.

## Cost control (the build wait dominates the bill)

- Every wait/poll/wakeup turn re-reads the full cached context — minimize those turns. Once the build
  is running, STOP: one line, end the turn. Do NOT Read/grep the log while it runs; the notification
  wakes you. Don't arm a short `ScheduleWakeup` (300s) for a build — the notification already wakes
  you; if you want a fallback use 1200s+.
- The stop-hook ("uncommitted changes") firing while you wait on the build is EXPECTED — do not commit
  before the build verifies.
- **A container restart can kill the background build mid-run.** Working-copy edits PERSIST
  (uncommitted), so don't re-do the fix — check `git status`/`git diff`, confirm the branch, and
  re-launch the same `mvn` build. Don't panic-commit unverified.

## GitHub (`gh` is NOT available — use the GitHub MCP tools)

- Check existing agent PRs once up front: `search_pull_requests` with
  `is:pr is:open label:llm-agent repo:xwiki/xwiki-platform`. Do NOT use `list_pull_requests` (~660KB).
- Create the PR with `create_pull_request` (base `master`); add the label with `issue_write`
  (method=update, `labels:["llm-agent"]`). Include the SonarCloud issue link(s) in the body.
- Creating the PR auto-subscribes the session to PR webhooks. XWiki CI is Jenkins and reports later, so
  `get_status` is `pending`/`total_count:0` right after creation — NOT a failure. Webhooks don't
  deliver CI-success / new-push / merge-conflict transitions; for long watches schedule a ~1h self
  check-in (`send_later` if present, else `CronCreate`) and re-arm silently until merged/closed.
- Reply to reviewers only when genuinely necessary (e.g. explaining why a suggestion can't be done);
  be frugal with comments.

## Process / conventions

- Commit + PR title (no JIRA): `[Misc] <desc; mention SonarCloud/SonarQube>`. Security issues: keep the
  description cryptic (public logs).
- **Author override:** `git config user.email <email>` AND `git commit --author="Name <email>"` AND a
  `Co-Authored-By: Name <email>` trailer — verify with `git log -1 --format='%an <%ae>'`. (This
  routine's override email differs from the git userEmail context — use the override.)
- Prefer a file with a SINGLE issue of the rule for clean one-PR-per-issue mode; the count of OPEN
  issues in a file should equal the number you convert (a clean cross-check).
- **Reset the designated feature branch to master FIRST — it persists across runs.** A previous run's
  commits may still sit on it; once their PR merged, `git fetch origin master` then `git log
  origin/master..HEAD` shows 0 (they're in master) — reset with `git checkout -B <branch>
  origin/master` before editing, or the new PR bundles all the old already-merged commits. (A stale
  local `origin/master` hides this: it can show N commits "ahead" that vanish after the fetch — always
  fetch before judging.)
- **Recording learnings (memory repo → `main`):** the xwiki-platform fix lives on a feature branch but
  learnings go to `main`. Do NOT edit on the feature branch then stash/checkout/pop (main has diverged;
  the pop conflicts and can bake `<<<<<<<` markers into the commit). Instead `git checkout main &&
  git pull origin main` FIRST, then edit and commit directly on main.

## Token-cost report (when asked)

- Parse the transcript jsonl at `/root/.claude/projects/<id>.jsonl`. Dedupe assistant turns by
  `message.id` keeping the LAST record per id (streamed partials accumulate; the tool_use block often
  appears only in a later partial), then sort by timestamp. Find phase boundaries by tool_use NAME
  (`b["name"]=="mcp__github__create_pull_request"` + the target Edit), NOT string-match on content
  (`"create_pull_request"` appears in early schema text → false boundary); check timestamps are monotonic.
- Buckets: (1) find = up to first fix edit; (2) fix = edit + build + commit + push; (3) post = PR +
  label + accept + report. Cache-read dwarfs everything. Add the report to the PR **body**, not a comment.
