# SonarQube fix — learnings

Generic, reusable playbook for the "fix one (or a batch of) SonarQube issues in xwiki-platform"
routine. Keep this compact and generic: record techniques and gotchas, NOT run history. When you
learn something, merge it into the right section and trim — don't append dated anecdotes or PR logs.

## Picking a target rule (find phase)

- Get the rule distribution cheaply FIRST (no issue bodies): one `issues/search?...&issueStatuses=OPEN&facets=rules&ps=1`
  call returns the whole project rule distribution. For an exact per-rule count read the response
  `total` (query with `&rules=java:SXXXX&ps=1`), not a facet value.
- Pools shift every run (see General techniques). If a rule's remaining issues are all non-convertible
  residue, pivot (skill rule: "if a fix is hard, drop it and pick another").
- **The BLOCKER/CRITICAL mechanical pool is frequently exhausted.** BLOCKER/CRITICAL-first is the
  skill's guidance but not a hard gate — a clean MAJOR fix beats forcing a risky higher-severity one.
  There is a deep MAJOR-severity clean pool.
- **Check open agent PRs up front** (`search_pull_requests` with `is:pr is:open label:llm-agent
  repo:xwiki/xwiki-platform`; `list_pull_requests` is too big). A recent PR can drain a WHOLE rule
  family. But scope the off-limits check by **(rule + module)**, not rule alone: a per-module batch PR
  only claims the files it touched, so the same rule in OTHER (incl. sibling) modules is fair game.
  When your planned rule already has multiple open PRs, PIVOT to a zero-PR rule family rather than
  threading the gaps. When you must know exactly which modules a wildcard "various modules" PR claims,
  read its file list (`pull_request_read` `get_files`).
- **Rule-family map, easiest/safest first:**
  - *Pure syntax/annotation* (zero dataflow, safest): `S1128` unused import, `S1197` array designator,
    `S1116` empty statement, `S1161` missing `@Override`, `S1611` redundant lambda-param parens,
    `S1124` modifier order, `S3878` redundant varargs array, `S1118` add private constructor to a
    utility class (additive).
  - *Pure simplification* (no use-verification): `S1125` redundant boolean literal, `S1488` inline
    returned local, `S1858` pointless `toString()` on String, `S2864` iterate `entrySet()`, `S1612`
    lambda→method ref, `S1155` `size()>0`→`!isEmpty()`, `S1126` if-then-else→single return.
  - *Constant extraction*: `S1192` duplicated literal.
  - *Unused-code removal* (light dataflow): `S1068` field, `S1481` local, `S1854` dead store, `S1144`
    unused private method, `S1185` remove override that only calls `super`.
  - *Structural*: `S1066` merge nested `if`, `S6201` instanceof pattern matching (the deepest pool),
    `S2147` combine catch, `S3626` remove redundant jump, `S2093` try-with-resources.
  - *Test-code*: `S5786` JUnit5 package-private, `S5785` assertEquals/assertSame not boolean assert.
- **Denylist — skip these** (bad ROI / risky / not one-liners): `S3776` (cognitive complexity),
  `S3252`/`S1845` (API/backward-compat), `S1186` (empty methods), `S115` (naming), `S2447` (null from
  Boolean method — in XWiki *script services* null is a deliberate "check getLastError()" contract),
  `S1214` (constants-in-interface, cross-module), `S1113` (finalize), `S1215` (`System.gc()` — the
  enclosing method may be a deliberately-exposed API, e.g. `$xwiki.gc()`), `S2696` (static field from
  instance method — usually lazy-init needing sync), `S2157` (add `clone()`), `S5845` (assert
  dissimilar types — erasure can make the assertion correct). Verify before "fixing" any of these.

## General batch-fix techniques (apply to every rule below)

Cross-cutting mechanics shared by all rules; each rule section notes only its *deltas*.

- **The repo is at the exact scan commit** → line numbers don't drift, so line-number-keyed editing
  is safe (process each file bottom-up, or map over ORIGINAL indices).
- **Apply a many-file mechanical batch in ONE assert-guarded script**, not dozens of `Edit` calls: for
  each `(file, old, new)` assert `content.count(old) == 1` FIRST (catches stale/drifted/ambiguous) and
  write NOTHING if any assertion fails. (`Edit`'s `replace_all` matches only the exact indentation you
  typed and SILENTLY leaves the same pattern at other depths — so prefer the script, and grep for the
  residual pattern after any batch replace.)
- **Delegate STRUCTURAL rules** (reindent/brace surgery — S1066/S6201) and **per-site dataflow rules**
  (S6204/S1068/…) to PARALLEL general-purpose subagents (NOT Explore — they must Edit) over DISJOINT
  files. **Never trust a subagent's self-reported "CONVERTED"** — it routinely reports (with a
  plausible rationale) editing a file it never touched, and a missed dataflow site still COMPILES so
  the build won't catch it. After any delegated batch: cross-check the EXACT expected file set against
  `git diff --name-only`, open every expected-but-absent file and apply the missed sites yourself, and
  re-grep the pre-fix pattern across changed files (only intentional keeps should remain).
- **Two issues on the SAME line** (e.g. two `new T[]{}` on one line) → do NOT stack two edits both keyed
  to the original full line: the second's stored "old" goes stale after the first applies and only one
  lands (or the assert aborts). COMBINE them into one edit covering both replacements.
- **Sonar attributes a multi-line statement's issue to the STATEMENT-START line, not the line the flagged
  expression is actually on** — the reported `line` can be a few lines above the token. When a
  line-number-keyed edit's `old` isn't found there, grep the file for the actual token to get the real line.
- **Line length is the #1 drop cause:** a rewritten line can breach 120 — first drop redundant parens,
  then pick a SHORTER in-scope name; a pre-existing long line with no slack is an unavoidable drop.
  Grep the diff for >120 after every batch.
- **Deleting code orphans its import** → remove the now-unused import too, but ONLY IF the type's
  SIMPLE NAME is absent from the FINAL content, matched with a WORD-BOUNDARY regex (`\bLogger\b`, else
  a plain substring sees `Logger` inside `LoggerFactory`).
- **Pools shift every run** as PRs land — clean rules get EXHAUSTED then slowly REGENERATE; always
  re-query, never assume a rule still has convertible issues.

## Batch mode (Vincent's "fix 20-50 / all of rule X" override)

- Mixed issue types in one PR are explicitly allowed — bundle a purely-additive rule (`S1161`
  @Override) into a removal/simplification batch for a clean multi-type PR.
- **Reach the 20-50 target by density-first module selection:** query the rule(s) with `&ps=500`,
  group by module (`component.split(':')[-1]` for the path — the projectKey has TWO colons, so
  `split(':',1)[1]` is WRONG and every file open fails). Then either: (a) one dense single module
  (oldcore often holds 30-90 of a rule = one cheap build); or (b) a WIDE reactor of many cheap leaf
  modules when the pool is thin-spread. A 30+-module reactor still builds green in ONE shot.
- **When a rule's small pool sits below 20 alone, MIX zero-PR pure-mechanical rules** (S1612 + S1125 +
  S2864 + S1155 + S1197 + S1128, all zero-dataflow single-line edits) across ~20 modules into one
  green reactor. Pivoting to zero-PR rules beats threading the gaps in a PR-saturated rule. Reserve
  mixed *dataflow*-rule batches for when unavoidable (each rule multiplies edit-error surface).
- The mechanics for APPLYING a batch — assert-guarded script, subagent delegation + verification, line
  length, orphaned imports — are in **General batch-fix techniques** above; this section covers only
  batch *composition* (which rules/modules to bundle).
- **Collect issue keys by a substring of the full component PATH** (`.../xwiki-platform-chart-macro/...`),
  NOT a guessed short module name (silently returns 0). Build the accept list by KEY, not edit count
  (a triple-nest S1066 merge or a class-level S5786 flag resolves more keys than edited sites).
- **Accept all issues in a loop** (per issue: `add_comment` + `do_transition accept`). Each issue is
  ~2 curls ≈ 4s, so 20+ issues blow a 2-min timeout — run the accept loop as a BACKGROUND task, and/or
  make it idempotent (re-query which keys are still OPEN). `do_transition`'s response does NOT reliably
  contain an `issues` key (don't `['issues']` → KeyError; the transition still applied). Confirm
  separately with `issues/search?issues=<keys>` → all ACCEPTED; re-POST accept for any straggler.

## java:S6201 — instanceof pattern matching (the deepest clean pool; go-to when small pools drained)

By far the largest clean pool (hundreds open, ~90-140 in oldcore alone) and it barely regenerates
down — the reliable batch source when the small mechanical rules are all simultaneously drained below
20 (the common concurrent-session state). Java 16+ (xwiki-platform 18.x is Java 17 — compiles).
Message: "Replace this instanceof check and cast with 'instanceof Foo foo'". Fix = bind a pattern
variable and delete the redundant cast:
- Positive guard: `if (x instanceof Foo) { ((Foo) x).m(); }` → `if (x instanceof Foo foo) { foo.m(); }`
- Compound `&&` (pattern var scopes rightward + into block): `x instanceof Foo && ((Foo) x).m()` →
  `x instanceof Foo foo && foo.m()`
- Ternary/return: `return x instanceof Foo ? ((Foo) x).m() : y;` → `... instanceof Foo foo ? foo.m() : y;`
- Negated guard + early exit (flow scoping): `if (!(x instanceof Foo)) { return; } Foo foo = (Foo) x;`
  → `if (!(x instanceof Foo foo)) { return; }` (delete the redundant decl, use `foo`).
- Negated `||` short-circuit (VALID): `!(x instanceof Foo) || ((Foo) x).m()` → `!(x instanceof Foo
  foo) || foo.m()` — the `||` RHS runs only when the instanceof is TRUE, so `foo` is assigned there.
- Existing explicit local: `if (x instanceof Foo) { Foo foo = (Foo) x; ... }` → reuse that local's
  name in the pattern and delete its declaration line. `Object[]` patterns work (`x instanceof Object[] arr`).

**Module choice.** oldcore's ~140 make a single-module batch (`-pl xwiki-platform-oldcore
install`, clears 50). When oldcore is PR-claimed, the next-densest FEATURE module is a
clean self-contained batch — PREFER one concentrated in FEW submodules (cheaper reactor) over the same
count spread wide, and DROP Solr-based submodules (`-*-index`, `-solr-*` — slow) and feed-api (~5 min).
**Concurrent sessions routinely leave 9-10 S6201 feature-module PRs open at once** (incl. oldcore +
several wildcard "various modules" PRs) — often leaving only thin residue spread 1-2 per module across
26+ submodules; when that's all that's left, PIVOT (the S1118/S1144/S1185 family above is the go-to).
**When even the big feature modules are ALL claimed, AGGREGATE untouched cheap leaf modules into one
reactor to reach 20-50.** Trust NO fixed list — the "obvious" aggregate sets get claimed too;
re-query by module each run. The `xwiki-platform-legacy-*` modules are a reliable untouched fallback
(cleanup waves skip them; add `-Plegacy`, they build fast). The aggregate-leaf pool is dominated by
near-100% 0-drop fodder: **event-listener `onEvent(Event event, ...)` guards**
(`if (event instanceof XEvent) { ...((XEvent)event)... }`), **exception-rethrow guards**
(`if (e instanceof XException) { throw (XException) e; }`), equals-style `if (!(o instanceof X))
return; X x=(X)o;`, servlet-filter `request/response instanceof HttpServlet*`, and internal
`*Reference`/`*Resolver`/`*Serializer` + AST-visitor converters (near-0-drop fodder).

**Mechanics & drops.** STRUCTURAL → delegate to PARALLEL general-purpose subagents (disjoint files,
~13 sites each), splitting BY module/submodule; verify full coverage after. A pattern var can be a
try-with-resources resource (effectively final). **Naming:** idiomatic camelCase (NOT Sonar's
all-lowercase); no collision with an in-scope name. **Replace EVERY cast within the pattern var's
scope**; a cast OUTSIDE scope (else branch, later statement, or to a DIFFERENT type) is a separate
site — leave it. **DROP** when: the cast can't reach the pattern var's flow scope (negated instanceof
with no early exit whose cast is under a separate positive instanceof; or a negated instanceof used as
a ternary/`&&` CONDITION whose cast is in the `:`/else branch — `x != null && !(x instanceof Y) ? ...
: ((Y) x)...` short-circuits via `x==null` too, so the var isn't definitely assigned); name collision;
unrelated expression. **Line length** (see General techniques) is the #1 drop cause in dense feature
modules — a long pre-existing line (lambda-field initializer, tight decl) with no slack is an
unavoidable drop. Fix rate ~95-100% (oldcore ~98%, dense feature modules ~90-93%).

## java:S1066 — merge collapsible nested `if` (structural; reliable pivot when all else drained)

Reliable when syntax/simplification/unused/test pools are all drained — STRUCTURAL, so those cleanup
waves never touch it, and it regenerates. Density NOT reliably oldcore-concentrated (sometimes ~70 in
oldcore = one batch; other runs thin-spread across ~16 modules). A wide reactor clears it in one
`install`. Sonar flags the INNER `if`. Fix: `if (A) { if (B) { BODY } }` →
`if (A && B) { BODY }` — merge with `&&`, delete the inner `if` line, DEDENT the body by 4, remove ONE
trailing brace. Wrap an operand containing top-level `||` in parens. NOT a pure line-keyed edit →
DELEGATE to parallel subagents, each file bottom-up. **A triple-nest `if(A){if(B){if(C){}}}` collapses
to `if (A && B && C)` and resolves TWO keys** — count accepted issues by KEY.
**Fixable ~75%.** DROP: merged condition >120 that can't cleanly two-line-wrap (+4 continuation
indent); the outer is `else if`; the inner `if` is not the sole statement of the outer body (siblings/
an `else`). A residual `X != null && X instanceof Y` is harmless. **A comment BETWEEN the two `if`s is
usually recoverable** (not an auto-drop): a single-line `//` describing the inner condition MOVES
above the merged `if` (same indent); DROP only a multi-line/block comment or one documenting the OUTER
condition.
**Subagent brace-surgery gotcha:** a subagent can leave a STRAY `}` (cascade `illegal start of type`).
ALWAYS build after structural subagent edits (and verify the file set per General techniques); one bad
file doesn't condemn the batch. **Cheap pre-build check:** per file, `open-Δ = #{(HEAD) - #{(working)` and
`close-Δ` similarly; a correct merge removes exactly one `{` and one `}` per issue, so
`open-Δ == close-Δ ==` the file's issue count. Any mismatch = stray/missing brace, inspect first.

## java:S1192 — define a constant for a duplicated literal

**Feasibility check FIRST:** the OPEN pool fluctuates and is sometimes <20 project-wide — then the
20-50 batch is IMPOSSIBLE, don't pick it; mix pure-simplification rules instead. If enough: combine
files across small leaf modules (cheap), or fix ONE module that alone holds ≥20 (oldcore often does).
**Fix:** add `private static final String NAME = "literal";` and replace EVERY occurrence in that ONE
file (rule fires at ≥3, so ≤2 copies left also resolves it).
**Count-verify each literal** (line numbers drift): count the quote-bounded token
(`content.count('"admin"')`, avoids `"administrator"` substring hits). When count != Sonar's N:
- Excess in COMMENTS/javadoc → FIXABLE: replace only on non-comment lines (skip lines whose lstrip
  starts with `*`/`//`/`/*`), assert the CODE count == N.
- Substring/fragment mismatch (Sonar counts a concatenation fragment, or your token is a substring of
  a longer literal) → NOT fixable, DROP.
- STALE (literal absent from file entirely) → already fixed on master, not rescanned; DROP.
Aim for a few more than the target so drops don't sink the batch.
**Script:** replace on original content asserting each count, THEN insert the constant block (insert
first and the replace pass rewrites your own decls). After the batch, grep added lines >120 — a
constant NAME longer than the literal (esp. a fully-qualified ref) can breach checkstyle; rewrap.
**Where to declare:** forward-reference gotcha — Java forbids a static field's simple name *before*
its textual declaration inside a static initializer (incl. static array/collection initializers);
declare the constant ABOVE such a use. Method-body-only uses: order irrelevant. XWiki checkstyle does
NOT enforce public-before-private field order, but where `DeclarationOrder` bites, place the constant
after public fields and before the first private field that uses it (near `LOGGER`). Private constants
need no javadoc.
**"Use already-defined constant" variant** is FIXABLE when the named constant is declared BEFORE the
duplicating line; UNFIXABLE (forward ref) when after. DROP when the value match is COINCIDENTAL and
semantics differ (a `DefaultPluginName="package"` vs an XML root-element `"package"`).
**Reviewer preferences (apply pre-emptively):**
1. A literal used ONLY in a log-message concat → don't introduce a constant; convert to slf4j
   parameterized syntax `LOGGER.x("Prefix [{}] ...", var)` (bracket the placeholder — XWiki
   convention). If that message is itself duplicated ≥2×, extract ONE constant for the WHOLE message.
2. A **domain property/field name** (an XWiki class property) → reuse/add a **public** constant on the
   owning `*DocumentInitializer` (grep its `add*Field("...")` calls); `"XWiki"` system-space already
   lives at `com.xpn.xwiki.XWiki.SYSTEM_SPACE`. BUT keep it a LOCAL private constant when the owning
   constant is `private` in an `internal` package, or a fully-qualified ref would breach 120 chars.
3. **Any newly-public API (incl. a field widened private→public) needs an `@since` tag.** Format =
   reactor `project.version` with `-SNAPSHOT`→`RC1` (grep `pom.xml` `<version>`, confirm against
   recent `@since 18.` tags). Last javadoc line, after a blank ` *` separator.
**Checkstyle-excludes trap:** legacy S1192-rich files are often excluded from Checkstyle entirely
(file-level `<excludes>` in the module pom) — that's *why* the duplicates accumulated. Sonar scans them
regardless, so the fix is valid. A reviewer may ask to un-exclude the file — do NOT do it in an S1192
PR (enables the full ruleset, surfaces massive unrelated legacy debt); reply with the violation count
and offer a separate PR. A class-level `@SuppressWarnings("checkstyle:MultipleStringLiterals")`
suppresses only checkstyle, NOT Sonar's S1192 — still fixable.

## Pure-simplification rules — best batch fodder (no dataflow check)

Behaviour-preserving with NO use-verification. Oldcore often holds 40-90; else thin-spread across leaf
modules (a ~10-module reactor of 3-5 each clears the target).
- `S1125` redundant boolean literal: `x == true`→`x`, `x == false`→`!x`; ternary shapes `cond ? x :
  true`→`!cond || x`, `cond ? x : false`→`cond && x`, `cond ? true : y`→`cond || y`, `cond ? false :
  y`→`!cond && y` (operand may be a boxed Boolean that autounboxes — fine).
- `S1488` inline an immediately-returned local.
- `S1858` drop `toString()` on a String receiver (TRUST the rule).
- `S2864` `keySet()`+`get(k)` → `entrySet()`; prefer `values().forEach(...)` when key unused, else the
  `entrySet()` enhanced-for (required when the key IS used, or the body throws checked / uses
  `continue`/`break`/mutates an outer local). `Map.Entry` needs no import.
- `S1612` `x -> obj.foo(x)` → `obj::foo`; also block-body `() -> { obj.foo(); }`, ctor `s -> new
  Foo(s)`→`Foo::new`, `x -> x instanceof Foo`→`Foo.class::isInstance`, enum `v -> v.name()`→`Enum::name`,
  qualified super. **Import gotcha (build-breaker):** a method ref names its target TYPE, which the
  lambda never needed imported — if that type is a NESTED class or the stream element type and isn't
  imported, build fails `cannot find symbol`; add the import. (`Type.class::isInstance`/`::cast` need NO
  new import.)
- `S1155` `size()>0`/`==0` → `!isEmpty()`/`isEmpty()`.
- `S1126` if-then-else returning boolean literals → single return: `if (c) {return true;} else
  {return false;}` → `return c;`; the `false`/`true` shape → `return !c;`; the equals-style tail
  `if (!c) {return false;} ... return true;` also collapses to `return c;`. When the flagged condition
  returns `false` you NEGATE it — De Morgan a multi-part `||` (`!(A||B||C)` → `A' && B' && C'`), wrapping
  a >120 result onto a `+4` continuation line. STRUCTURAL (multi-line old-string) → use the assert-guarded
  script with the exact block, never `replace_all`. A `// comment` between the `if` and the final `return`
  survives above the merged return (same as S1066). Concentrated in oldcore (equals()/boolean getters) —
  a DIFFERENT rule from any open oldcore S6201 PR, so oldcore stays fair game for it.

## Pure-syntax/annotation group — S1128 / S1197 / S1116 / S1161 / S1611 / S1124 / S3878 (safest fodder)

Zero dataflow, deep regenerating pools; a wide reactor cleanly satisfies the override. Apply by line
number in one assert-guarded script (see General techniques).
- `S1128` unused import: delete the flagged `import ...;` (assert it starts `import`, ends `;`). TRUST
  Sonar (it keeps `{@link}`-only imports). A delegating subclass can legitimately have 10-14 removable.
- `S1197` array designator: `TYPE NAME[]`→`TYPE[] NAME`. **Regex gotcha:** a naive `\b(\w+)\s+(\w+)\[\]`
  wrongly grabs a return type already in `[]` form when a modifier precedes it. Use negative lookahead
  `\b(\w+)(\s+)(\w+)\[\](?!\s*\w)` → `\1[]\2\3` (a `[]` followed by ` identifier` is already correct).
- `S1116` empty statement: lone `;` (delete line); trailing `;;` (strip one); `};` where `}` closes a
  block/method (strip the `;`). NEVER strip `;` from `new Foo(){...};` / `Type x = ...{...};` (required;
  Sonar won't flag those). Distinguish by exact line number.
- `S1161` missing `@Override`: insert a line `<indent>@Override` ABOVE each flagged signature (purely
  additive). Deep pool, often concentrated in TEST files (anonymous-class methods). TRUST Sonar; assert
  the line has `(` and neither it nor its predecessor is already `@Override`. Interface methods
  redeclaring a super-interface method legitimately take `@Override` (Java 6+).
- `S1611` redundant lambda-param parens: `(x) -> ...` → `x -> ...` (single untyped param only). Thin-
  spread across modules; ideal filler to bundle into any batch. Match a UNIQUE token — `(x) -> {` or
  `(x) -> body`, not the bare `(x)` (which recurs); a `.thenAnswer((invocation) -> ...)` shape is common
  in Mockito test setup. Assert the per-file count (a file can hold >1 flagged lambda with the same body).
- `S1124` modifier order: reorder the LEADING modifier run to canonical JLS order (public/protected/
  private → abstract → static → final → transient → volatile → synchronized → native → strictfp). Almost
  always `final static`→`static final` or `static public`→`public static`. FULLY scriptable: regex-consume
  the leading run of modifier keywords, sort by canonical index, keep the type+rest verbatim. Zero
  behaviour/visibility change → NO `@since` even on public constants. Usually ZERO open PRs (cleanup waves
  skip it); dense in oldcore + spread across many leaf modules — a solid unclaimed backbone batch.
- `S3878` arrays created for varargs: remove the `new T[]{...}` wrapper and pass the elements. TWO message
  variants, both usually clean spreads: "Remove this array creation / and simply pass the elements" AND
  "Disambiguate by casting as Object/Object[]" (the latter is typically `MessageFormat.format`, SLF4J
  `LOGGER.x`, `Arrays.asList`, or reflection `getMethod`/`getConstructor`/`newInstance` — spreading is
  equivalent and preferred). DROP the genuinely ambiguous ones: an EMPTY-array delegation `foo(x, new
  Object[]{})` (overload-resolution / self-recursion risk) and a single `new Object[]{y}` where `y` could
  itself be an array. Multi-line arrays: edit the open line (drop `new T[]{`) and the close line (drop one
  `}`), then normalize any over-indented continuation to +4. Concentrated in oldcore + legacy.

## Utility-class / dead-code family — S1118 / S1144 / S1185 (fresh pivot when S6201/S6204 saturated)

When the small mechanical pools AND S6201/S6204/S1066 are all simultaneously drained or PR-saturated
(the common concurrent-session state), this trio is a reliable unclaimed pivot — deep MAJOR pools,
cleanup waves rarely touch them, they regenerate, and they're near-always zero open PRs. **oldcore is
the densest source and is FAIR GAME** for them even when an open oldcore PR exists, as long as that PR
is a *different* rule (the recurring open oldcore PR is S6201-only). All three are additive/removal, so
they bundle into one multi-type PR over an oldcore-dominant reactor (one build). Verify each per below —
delegate the per-site reading to parallel general-purpose subagents over DISJOINT files, then cross-check
`git diff --name-only` against the expected set and grep each removed name is gone (General techniques).
- **`S1118`** add a `private` constructor to a utility class. Two variants: "Add a private constructor
  to hide the implicit public one" (class has NO ctor → insert `private Foo() { }` after fields/before
  first method, respecting DeclarationOrder; a short Javadoc is safe, private ctors don't strictly need
  it) and "Hide this public constructor" (class HAS a `public` ctor → change `public`→`private`).
  **DROP the "hide" variant when the ctor is actually instantiated** (`grep "new Foo("` across the
  reactor) — e.g. factory classes exposing instance methods, `new XxxFactory()` called from a service.
  Purely additive → the safest, highest-yield of the three; near-0 drops for the "add" variant.
- **`S1185`** remove an override whose body is ONLY `super.x(sameArgs)` (optionally `return`ed). Delete
  the whole method incl. Javadoc + `@Override`. DROP if it does anything else, changes
  return/throws/visibility meaningfully, or adds a behaviour-bearing annotation. Removing the sole
  method of a class ORPHANS its imports → clean them (word-boundary rule).
- **`S1144`** remove an unused `private` method (+ orphaned fields/imports). GOTCHAS: (1) **Hibernate/JPA
  reflective accessors** — a `getX`/`setX` on a persistent entity (e.g. `XWikiDocument`) may be mapped by
  property name in `*.hbm.xml`; grep the `.hbm.xml` mappings before removing any getter/setter, DROP if
  mapped. (2) skip serialization hooks (`writeObject`/`readObject`/`readResolve`/`writeReplace`). (3)
  **Removal CASCADES** — deleting the method can orphan a private helper it was the sole caller of (e.g.
  removing `localizePlainOrKey` orphaned `getLocalization()` + its field); trace and delete the whole
  dead chain, else you leave a fresh S1144. Process multiple methods in the SAME file highest-line-first.

## java:S1068 / S1481 / S1854 — unused-code removal (deep MAJOR pool)

~100+ open; sometimes ~45 in oldcore alone, else thin-spread (then a wide leaf-module reactor, and you
can EXCLUDE oldcore when all its hits are DROPs). Bundle `S1161` (@Override) for a clean multi-type PR.
Needs light dataflow judgement → delegate reading to ONE Explore subagent returning a precise action
per issue KEY (DELETE range / STRIP_PREFIX / INSERT_OVERRIDE / DROP+reason); apply by line number in
one assert-guarded script. Expect ~40 of ~45 after drops.
- **S1481 + S1854 fire as a PAIR** on the same `Type x = expr;` line — one edit clears both.
- Pure RHS → delete the whole decl line. Side-effecting RHS → KEEP the call as a bare statement, drop
  only the `Type name = ` prefix (`indent + line.split(' = ',1)[1]`). Must-keep side effects:
  `doc.addAttachment`/`newXObject` (mutates the doc the asserts check), `registerMockComponent`, a
  lazily-initializing getter, `velocityManager.getVelocityContext()`.
- `x = null` dead store in an `else` whose `if` returns → delete the whole `} else { x = null; }`. A
  trailing dead `timer++` → drop just the `++`, keep the read.
- **Removing a private LOGGER/field usually ORPHANS its import** → delete it too (orphaned-import rule,
  General techniques).
- Clean up what removal leaves: a COMMENT that solely described the removed line (or a field's javadoc);
  STRAY BLANK LINES (double blanks, trailing before `}`, leading after `{`). ASSERT each deleted blank
  is truly blank (`line.strip()==''`) — a hint can mis-point at the next member's `/**` opener.
- **Removal CASCADES:** deleting `T x = other.getFoo()` can newly-orphan `other` or a sibling local/block
  that only fed it. Sonar flags only the outermost; trace each removed RHS's inputs and delete the whole
  dead chain (pure getters only; keep side-effecting calls).
- A write-only field/var assigned in ONE place IS fixable (delete decl + that assignment; `@Override`
  setter's now-unused param is not re-flagged — S1172 skips overrides). **DROP:** assigned in SEVERAL
  places; exposed via a public setter (API); a dead store whose call must MOVE. ~10-15% residue.

## Other clean rules (re-query each run)

- **`S6204`/`S6211` `Stream.collect(Collectors.toList()/toSet())` → `Stream.toList()`/`.toSet()`** — a
  deep (~100 open) rarely-PR-touched pool; the GO-TO pivot when the mechanical/simplification/unused/
  S1066/S6201 families are ALL simultaneously PR-drained to 0 (a common concurrent-session state — verify
  with a facet query). Thin-spread across modules (a wide ~20-25-module reactor is fine — 59 sites in
  one green build, excluding the slow Solr/`-index` modules); aggregate a dense same-family reactor
  (e.g. the 3 livedata modules held 25). CAVEAT: `.toList()` is UNMODIFIABLE — convert only when the
  result is read-only (returned, iterated, `isEmpty`/`size`/`get`/`toArray`, or used as an `addAll`
  SOURCE) or passed to a non-mutating setter/ctor; DROP if it is later `add`/`set`/`remove`/`sort`/
  `removeIf`-ed or assigned to an `ArrayList`-typed target. Delegate the per-site dataflow read to
  subagents — general-purpose (not Explore), and verify their edits landed (General techniques). In
  test code, a list built only for `assertEquals`/iteration is a safe convert (near-0 drops overall).
  `.toList()` is 19 chars shorter than the original so line length never breaches. **Auto-derive the
  orphaned-import removal:** after converting a file's flagged lines, drop `import
  java.util.stream.Collectors;` (or the `import static java.util.stream.Collectors.toList;` variant,
  which also shows up as `.collect(toList())`) iff `Collectors.`/`toList(` no longer appears in the
  body (KEEP when a sibling `Collectors.toSet/joining/toMap/groupingBy` survives) — reproduces the
  correct per-file REMOVE/KEEP with no manual bookkeeping.
- `S2093` try-with-resources: `R r = new ...(); try {...} finally { r.close() }` → `try (R r = new
  ...()) {...}`. ~half of hits are NOT real closes (push/pop, `reset()`, semaphore release, resource
  created mid-body) — verify the finally actually CLOSES an `AutoCloseable` declared just before `try`.
  Implicit `close()` throws `IOException`; if the surrounding catch is narrower and the method doesn't
  declare it, add a `catch (IOException)`. Removing `IOUtils.closeQuietly` may orphan the `IOUtils` import.
- `S2119` reuse Random: extract `new Random()`/`new SecureRandom()` to a `private static final` field.
- `S1143`+`S1163` (a `finally` throws, masking the try's exception): fire as a PAIR. Replace the `throw`
  in the finally with `logger.warn("Failed to close ...: [{}]", ExceptionUtils.getRootCauseMessage(e))`.
  Wrap EVERY placeholder in `[{}]`. If an XWiki `@Component` with no logger, add `@Inject private Logger
  logger;` (org.slf4j, NOT static) + the ExceptionUtils import (import order: java/javax, then org.*
  alphabetical — slf4j between apache and xwiki). Grep a sibling for the existing message style.
- `S5361` replaceAll→replace: convert only when the first arg reduces to a literal (no regex metachars,
  or a single escaped one `"\\+"`→`"+"`) AND the replacement has no `$`/`\`.
- `S2147` combine catch clauses with identical bodies: `catch (A e){B} catch (B e){B}` → `catch (A | B
  e){B}`. Gotchas: (1) if the types are in a SUBTYPE relationship, multi-catch is a COMPILE ERROR —
  DELETE the redundant subclass catch instead. (2) Deleting a catch can ORPHAN that exception's import.
- `S3626` remove redundant jump — clean ONLY for a TRULY-trailing jump (last statement of a void
  method; a branch-final `continue`/`return` when the if-else chain is the WHOLE loop body). DROP: the
  jump is the branch's only statement (removing → EMPTY block, checkstyle `EmptyBlock`); complex nested
  try/catch/finally.

## Test-code rules — deep clean pool once syntax/simplification/unused are drained

Pure test-code edits (production untouched, low review risk). The module's tests now RUN (no
`-DskipTests`), so these edited tests are exercised directly; Checkstyle also runs in `install`.
- **`S5786` JUnit5 test class/method should be package-private — the best test-rule batch.** Two
  message variants (method-level "Remove this 'public' modifier", class-level "Remove redundant
  visibility modifiers..."). Do NOT infer scope from the message/line (the method-level message often
  points at the CLASS decl). Key the fix BY FILE: strip the leading `public ` (`re.sub(r'\bpublic\s+',
  '', line, count=1)`) from (a) the class-decl line (incl. nested/`@Nested`) and (b) every method whose
  immediately-preceding contiguous annotation block contains a REAL JUnit annotation (`@Test`/
  `@BeforeEach`/`@AfterEach`/`@BeforeAll`/`@AfterAll`/`@ParameterizedTest`/`@RepeatedTest`/`@TestFactory`/
  `@TestTemplate`/`@Nested`). Keep other modifiers (`@BeforeAll public static void`→`static void`). Do
  NOT touch fields, unannotated helpers, or methods with an XWiki-specific (non-JUnit) lifecycle
  annotation (`@BeforeComponent`/`@AfterComponent` stay public). A class-level flag makes the script
  strip ALL that file's test methods, so a dense file yields far more removals than its issue count —
  expected. **Cross-module compile check** (oldcore publishes a widely-used test-jar): for each class
  made package-private, `grep -rl "extends <Class>" --include=*.java xwiki-platform-core | grep -v
  <thisModule>` — the risk is only `abstract`/base test classes (a class NAMED `Abstract*Test` is often
  not abstract — read the decl). Open S5786 PRs usually sit on SIBLING modules (distinct, no file
  overlap) — scope off-limits by exact module path.
- **`S5785` use assertEquals/assertNotEquals/assertNull/assertSame — fully SCRIPTABLE by line number**
  (the message names the exact target). **Default RECEIVER-FIRST for `.equals()` shapes (universally
  safe):** `assertTrue(a.equals(b))`→`assertEquals(a, b)`, `assertFalse(a.equals(b))`→`assertNotEquals(a,
  b)` — JUnit's `assertEquals(expected, actual)` calls `expected.equals(actual)`, so keeping the
  original receiver first reproduces the exact call (correct even for custom/asymmetric equals or a
  different-typed `b`). Never flip operands (a flip caused a real test failure). Other shapes:
  `assertTrue(LIT == x)`→`assertEquals(LIT, x)`; `assertTrue(x != LIT)`→`assertNotEquals(LIT, x)` (covers
  `hashCode() != 0`); `assertTrue(null == x)`→`assertNull(x)`; degenerate `assertFalse(x.equals(null))`/
  `assertTrue(x.equals(x))` convert too (JUnit uses `Objects.equals`). **`==`/`!=` between REFERENCES** is
  a distinct message → `assertSame`/`assertNotSame` (order cosmetic; TRUST the message). Only convert
  FLAGGED lines (`assertTrue(x instanceof Y)` siblings stay). A message arg moves to the end. **Imports:**
  add the new static imports (alphabetical), remove `assertTrue`/`assertFalse` only when the file no
  longer uses them (grep after). Duplicate-line gotcha: identical assert lines can recur — a
  `count==1` assert then trips, diagnose. Already-half-fixed site: a flagged line directly above an
  existing equivalent → DELETE the redundant flagged line. **Run the changed test classes as the
  operand-correctness net:** `install ... -Dtest=<flagged classes> -DfailIfNoTests=false`. Since ONLY
  test code changed, the flagged classes are exactly the tests that can break — this RUNS them, it does
  not skip tests (Checkstyle + full test-compile still run, ~5s even in oldcore). For production-code
  rules never narrow like this: run the whole edited module (see Building / verifying).

## Find-phase cost

- Inline `sed`/`Read` (offset/limit) of ONE candidate region is cheaper than an Explore subagent for
  mechanical rules; use a subagent only when you must read & reject several candidates. Always trim
  `issues/search` JSON through `python3`/`jq` (keep key,rule,component,line,message) — some rules attach
  huge `flows`/`locations`; never dump raw responses into context.
- Component key = `groupId:artifactId:path` (TWO colons). Path = `component.split(':')[-1]`. Read
  locally at `/home/user/xwiki-platform/<path>`; never fetch file contents remotely.

## Building / verifying

- Checkstyle + Spoon run in `install` (that's what catches the Sonar issues). **Run the tests too —
  never `-DskipTests` (or `-DskipITs`).** Even a mechanical Sonar fix can change runtime behaviour, and
  the edited modules' unit tests are what reveal a regression the compiler can't. If a test fails
  because of your change, fix the change — or drop the issue and pick another; never skip tests to go
  green. Under the background-build discipline in **Cost control** this is a TIME cost, not a token cost.
- **Build ALL affected modules in ONE reactor:** `mvn clean install -B -ntp -pl m1,m2,... -Plegacy,snapshot
  -Dxwiki.revapi.skip=true -Dxwiki.surefire.captureconsole.skip=true`. Maven sorts by
  dependency order; missing transitive deps resolve from the remote snapshot repo (`-Psnapshot`).
  `-Plegacy` is required to include `*-legacy-*` modules.
- **Snapshot-repo lag after a version bump:** right after master's "prepare for next development
  iteration" commit, the remote snapshot repo may not yet hold EVERY sibling at the new `X.Y.0-SNAPSHOT`,
  so a `-pl` build fails with `Could not find artifact org.xwiki.platform:<sibling>:jar:X.Y.0-SNAPSHOT`
  (a resolution error, NOT your code). Fix: re-run the FAILED module with `-am` (also-make) so Maven
  builds the missing local siblings from source — the already-installed modules from the first run are
  reused, so this is cheap and only the failing sub-tree rebuilds.
- **Always run mvn from the repo root** (`cd /home/user/xwiki-platform && mvn ...`): the shell cwd can
  silently reset to `/home/user` between turns; a `-pl <relative>` build from the wrong cwd fails fast
  with "Could not find the selected project in the reactor" (a path error, not code — relaunch from root).
- Rough datapoints (build only; running the modules' unit tests adds substantially more time —
  oldcore's suite is large): small leaf modules ~10-45s warm; oldcore ~3.5 min warm / ~6.5 cold;
  feed-api ~5 min. Pick the smallest leaf modules; avoid Solr submodules and feed-api. Now that tests
  run, prefer a few dense modules over a wide reactor — fewer test suites to execute.
- Run the build in the **background**, letting the tool capture stdout to its own `tasks/<id>.output`.
  Do NOT add your own `> build.log` redirect, do NOT `nohup … &`, NO `| tail`. The completion
  notification carries the exit code; ONE grep for `BUILD SUCCESS` afterwards confirms.

## Cost control (the build wait dominates the bill — but in TIME, not tokens)

- The token bill is driven by how many TURNS happen while the build runs, NOT how long it takes.
  Running the tests (see Building / verifying) makes builds slower in wall-clock but costs ~the same
  tokens, PROVIDED you keep this discipline: once the build is running, STOP — one line, end the turn.
  Do NOT Read/grep the log while it runs; the completion notification wakes you (one wake-up turn
  re-reads the cached context once, whether the build took 3 min or 25). Don't arm a short
  `ScheduleWakeup` for a build; if you want a fallback use 1200s+.
- Tests add real tokens in exactly one case: a test FAILS and you must diagnose/fix/rebuild — which is
  the regression you WANTED to catch. On clean mechanical fixes this is rare.
- The stop-hook ("uncommitted changes") firing while you wait on the build is EXPECTED — do NOT commit
  before the build verifies.
- **A container restart can kill the background build mid-run.** Working-copy edits PERSIST
  (uncommitted) — don't re-do the fix; check `git status`/`git diff`, confirm the branch, re-launch the
  same `mvn` build. Don't panic-commit unverified.

## GitHub (`gh` is NOT available — use the GitHub MCP tools)

- Check existing agent PRs once up front with `search_pull_requests` (`is:pr is:open label:llm-agent
  repo:xwiki/xwiki-platform`). Do NOT use `list_pull_requests` (~660KB). The `search_pull_requests`
  result can itself exceed the token limit → parse it from the saved file with python.
- Create the PR with `create_pull_request` (base `master`); add the label + assignee with `issue_write`
  (method=update, `labels:["llm-agent"]`, `assignees:[...]`). Include the SonarCloud issue link(s) in
  the body. (Vincent's override: assign the PR to him — GitHub user `vmassol`.)
- Creating the PR auto-subscribes the session to PR webhooks. XWiki CI is Jenkins and reports later, so
  `get_status` is `pending`/`total_count:0` right after creation — NOT a failure. Webhooks don't deliver
  CI-success / new-push / merge-conflict transitions; for long watches schedule a ~1h self check-in and
  re-arm silently until merged/closed. Reply to reviewers only when genuinely necessary.

## Process / conventions

- Commit + PR title (no JIRA): `[Misc] <desc; mention SonarCloud/SonarQube>`. Security issues: keep the
  description cryptic (public logs).
- **Author override:** `git config user.email <email>` AND `git commit --author="Name <email>"` AND a
  `Co-Authored-By: Name <email>` trailer — verify with `git log -1 --format='%an <%ae>'`. (This
  routine's override email differs from the git userEmail context — use the override.)
- **Reset the designated feature branch to master FIRST — it persists across runs.** `git fetch origin
  master` then `git checkout -B <branch> origin/master` before editing, or the new PR bundles old
  already-merged commits. (A stale local `origin/master` hides this — always fetch before judging.)
- **Recording learnings (memory repo → `main`):** the xwiki-platform fix lives on a feature branch but
  learnings go to `main`. Do NOT edit on the feature branch then stash/checkout/pop (main has diverged;
  the pop bakes `<<<<<<<` markers into the commit). Instead `git checkout main && git pull origin main`
  FIRST, then edit and commit directly on main.

## Token-cost report (when asked)

- Parse the transcript jsonl at `/root/.claude/projects/<id>.jsonl`. Dedupe assistant turns by
  `message.id` keeping the LAST record per id (streamed partials accumulate; the tool_use block often
  appears only in a later partial), then sort by timestamp. Find phase boundaries by tool_use NAME
  (not string-match on content). Buckets: (1) find = up to first fix edit; (2) fix = edit + build +
  commit + push; (3) post = PR + label + accept + report. Cache-read dwarfs everything. Add the report
  to the PR **body**, not a comment.
