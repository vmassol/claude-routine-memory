# SonarQube fix — learnings

Generic, accumulated learnings for the "fix one SonarQube issue in xwiki-platform" routine.
Merge & trim — keep this compact; don't just append.

## Picking a target rule (find phase)

- Get the rule distribution cheaply first (no issue bodies), restricting to the mechanical allowlist:
  `.../api/issues/search?...&issueStatuses=OPEN&severities=BLOCKER,CRITICAL&rules=java:S5361,java:S1481,java:S1854,java:S2093,java:S2147&facets=rules&ps=1`
- **`java:S2093` (try-with-resources) clean fixes are now EXHAUSTED (2026-07-01).** All the genuinely
  convertible ones were fixed in prior runs; the 11 still-open (all CRITICAL) are the residue and are
  ALL non-convertible (verified 0/11: `pop()`, `setWikiReference()`, cache `put()`,
  `removeAttribute()`, semaphore release, `stream.reset()`, resource created mid-body). S5361/S1481/
  S1854/S2147 are also EXHAUSTED (0). **So don't force S2093** — Vincent's "fix 5 S2093" override is
  conditional on convertible ones existing; when none do, pivot to another rule and fix ONE (skill:
  "if a fix is hard, drop it and pick another"). Re-check S2093 counts each run in case new ones land.
  Conversion pattern (for when convertible ones reappear): `Resource r = new ...(); try { ... } [catch]
  finally { r.close()/IOUtils.closeQuietly(r) }` → `try (Resource r = new ...()) { ... } [catch] }`.
- **`java:S1192` (define a constant for a duplicated literal) is the go-to clean fallback
  (~54 open BLOCKER/CRITICAL as of 2026-07-07, dropping fast as PRs land — down from ~81 on 07-06).**
  The fat oldcore single-file clusters (XWiki.java etc.) are now largely DRAINED, so Vincent's "fix
  20-50 in one PR" override now means **combining several files across a few SMALL leaf modules** in
  one reactor build (no longer oldcore-only). Query `&rules=java:S1192&ps=500`, group by `component`,
  pick the fattest files and prefer small leaf modules (cheap build). Good clusters found 2026-07-07
  (PR #5787, 21 fixed): `MimeTypesUtil` 9 (mailsender), `FeedPlugin` 6 (feed-api), `SchedulerPlugin` 4
  + `SchedulerPluginApi` 2 (scheduler-api) — 3 small modules, ~5-min reactor build. Fix: add
  `private static final String NAME = "literal";` and replace every occurrence in that ONE file.
  VERIFY by counting the quote-bounded literal
  (`content.count('"admin"')`) — line numbers drift, and quote-bounding avoids substring hits (`"admin"`
  never matches `"administrator"`, `"login"` never matches `"loginerror"`, `"delete"` never matches
  `"undelete"`). **When quote-bounded grep != Sonar's stated N, DIAGNOSE before dropping — the excess
  is often FIXABLE:** grep the occurrences with line numbers and classify.
  (a) **Excess is in COMMENTS/javadoc** (whole-token in prose, e.g. `status "Paused"`,
  `$xcontext.get("error")` inside a `* #error(...)` javadoc line) → SAFE and fixable: replace only on
  non-comment lines (skip lines whose `lstrip` starts with `*`/`//`/`/*`) and assert the
  CODE-occurrence count == N. Did exactly this 2026-07-07: SchedulerPlugin `"Paused"`/`"Normal"` grep
  4 vs N 3 (1 javadoc each), SchedulerPluginApi `"error"` grep 11 vs N 9 (2 javadoc) — all fixed.
  (b) **Excess/shortfall is a substring/fragment mismatch** (grep==0 because Sonar counts a
  concatenation fragment that isn't a standalone `"..."` token, e.g. `"groovy_missingrights"` grep=0;
  or grep>N because your token is a substring of a longer literal Sonar doesn't count) → NOT fixable,
  DROP it (2026-07-05: XWikiDocument `"reference"`/`"inline"`/`"document"`, SaveAction
  `"previousVersion"`, XWikiServletURLFactory `"UTF-8"`). So: comment-inflation = skip-comments-and-fix;
  fragment mismatch = drop. Aim for a bit more than the target so 20-50 still clears after any drops.
  Match the class's constant style;
  consecutive `private static final String` decls with NO blank line between them are fine (see
  TextAreaClass/NumberClass) — no per-field blank line or javadoc needed for private constants.
  **Insertion / formatting (script gotcha):** two clean strategies, both build-verified:
  (a) if the class HAS an existing private static-final field, anchor to it and insert right after
  (blank line between anchor and first constant; if the anchor was followed by a blank line you now
  have a triple `\n` — collapse it; verify `\n\n\n` count unchanged);
  (b) if the class has NO existing constant, insert the new `private static final String` as the
  FIRST member right after the class-body opening brace (find the first `{` at/after the class-decl
  line — this correctly handles multi-line `implements X, Y` declarations) with a trailing blank
  line. XWiki checkstyle does NOT enforce public-before-private field order — a private constant
  inserted ABOVE existing public constants (XWikiServletRequest, 2026-07-06) built fine, as does
  XWikiDocument's private LOGGER sitting above public `CKEY_*`. 24-literal
  batch across 6 oldcore files built clean in ONE ~5-min oldcore build (PR #5771, 2026-07-05);
  27-literal batch across 12 oldcore files (XWiki + 11 others, mixing both strategies) in ONE
  ~6-min build (PR #5779, 2026-07-06) — the count==N check dropped 3 XWiki.java literals
  (`default` 4≠3, `backlinks` 4≠3, `groovy_missingrights` grep=0) cleanly before any build.
  **THE forward-reference / declaration-order gotcha (still real when a literal is used in a
  static-field initializer):** Java forbids referencing a static field by simple name *before* its
  textual declaration in a static-field initializer (an "illegal forward reference" compile error).
  This bites STATIC ARRAY/collection initializers too: in `MimeTypesUtil` (2026-07-07) the literals
  live inside `protected static final String[][] MIME_TYPES = { {CONST, "bin"}, ... }`, so the 9 new
  constants MUST be declared textually BEFORE `MIME_TYPES` — inserted them right after the class-opening
  brace (private constants sitting ABOVE the `protected` array; checkstyle accepted private-before-protected,
  consistent with its not enforcing public-before-private). When the literal is only used in *method
  bodies* (the common case), declaration order is irrelevant — insert anywhere among existing private
  constants. When checkstyle DeclarationOrder DOES bite (public
  before private) in a given class, insert the constant
  block AFTER the public fields but BEFORE the first (private) field that uses it (e.g. right
  before/after `LOGGER`). If a literal ALSO appears in a *public* static-field initializer sitting
  above your block, DON'T replace those occurrences — leaving ≤2 literal copies still resolves the
  issue (rule fires at ≥3). (Did exactly this for `"XWiki"` in XWikiRightServiceImpl: converted 3 of
  5, left the 2 in public fields.) The **"use already-defined constant" S1192 variant is frequently
  UNFIXABLE** for the same reason: the existing constant is declared *after* the public field that
  duplicates it (XWikiGroupServiceImpl: `CLASS_SUFFIX_XWIKIGROUPS`/`DEFAULT_MEMBER_SPACE` at lines
  73/103 but the dup is the public `GROUPCLASS_REFERENCE` at line 66) → skip it.
  **Script order matters:** do the literal→constant replacements FIRST (on original content,
  asserting each count), THEN insert the constant block — otherwise the replace pass rewrites your own
  new declarations into `ADMIN = ADMIN`. Other caveats: a class-level `@SuppressWarnings("checkstyle:
  MultipleStringLiterals")` suppresses only the *checkstyle* check, NOT Sonar's S1192 — still fixable;
  skip a second S1192 on the SAME line for a different literal (fix only the reported one). Did
  `DatabaseKeywordSearchSource` `"keywords"` (rest-server, PR #5762); 22-literal batch across
  XWikiRightServiceImpl/XWikiHibernateStore/XWikiAuthServiceImpl (oldcore, PR #5763, 2026-07-05);
  21-literal batch MimeTypesUtil+FeedPlugin+SchedulerPlugin+SchedulerPluginApi (3 leaf modules,
  PR #5787, 2026-07-07 — all-local-private-constants, no cross-class changes).
  **Reviewer preferences on S1192 (tmortagne, PR #5779 — apply pre-emptively to avoid review churn):**
  (1) A literal used ONLY in a log-message concatenation (`LOGGER.x(PREFIX + var + " ...")`) — do NOT
  introduce a `PREFIX` constant; instead convert the call to slf4j parameterized syntax
  `LOGGER.x("Prefix [{}] ...", var)` (bracket the placeholder, XWiki convention). This eliminates the
  duplicate literal entirely (no constant needed) AND is what reviewers want. BUT if the resulting
  parameterized message string ends up duplicated across ≥2 calls (e.g. two auth flows both logging
  `"User [{}] is authentified"`), extract ONE constant for the WHOLE message — a reviewer (vmassol,
  PR #5781) will flag even a 2× duplicate that's below Sonar's own ≥3 threshold. (2) If a literal is a
  **domain property/field name** (e.g. an `XWiki.XWikiUsers` class property — `email`/`password`/
  `validkey`), don't make a local private constant; add/reuse a **public** constant on the owning
  class (`XWikiUsersDocumentInitializer` for user-class fields — grep its `add*Field("...")` calls to
  identify which of your literals are its properties) and reference it. Before extracting a literal,
  ask "is this an entity property name that already has a home class?" — if so, REUSE the existing
  public constant if one exists (e.g. `XWikiUser.ACTIVE_PROPERTY`/`EMAIL_CHECKED_PROPERTY` already
  hold user-property names) rather than creating a duplicate; only add a new public constant when
  none exists, in the owning class (`XWikiUsersDocumentInitializer`). The `"XWiki"` system-space
  literal already has a home: reuse `com.xpn.xwiki.XWiki.SYSTEM_SPACE` (`= "XWiki"`) instead of a
  local constant (tmortagne, PR #5781). Watch the 120-char line limit
  after swapping a short local name for a `LongClassName.FIELD` reference — wrap the call.
  **BUT do NOT force owning-class-reuse in two cases (2026-07-07): (i) the owning constant is `private`
  in an `internal` package** — making it public just to reuse it exposes internal API (worse than a
  local constant), and **(ii) the qualified reference blows the 120-char limit** — e.g.
  `SchedulerJobClassDocumentInitializer.FIELD_JOBNAME` (+41 chars over `"jobName"`) pushed 14 lines to
  ~148 chars, needing mass wrapping. In both cases a LOCAL `private static final` constant is the
  cleaner call — and note the class often ALREADY has a local constant despite the initializer's
  existing one (SchedulerPlugin has local `XWIKI_JOB_CLASS` even though the initializer exposes
  `XWIKI_JOB_CLASSREFERENCE`), so local private constants are an accepted pattern. Local names are also
  usually SHORTER than the quoted literal (`JOB_NAME` < `"jobName"`, `STATUS` < `"status"`), so lines
  shrink and never overflow. (Also: pre-existing >120-char lines, e.g. HQL/SQL strings, are tolerated
  by the build — only worry about a line YOUR edit lengthens.)
  (3) **Any newly public API needs an `@since` javadoc tag** or a reviewer will flag it (tmortagne,
  PR #5780) — including a field you merely widened from private to public. Format = the reactor
  `project.version` with `-SNAPSHOT`→`RC1` (grep `pom.xml` `<version>`, e.g. `18.6.0-SNAPSHOT` →
  `@since 18.6.0RC1`; confirm the style by grepping recent `@since 18.` tags). Place it as the last
  javadoc line after a blank ` *` separator line. Add it PROACTIVELY when making a constant public.
- **Legacy files rich in S1192 are often EXCLUDED from Checkstyle** (file-level `<excludes>` in the
  module pom's `maven-checkstyle-plugin` `default` execution) — that exclusion is *why* the duplicate
  literals accumulated (Checkstyle's `MultipleStringLiterals` never ran on them). Sonar scans them
  regardless, so the S1192 fix is fine. **But a reviewer may ask you to "update the excludes
  accordingly" (i.e. un-exclude the file now that literals are fixed) — do NOT attempt it in an
  S1192 PR.** Un-excluding enables the FULL ruleset, surfacing massive pre-existing debt (2026-07-07,
  PR #5787: FeedPlugin alone = 264 violations — MissingJavadoc, complexity, DeclarationOrder,
  EmptyCatchBlock, plus more `MultipleStringLiterals` at Checkstyle's stricter ≥2 threshold that Sonar's
  ≥3 S1192 never flags). Reply with the concrete numbers explaining it's a separate large cleanup per
  file; keep `<excludes>` as-is. (To test: remove the file from `<excludes>` and run the normal
  `install` build — NOT `mvn checkstyle:check` from CLI, which uses the default sun_checks config, not
  XWiki's, and falsely passes.)
- **Prior clean fallbacks are now DRAINED (all 0 as of 2026-07-04):** `java:S2119` (reuse Random),
  `java:S1143`+`java:S1163` (finally throws). They may regenerate — re-query counts each run — but
  don't assume they're available. Historical CLEAN patterns kept for when they reappear:
  `java:S2119` — extract `new Random()/new SecureRandom()` to a `private static final` field
  (SecureRandom is thread-safe, shared static is safe; did `PasswordClass.randomSalt`).
  `java:S1143`+`java:S1163` (a `finally` block throws, masking whatever the `try` threw) — always
  fire in PAIRS on the SAME line (same defect, two rules); fix once, link+accept both keys in one PR.
  Fix pattern: replace
  the `throw new XyzException(...)` in the `finally`'s inner catch with `logger.warn("Failed to close
  ...: [{}]", ExceptionUtils.getRootCauseMessage(e))`. **XWiki logging convention: wrap EVERY
  parameter placeholder in square brackets `[{}]`, never a bare `{}`** (so value boundaries / empty
  values show in the log). Don't blindly copy a sibling's exact string — some existing lines omit the
  brackets on a trailing message; follow the `[{}]` convention regardless. Grep the SAME module for a sibling that already
  does this before inventing one (`XWikiAttachment#getContentInputStream` in oldcore;
  `WikiReader`/`AbstractReader` in xar filter-stream). Needs a logger field: if none exists and the
  class is an XWiki `@Component`, add `@Inject private Logger logger;` (org.slf4j.Logger) — NOT a
  static LOGGER — matching the component idiom; also add the `commons-lang3`
  `org.apache.commons.lang3.exception.ExceptionUtils` import. Mind checkstyle import ordering
  (java/javax, then org.* alphabetical: slf4j sits between apache and xwiki). AVOID:
  `java:S2447` (~10, null from Boolean method) — in XWiki *script services* returning null is a
  deliberate "check getLastError()" pattern; changing to false alters behavior — skip script
  services. `java:S1214` (~8, constants-in-interface) = cross-module refactor, skip. `java:S1113`
  (~2, override finalize) = needs Cleaner/PhantomRef refactor, skip. `java:S5845` (~2, assert
  dissimilar types) in tests can be subtle — runtime type may differ from the declared generic
  (erasure), so the assertion is actually correct; verify before "fixing". `java:S1215` ("remove
  this System.gc() call") — check the enclosing method isn't a deliberately-exposed public API (e.g.
  `XWiki#gc()` is called from Velocity as `$xwiki.gc()`); removing the actual call there breaks the
  API contract even though Sonar flags the call itself — skip these. `java:S2696` (static field
  written from an instance method) — often a lazy-init/memoization pattern (e.g. a
  `private static Map` populated on first call); a correct fix needs synchronization or a static
  initializer, not a one-liner — skip unless you're willing to do that refactor. `java:S2157`
  (class should implement `Cloneable`/add `clone()`) — non-trivial to do correctly, skip for
  quick fixes.
- **S2093 (when it regenerates): ~half the hits are NOT real resource closes** — Sonar fires on any
  try/finally it *thinks* could be try-with-resources (push/pop, `stream.reset()`, semaphore
  release, resource created mid-body, state restoration) that is NOT `AutoCloseable`. Verify the
  finally actually CLOSES an AutoCloseable declared just before `try` before converting.
- **Compile-safety check before converting:** the implicit `close()` throws `IOException`. If the
  surrounding catch only catches a *narrower* type (e.g. `FileNotFoundException`) and the method does
  NOT declare `throws IOException`, the converted code won't compile. Either the method already
  declares/catches `IOException` (fine), or add a `catch (IOException)` that wraps it sensibly (did
  this for `XWikiConfig(String)` — wrapped as config FORMATERROR). Check throws clause + catch types.
- **Removing `IOUtils.closeQuietly` may orphan the `IOUtils` import** → checkstyle failure. `grep -n
  IOUtils <file>` first; if it was the only use, delete the import line too.
- Done: `XarPackage.read` (2026-06-30); batch of 5 S2093 on 2026-06-30 — `Packager`, `XWikiExecutor`,
  `Importer`, `XWikiConfig`, `ZipExplorerPlugin`; `PasswordClass` S2119 (2026-07-01, oldcore, PR
  #5706); `XarPackage#write` S1143+S1163 (2026-07-02, xar-model, PR #5713);
  `XARInputFilterStream` S1143+S1163 (2026-07-03, xwiki-platform-filter-stream-xar, PR #5750 — a
  FRESH pair, confirming the rule regenerates); `DatabaseKeywordSearchSource` S1192 `"keywords"`
  (2026-07-04, rest-server, PR #5762). S2093 still 11/11
  non-convertible as of 2026-07-04 (unchanged) — re-verify count each run but don't re-triage the
  same 11. S2093 issues are each in a DIFFERENT module (no
  single-module cluster), so a 5-fix PR needs a multi-module reactor build (~8-9 min: oldcore ~5 min
  cold + packager-plugin ~2.5 min + 3 leaf modules <20s each). Feed the triage subagent ALL candidate
  file:line pairs in ONE message (forgot 7 the first time and had to do a 2nd round).
- **`java:S5361` (replaceAll → replace)** — kept for reference / the batch override. Convert when the
  first arg reduces to a literal (no regex metachars `. * + ? [ ] ( ) { } | \ ^ $`, OR an escaped
  metachar that is one literal char: `"\\+"`→`"+"`, `"\\."`→`"."`) AND the replacement has no `$`/`\`
  (specials only in `replaceAll`'s replacement). Char classes, `\s+`, backrefs, `;jsessionid=.*?` are
  genuine regex — Sonar does NOT flag them; fix only the reported lines, not every `replaceAll`.
- **Counts shift fast as prior PRs land — always re-query.** Rough denylist by rule: java:S3776
  (complexity), java:S3252 (API), java:S1186 (empty methods), java:S115 (naming) — avoid. java:S1192
  (constant) is the reliable OK fallback.
- Component key = `projectKey:path`; strip prefix, read locally at `/home/user/xwiki-platform/<path>`.
  Never fetch file contents over a remote API.
- **One-PR-per-issue mode:** prefer a file with a SINGLE issue of the rule (clean accept-step).
- **Don't trust Sonar's per-issue line numbers blindly** — they can drift from your working copy.
  Cross-check: `grep -n "\.replaceAll(" <file>` and match by the literal pattern, not the line. The
  count of OPEN issues in a file should equal the number you convert (clean cross-check).

## Find-phase cost

- Inline `Read` (offset/limit) of ONE candidate region is cheaper than an Explore subagent for
  mechanical rules. Use a subagent only when you must read & reject several candidates.
- Always trim `issues/search` JSON through `python3`/`jq` (keep key,rule,component,line,message) —
  some rules attach huge `flows`/`locations`. Never dump raw responses into context.

## Batch / "fix ALL of rule X" mode (e.g. Vincent's S5361 override)

- One `issues/search?rules=java:S5361&ps=100` gives every open issue + component + line. Group by
  file/module.
- **Apply scattered single-line fixes with a per-line Python script**, not sed (sed escaping of
  `\\+`/quotes is error-prone). For each `(relpath, 1-based line, old_substr, new_substr)`: assert
  `old_substr in lines[idx]` and fail loudly on mismatch before writing. This caught nothing wrong
  this run (19/19) precisely because grep gave working-copy line numbers. Delete the helper script
  before committing.
- **Build ALL affected modules in ONE reactor invocation**, not one-by-one:
  `mvn clean install -B -ntp -pl m1,m2,m3,... -Plegacy,snapshot -Dxwiki.revapi.skip=true
  -Dxwiki.surefire.captureconsole.skip=true -DskipTests`. Maven sorts them in dependency order;
  missing transitive deps resolve from the remote snapshot repo (`-Psnapshot`). 19 fixes across 5
  modules (oldcore + legacy-oldcore + query-manager + feed-api + notifications-notifiers-default)
  built in ONE ~8-min wait. **`-Plegacy` is required** to include `*-legacy-oldcore`.
- Accept all issues in one loop (each issue: add_comment + do_transition accept). 19×2 curls ≈ 45s
  (OK in foreground). For 50+ run it as a **background** task. **The `do_transition` response does NOT
  reliably contain an `issues` key — don't `json.load(...)['issues']` on it (KeyError). The
  transition still applied; confirm separately** with one
  `issues/search?issues=<comma-keys>` → all RESOLVED/WONTFIX.

## Building / verifying

- Checkstyle + Spoon run in `install` (that's what catches issues); `-DskipTests` is fine.
- Pick the smallest leaf module(s). Datapoints: notifications-notifiers-api ~3 min, oldcore ~3.5 min
  in a warm reactor (~6.5 min cold), feed-api/legacy-oldcore <1 min each.
- Run the build in the **background**, full output to a file (NO `| tail` — it breaks incremental
  grep). The completion notification wakes you with the exit code. **One layer of backgrounding only**
  — pass the bare `mvn …` to the run-in-background tool; do NOT also wrap in `nohup … &`.
- **Don't add your own `> build.log` redirect to a backgrounded mvn.** The run-in-background tool
  already captures stdout to its own `tasks/<id>.output` file; a `>` redirect leaves that file nearly
  empty, so grepping it for `BUILD SUCCESS` finds nothing and looks like a failure. Either grep the
  file you redirected to, or (simpler) drop the redirect and grep the tool's `.output` file.

## Cost control (the build wait dominates the bill)

- The fix phase is most expensive because it spans the build wait: every wait/poll/wakeup turn
  re-reads the full cached context (~50-65K each). Minimize those turns.
- Once the build is running, STOP: one line, end the turn. Do NOT Read/grep the log while it runs.
  The notification carries the exit code; ONE grep of the log afterwards confirms BUILD SUCCESS.
- Do NOT arm a short `ScheduleWakeup` (300s) for a background build — the completion notification
  already wakes you. If you want a fallback, use 1200s+.
- The stop-hook ("uncommitted changes") fires while you correctly wait on the build — expected, do
  NOT commit before the build verifies.
- **A container restart can kill the background build mid-run** (seen 2026-07-03). The working-copy
  edits PERSIST (uncommitted), so don't re-do the fix — just check `git status`/`git diff`, confirm
  the branch is still checked out, and re-launch the same `mvn` build. Don't panic-commit unverified.

## GitHub: `gh` is NOT available — use the GitHub MCP tools

- Check existing agent PRs once up front: `search_pull_requests` with
  `is:pr is:open label:llm-agent repo:xwiki/xwiki-platform`. Do NOT use `list_pull_requests` (~660KB).
- Create PR with `create_pull_request` (base `master`). Add label with `issue_write`
  (method=update, `labels:["llm-agent"]`). Ignore the skill's "open with gh".
- Creating the PR auto-subscribes the session to PR webhook activity. XWiki CI is Jenkins and reports
  later, so `get_status` is `pending`/`total_count:0` right after creation — NOT a failure. Webhooks
  don't deliver CI success/new pushes/merge-conflict transitions; for long watches schedule a
  ~1h self check-in (`send_later` if present, else `CronCreate` one-shot) and re-arm silently.

## Process / conventions

- Commit + PR title (no JIRA): `[Misc] <desc; mention SonarCloud/SonarQube>`.
- **Author override:** `git config user.email <email>` AND `git commit --author="Name <email>"`,
  AND a `Co-Authored-By: Name <email>` trailer — verify with `git log -1 --format='%an <%ae>'`.
  (This routine's override email differs from the git userEmail context — use the override.)
- Push with `git push -u origin <branch>`. Include the SonarCloud issue link(s) in the PR body.
- Security issues: keep PR/commit description cryptic (public logs).
- **Recording learnings (memory repo, pushed to `main`):** the routine's xwiki-platform fix lives on
  a feature branch but learnings go to `main`. Do NOT edit learnings.md on the feature branch then
  `git stash`/`checkout main`/`stash pop` — main has diverged, the pop conflicts, and a careless
  `add`+`commit` will bake `<<<<<<<` markers into the commit. Instead: `git checkout main &&
  git pull origin main` FIRST, then edit learnings.md directly on the freshly-pulled main and commit.

## Token-cost report (when asked)

- Parse the transcript jsonl at `/root/.claude/projects/<id>.jsonl`. Dedupe assistant turns by
  `message.id` — but **keep the LAST record per id, not the first**: streamed assistant messages are
  logged multiple times under the same `message.id` as content accumulates, and the tool_use block
  (e.g. the PR-creation call) often only appears in a later partial, not the first one. Deduping to
  the first-seen record silently drops it and boundary detection finds nothing. Sort the deduped
  entries by timestamp before scanning for boundaries.
- **Find phase boundaries by tool_use NAME, not string-match on content.** Matching the literal
  string `"create_pull_request"` in message content gives a FALSE early boundary (the name appears in
  early schema-loading text). Iterate `content[]` blocks, take `type=="tool_use"` and match
  `b["name"]=="mcp__github__create_pull_request"` (post boundary) and the Edit of the target file
  (find→fix boundary). Verify boundary timestamps are monotonic before trusting them.
- Buckets: (1) find = up to the first fix edit; (2) fix = edit + build + commit + push; (3) post = PR
  + label + accept + report (necessarily a mid-phase snapshot). Cache-read dwarfs everything;
  fresh-input/output stay tiny with narrow reads. Edit the PR **body** to add the report (don't
  comment) if the routine asks for it.
