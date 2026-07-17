# java:S1192 — define a constant for a duplicated literal

> Load only when fixing S1192. Cross-cutting mechanics live in `learnings.md` → *General
> batch-fix techniques*.

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
