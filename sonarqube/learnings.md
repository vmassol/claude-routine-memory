# SonarQube fix — learnings

Generic, reusable playbook for the "fix one (or a batch of) SonarQube issues in xwiki-platform"
routine. Keep this compact and generic: record techniques and gotchas, NOT run history. When you
learn something, merge it into the right section and trim — don't append dated anecdotes or PR logs.

## Picking a target rule (find phase)

- Get the rule distribution cheaply FIRST (no issue bodies), restricted to a mechanical allowlist:
  `.../api/issues/search?...&issueStatuses=OPEN&severities=BLOCKER,CRITICAL&rules=java:S1192,java:S5361,java:S2093,java:S1481,java:S1854,java:S2147&facets=rules&ps=1`
- **Counts shift every run as PRs land, and clean rules get EXHAUSTED then slowly REGENERATE — always
  re-query, never assume a rule still has convertible issues.** If a rule's remaining issues are all
  the non-convertible residue, pivot to another rule (skill rule: "if a fix is hard, drop it and pick
  another"). `java:S1192` is the reliable perennial fallback (see below).
- **The BLOCKER/CRITICAL mechanical pool is frequently exhausted** — S1192 tiny (see feasibility
  check), S2093 often ALL residue. When it is, drop the severity filter: there is a deep MAJOR-severity
  clean pool for single-issue fixes. Reliable MAJOR winners (re-query counts): `java:S1155`
  (`size()>0`/`==0` → `!isEmpty()`/`isEmpty()`), `java:S1068`/`java:S1481`/`java:S1854` (unused
  field/var, dead store), `java:S2864` (iterate `entrySet()` not `keySet()`), `java:S1858` (pointless
  `toString()` on a String), `java:S1612` (lambda → method reference), `java:S1488` (inline
  return-of-temp), `java:S1125` (redundant boolean literal). Starting BLOCKER/CRITICAL is the skill's
  guidance but is not a hard gate — a clean MAJOR fix beats forcing a risky higher-severity one.
- **A recent open agent PR can drain a WHOLE rule family, not just single issues.** Before committing
  to a rule, read the open `llm-agent` PRs' titles/bodies — if one already batched e.g. S1488/S1125/
  S1612/S2864/S1858, those issues are still OPEN in Sonar (PR unmerged) but OFF-LIMITS; pivot to an
  untouched family. The safest untouched fodder is the **pure-syntax trio** (own section below):
  `java:S1128` (unused import), `java:S1197` (array designator), `java:S1116` (empty statement) —
  zero-dataflow, even safer than the simplification rules, and they regenerate.
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

`java:S1125` (`x == true`/`== false` → `x`/`!x`), `java:S1488` (inline an immediately-returned local),
`java:S1858` (drop `toString()` on a value already a `String` — TRUST the rule, it only fires on String
receivers), `java:S2864` (`keySet()`+`get(k)` → `entrySet()`; prefer `values().forEach(v -> ...)` when
the key is unused, else the `entrySet()` enhanced-for `for (Map.Entry<K,V> e : m.entrySet())` — required
whenever the key IS used or the body throws a checked exception / uses `continue`/`break`/a mutated
outer local; `Map.Entry` needs no import), `java:S1612` (`x -> obj.foo(x)` → `obj::foo`; also fires on
block-body `() -> { obj.foo(); }`, constructor `s -> new Foo(s)` → `Foo::new`, `x -> x instanceof Foo`
→ `Foo.class::isInstance`, qualified super `() -> Outer.super.foo()` → `Outer.super::foo`; overloaded
targets and `assertThrows(..., obj::method)` clusters in tests are fine). These are behaviour-preserving
with NO use-verification, unlike the removal rules `java:S1481`/`java:S1854`/
`java:S1068` (must confirm the var/field is truly unused and its RHS has no side effect). For a large
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

## Pure-syntax trio — S1128 / S1197 / S1116 (zero-dataflow, the safest batch fodder)

Even safer than the simplification rules (no use/side-effect verification at all) and a deep,
regenerating pool. A single batch of all three across ~20 leaf modules (one reactor) cleanly satisfies
the 20-50 override; build ~13 min with oldcore in the set. Exclude a heavy single-issue module (e.g.
feed-api ~5 min for 1 issue) — fix "almost all", not literally all, for ROI. Apply all three in one
atomic Python script keyed BY LINE NUMBER (the repo is at the exact master commit Sonar scanned, so
lines don't drift); assert each target line's shape before editing and write nothing if any assert
fails; process each file's edits mapping over ORIGINAL indices so a deletion doesn't shift later ones.

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

## Find-phase cost

- Inline `sed`/`Read` (offset/limit) of ONE candidate region is cheaper than an Explore subagent for
  mechanical rules; use a subagent only when you must read & reject several candidates. Always trim
  `issues/search` JSON through `python3`/`jq` (keep key,rule,component,line,message) — some rules
  attach huge `flows`/`locations`; never dump raw responses into context.
- Component key = `projectKey:path`; strip the prefix and read locally at
  `/home/user/xwiki-platform/<path>`. Never fetch file contents over a remote API.

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
- Accept all issues in one loop (per issue: `add_comment` + `do_transition` accept). Each issue is 2
  curls ≈ 1.4s, so 43 already blows a 2-min foreground timeout — run the accept loop as a background
  task for 20+ issues. **The `do_transition` response does NOT reliably contain an `issues` key** — don't
  `json.load(...)['issues']` (KeyError); the transition still applied. Confirm separately with one
  `issues/search?issues=<comma-keys>` → all ACCEPTED/RESOLVED.

## Building / verifying

- Checkstyle + Spoon run in `install` (that's what catches issues); `-DskipTests` is fine.
- Pick the smallest leaf module(s). Rough datapoints: small plugin/notifier leaf modules ~10s each in
  a warm reactor; oldcore ~3.5 min warm / ~6.5 min cold; feed-api ~5 min.
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
- **Recording learnings (memory repo → `main`):** the xwiki-platform fix lives on a feature branch but
  learnings go to `main`. Do NOT edit on the feature branch then stash/checkout/pop (main has diverged;
  the pop conflicts and can bake `<<<<<<<` markers into the commit). Instead `git checkout main &&
  git pull origin main` FIRST, then edit and commit directly on main.

## Token-cost report (when asked)

- Parse the transcript jsonl at `/root/.claude/projects/<id>.jsonl`. Dedupe assistant turns by
  `message.id` but **keep the LAST record per id** (streamed messages accumulate under one id; the
  tool_use block, e.g. the PR call, often appears only in a later partial). Sort deduped entries by
  timestamp before scanning for boundaries.
- **Find phase boundaries by tool_use NAME, not string-match on content** (the literal string
  `"create_pull_request"` appears in early schema-loading text → false boundary). Match
  `b["name"]=="mcp__github__create_pull_request"` and the Edit of the target file. Verify boundary
  timestamps are monotonic.
- Buckets: (1) find = up to first fix edit; (2) fix = edit + build + commit + push; (3) post = PR +
  label + accept + report. Cache-read dwarfs everything; keep reads narrow. Add the report by editing
  the PR **body**, not a comment, if the routine asks for it.
