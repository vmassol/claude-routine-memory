# Dropped issues — analyzed, decided NOT to fix (skip these on future runs)

Per Vincent's override: record every issue analyzed but not fixed, with the reason, so future runs
SKIP them instead of re-triaging. Keyed by SonarCloud issue key (stable across scans until the code
changes). When a listed file is later edited, its keys may go stale — that is harmless (the key just
won't match an open issue). Group by rule; keep the reason to one line. This is a skip-index, not run
history — merge/trim in place, don't append dated anecdotes.

**Whole rules** rejected during triage are NOT listed key-by-key here: the permanent ones are in the
OKF denylist and the recently-triaged ones in `pool-state.md` (currently S6213 rename-restricted-
identifier and S4144 identical-method-bodies). Check both before triaging individual keys of a rule.

Sections are per REPO — issue keys are unique per SonarCloud project, so a key from
`org.xwiki.commons:xwiki-commons` never collides with a platform one. Repos with no drops yet are
listed as such rather than omitted, so a future run knows the absence is real and not an oversight.

## xwiki-commons

(A 37-site `java:S7158` sweep, a 34-site `java:S6201` sweep of `xwiki-commons-xml` +
`xwiki-commons-filter-xml`, a 28-site `java:S7476`+`java:S3706` sweep, and a 105-site 14-rule sweep
of every non-`extension` module were each analyzed in full; only the sites listed below were dropped.)

### java:S3878 (varargs array) — SLF4J Logger delegation, spreading the array recurses infinitely
Every one of the 30 open commons S3878 issues is an SLF4J `Logger` override
`xxx(Marker, String, Object[, Object])` delegating to the varargs sibling via `new Object[] { … }`.
Removing the wrapper re-binds the call to the fixed-arity overload = the enclosing method. The whole
rule is a permanent DROP in `xwiki-commons-logging-*`.
- `logging-api` LogQueue (10): `AWgZSTmjUMkE2J58eTVe`, `AWgZSTmjUMkE2J58eTVf`, `AWgZSTmjUMkE2J58eTVg`,
  `AWgZSTmjUMkE2J58eTVh`, `AWgZSTmjUMkE2J58eTVi`, `AWgZSTmjUMkE2J58eTVj`, `AWgZSTmjUMkE2J58eTVk`,
  `AWgZSTmjUMkE2J58eTVl`, `AWgZSTmjUMkE2J58eTVm`, `AWgZSTmjUMkE2J58eTVn`.
- `logging-api` LogTree (10): `AWgZSTmHUMkE2J58eTVQ`, `AWgZSTmHUMkE2J58eTVR`, `AWgZSTmHUMkE2J58eTVS`,
  `AWgZSTmHUMkE2J58eTVT`, `AWgZSTmHUMkE2J58eTVU`, `AWgZSTmHUMkE2J58eTVV`, `AWgZSTmHUMkE2J58eTVW`,
  `AWgZSTmHUMkE2J58eTVX`, `AWgZSTmHUMkE2J58eTVY`, `AWgZSTmHUMkE2J58eTVZ`.
- `logging-common` AbstractLogger (10): `AW2vdi1BKOCjNAOnQHTl`, `AW2vdi1BKOCjNAOnQHTm`,
  `AW2vdi1BKOCjNAOnQHTn`, `AW2vdi1BKOCjNAOnQHTo`, `AW2vdi1BKOCjNAOnQHTp`, `AW2vdi1BKOCjNAOnQHTq`,
  `AW2vdi1BKOCjNAOnQHTr`, `AW2vdi1BKOCjNAOnQHTs`, `AW2vdi1BKOCjNAOnQHTt`, `AW2vdi1BKOCjNAOnQHTu`.

### java:S6201 (instanceof pattern) — module `xwiki-commons-crypto-cipher` fails the coverage gate
The 4 conversions are valid and compile, but removing the covered `CHECKCAST` instructions takes the
module's JaCoCo instruction ratio from 0.70 to 0.69 → `jacoco:check` fails under `-Pquality`. Re-try
only together with tests that raise the module's coverage; do not lower the pinned ratio.
- `AYyDaUiBjVDzm496Abeh` AbstractBcAsymmetricCipherFactory L142; `AYyDaUiKjVDzm496Abei`
  BcRc2CbcPaddedCipherFactory L47; `AYyDaUiOjVDzm496Abej` BcRc5b128CbcPaddedCipherFactory L53;
  `AYyDaUiTjVDzm496Abek` BcRc5b64CbcPaddedCipherFactory L52.

### java:S6204 (Stream.toList()) — unmodifiable list escapes a public API
- `AYyDaUxcjVDzm496Abgb` AbstractJobStatus L554 — `getLog(LogLevel)` is a public (deprecated) API
  returning the list to arbitrary callers, and no sibling branch already returns an unmodifiable one.

### java:S1604 (anonymous class → lambda) — `AccessController.doPrivileged` is ambiguous
The four `URIClassLoader` sites are `doPrivileged(new PrivilegedAction<T>(){…}, this.acc)`. A lambda
there does not compile: `doPrivileged(PrivilegedAction, AccessControlContext)` and
`doPrivileged(PrivilegedExceptionAction, AccessControlContext)` both match an implicitly-typed lambda
(verified with `javac`). Permanent drop for every `doPrivileged` site.
- `AWgZSU4oUMkE2J58eTbH` L278, `AWgZSU4oUMkE2J58eTbI` L298, `AWgZSU4oUMkE2J58eTbJ` L357,
  `AWgZSU4oUMkE2J58eTbK` L405 (legacy-classloader `URIClassLoader`).

### java:S7476 / java:S3706 in `xwiki-commons-extension-api` — NO LONGER DROPPED
The former blocker (the module was red on master on its own `revapi` check, JSpecify `@Nullable`
migration) is **fixed**: `mvn package revapi:check -Pquality -DskipTests -pl <module>` now passes.
The seven `DefaultCoreExtensionScanner` S7476 keys and the `AbstractExtensionHandlerTest` S3706 key
are fair game again — do not skip them.

## xwiki-rendering

### java:S1118 (add private constructor) — public utility classes in the `wikimodel` public API
All six are `public final class XxxUtil` with only static members, in the exported
`org.xwiki.rendering.wikimodel.*` packages: adding a private ctor removes the implicit PUBLIC one →
revapi `java.method.visibilityReduced`.
- `AV2j0WlOpvRVEt3bvRma` WikiPageUtil, `AV2j0WmYpvRVEt3bvRnJ` ImageUtil, `AV2j0WmypvRVEt3bvRn1`
  WikiScannerUtil, `AV2j0WndpvRVEt3bvRpt` HtmlEntityUtil, `AV2j0WoGpvRVEt3bvRqD` WikiEntityUtil,
  `AV2j0WsMpvRVEt3bvRsp` XWikiScannerUtil.

### java:S1066 (merge nested if) — comment between the two ifs
- `AZFHCq1E-4lPKEDZswFl` AbstractBoxMacro L261 — a MULTI-line `//` comment sits between the outer and
  inner `if` (a single-line one would be recoverable).

### java:S6126 (text block) — the whole `xwiki-rendering-wikimodel` parser-test pool is a drop
All 13 rendering S6126 issues are parser-test fixtures that a text block cannot reproduce byte for
byte. Do not re-triage them one by one; the rule is closed in this repo until those tests change.
- Meaningful trailing whitespace on content lines (wiki table/heading fixtures — `"| Multi \n"`,
  `"header \n"`, `"))) "`, `"…</span> \n"`): `AY98gWSIeYJJhSa54Ik8` (GWikiParserTest:133 — also an
  all-whitespace first line), `AY98gWRBeYJJhSa54Iiz` (JspWikiParserTest:184),
  `AY98gWRyeYJJhSa54Ijr`, `AY98gWRyeYJJhSa54Ijt`, `AY98gWRyeYJJhSa54IkU`, `AY98gWRyeYJJhSa54IkV`,
  `AY98gWRyeYJJhSa54Ikj`, `AY98gWRyeYJJhSa54Ikk`, `AZ018OgzZ_dZP7ZB0yJr` (XWiki20ParserTest
  165/177/673/676/866×2/890).
- `\r\n` line terminators — a text block always normalises to `\n`: `AY98gWRUeYJJhSa54IjL`
  (XHtmlParserTest:370).
- Leading-indent ladder with no line at the baseline, so the minimum-indent strip eats one space
  (only `\s` escapes could reproduce it, which is worse than the concatenation):
  `AY98gWSueYJJhSa54IlE`, `AY98gWSueYJJhSa54IlF`, `AY98gWSueYJJhSa54IlG` (ListBuilderTest 75/101/135).

### java:S6035 (alternation → character class) — public constant value change breaks Revapi
- `AXZl7LuPr56YxuFA79Xv` ReferenceHandler L32 (`PREFIX_DOWNLOAD`), `AXZl7LuPr56YxuFA79Xu` L36
  (`PREFIX_IMAGE`) — both `public static final String` compile-time constants;
  `java.field.constantValueChanged` fails `-Pquality` even though `"^(?:d|F)ownload:.*"` and
  `"^[dF]ownload:.*"` match identically.

### java:S4973 (compare with equals) — the `==` is a deliberate identity check
- `AYs84zH_g8eGkQKScd9o` DefaultMacroContentParser L124 — `getCurrentMacroBlock().getContent() == content`
  asks "is this the very same content object?"; the comment above it says so. `equals()` changes behaviour.

### java:S5976 (merge into a parameterized test) — test-design decision
- `AZX7pGrRm8lh03R4YlFO` XWikiReferenceParserTest L77 — merging 3 tests into one parameterized test is
  a design call, not a mechanical cleanup.

### java:S6878 (record pattern) — hurts readability in `equals()`
- `AZp8c-o9eyVFKzl0_wpi` MacroTransformation L132 — deconstructing `MacroItem` into its three
  components inside `equals()` collides with the record's own accessor names.

### java:S2094 (empty class) — deliberately empty, documented
- `AWgjJiye1_eUtAp8ETP3` FootnoteMacroParameters — Javadoc: "None at the moment, but the rendering
  engine requires specifying a class for parameters." (The sibling `AbstractXWikiSyntaxResourceRenderer`
  was NOT dropped: empty, `internal`, zero references → deleted.)

### java:S3824 (computeIfAbsent) — guarded block does more than the put
- `AX-cPTwWIdY56kbgh63l` AbstractXHTMLImageTypeRenderer L119 — the `if` also runs
  `computeIfPresent`/`putIfAbsent` on the `CLASS` attribute, so `computeIfAbsent(ID, …)` is not an
  equivalent rewrite.

### java:S2589 (condition always true) — deliberate defensive check in concurrent code
- `AWgjJi6N1_eUtAp8ETQL` DefaultLinkCheckerThread L176 — `shouldBeChecked && queueItem != null`: the
  null test is provably dead (an earlier line would already have NPE'd), but it guards a
  `linkQueue.poll()` result and reads as intentional. Removing it changes nothing except the intent.

### java:S3457 (use %n instead of \n) — would make one half of a message inconsistent
- `AXWJW7i3MJOM0yBUZBxP` DefaultTransformationManager L101 — the exception message is built from two
  `String.format` calls and only this one is flagged; converting it alone mixes `%n` and `\n`.

### java:S2925 (Thread.sleep in a test) — needs Awaitility + a timing rewrite
- `AWgjJi7D1_eUtAp8ETQO` (L189), `AWgjJi7D1_eUtAp8ETQP` (L256), `AWgjJi7D1_eUtAp8ETQQ` (L350)
  LinkCheckerTransformationTest.

### java:S4274 / java:S1182 / java:S1700 — behaviour or API changes, not cleanups
- `AWgjJjTX1_eUtAp8ETRK` WikiReference L98 (replace `assert` with a real check — changes runtime
  behaviour); `AV2j0WWopvRVEt3bvRjR` AbstractBlock L537 (`clone()` should call `super.clone()` — real
  design change); `AV2j0WeHpvRVEt3bvRkk` MetaData L78 (rename the `metadata` field — public-ish rename).

### java:S5785 — NOT dropped, suppressed in code
The 30 rendering S5785 issues sit in equals()/hashCode() contract tests. They are handled by
`@SuppressWarnings("java:S5785")` + an in-code rationale on the 6 concerned test methods
(`xwiki/xwiki-rendering#390`), NOT by accepting them in SonarCloud — so they are deliberately NOT
listed as dropped keys here. If they resurface, add the suppression rather than converting the
assertions; see `rules/test-code.md`.

## xwiki-platform

### java:S9016 (extract nested mock creation) — type-inferred `mock()` with no class literal
Not defects: the extraction is valid, but Sonar's `textRange` gives no type to name in the declaration,
so a scripted batch cannot do it. Recoverable by hand *when the stubbed method's return type is already
imported in the test* (that is how 6 of 7 such sites were converted); a drop only when naming the type
would mean adding an import from a foreign library.
- `AZ-5kehz8JWMl105u6eb` (L305), `AZ-5kehz8JWMl105u6ed` (L334) DefaultModelBridgeTest;
  `AZ-5kZa38JWMl105u6dV` InternalTemplateManagerTest L90.
- `AZ-5kc4M8JWMl105u6eJ` DefaultBaseClassRequiredRightAnalyzerTest L262 — naming the type of
  `this.hibernate.getConfiguration()` requires importing a Hibernate type into the test.

### java:S1118 (add/hide private constructor) — instantiated factories & abstract bases
- `AW5-S6zG1Yj5qvzeRnt9` DurationFactory — instantiated via `new DurationFactory()` in XWikiCriteriaServiceImpl.
- `AW5-S6yZ1Yj5qvzeRntv` PeriodFactory — instantiated via `new PeriodFactory()`.
- `AW5-S6zW1Yj5qvzeRnuB` RangeFactory — instantiated via `new RangeFactory()`.
- `AW5-S6yi1Yj5qvzeRnty` ScopeFactory — instantiated via `new ScopeFactory()`.
- `AW5-S5WZ1Yj5qvzeRm0r` AbstractNode — abstract base designed for extension (private ctor breaks subclasses).
- `AW5-S49G1Yj5qvzeRmv5` AbstractSecurityConfiguration — abstract base designed for extension.
- `AW5-S4nV1Yj5qvzeRmsT` CodeMacroLayout.Constants — nested holder in public-API package `org.xwiki.rendering.macro.code`; private ctor removes the implicit public one → revapi `visibilityReduced`.

### java:S2093 (use try-with-resources) — finally is a state RESTORE, not a close (whole batch dropped)
- `AZbU8DY3GHlYUfXgHgO9` DefaultRequestParameterConverter L136 — finally does rendering-context `pop()`.
- `AZO91cC0xltl5snfPKEv` WikiResourceImpl L100 — finally does `xcontext.setWikiReference(previous)`.
- `AX4vwXs1QZ7ZdoOVq7Ht` CachedLESSCompiler L151 — finally restores `xcontext.put(SKIN_CONTEXT_KEY, currentSkin)`.
- `AW-AiMNSpjn0nASQAOhU` CachedLESSCompiler L91 — finally does `semaphore.release()` (not a close).
- `AW5-S8vk1Yj5qvzeRoOt` DefaultTemplateHTMLDisplayer L101 — finally does `scriptContext.removeAttribute(...)`.
- `AW5-S6141Yj5qvzeRnva` HtmlPackager L366 — `zos` created mid-body; finally deletes a temp dir, no close.
- `AW5-S6IL1Yj5qvzeRnAe` VelocityTemplateEvaluator L102 — finally does rendering-context `pop()` + `progress.endStep`.
- `AW5-S4nz1Yj5qvzeRmsb` AbstractJSR223ScriptMacro L232 — finally restores writer/reader/binding.
- `AW5-S9J01Yj5qvzeRoVb` DefaultHTMLConverter L134 — finally does rendering-context `pop()`.
- `AW5-S9J01Yj5qvzeRoVe` DefaultHTMLConverter L207 — finally does rendering-context `pop()`.
- `AW5-S-oQ1Yj5qvzeRo5E` ZipExplorerPlugin L301 — finally does `filecontent.reset()` on a parameter InputStream.

### java:S1185 (remove super-only override) — reflective plugin dispatch / intent comment
- Plugin classes (`com.xpn.xwiki.plugin.*`, XWikiPluginManager.initPlugin reflective dispatch — must redeclare): `AW5-S6sU1Yj5qvzeRnm5`, `AW5-S6sU1Yj5qvzeRnm6` (FileUploadPlugin), `AW5-S6st1Yj5qvzeRnnJ` (PackagePlugin), `AW5-S-Ec1Yj5qvzeRouz`, `AW5-S-DR1Yj5qvzeRouJ`, `AW5-S-EL1Yj5qvzeRouf`, `AW5-S-EL1Yj5qvzeRoug`, `AW5-S-ED1Yj5qvzeRoue`, `AW5-S-DB1Yj5qvzeRouE`, `AW5-S-DJ1Yj5qvzeRouF`, `AW5-S-DJ1Yj5qvzeRouI`, `AW5-S-Dy1Yj5qvzeRouU` (skinx plugins), `AW5-S-pB1Yj5qvzeRo5O` (JodaTimePlugin).
- `AZwedJQk7L5XoT2tQWhQ` EmbeddedClient.queryAndStreamResponse — a TODO comment documents it as a deliberate workaround placeholder.

### java:S1192 (use existing constant) — forward-ref / coincidental value
- `AW5-S6tI1Yj5qvzeRnnj` Package L1091 (DefaultPluginName) — coincidental: literal is an XML root element name, not the plugin name.
- `AW5-S6L11Yj5qvzeRnCh`, `AW5-S6L11Yj5qvzeRnCi` XWikiGroupServiceImpl L66 — constants declared AFTER the line (forward reference).
- `AW5-S-g71Yj5qvzeRo3n` XWikiRepositoryModel L359 (SOLR_BOOLEAN) — forward reference (constant at L365).

### java:S1068 (unused field) — public API
- `AZBvgQ3Tcj_-G2g1uBr0` MockitoOldcore.notifyDocumentUpdatedEvent — exposed via a public setter used by external tests.

### java:S3626 (redundant jump) — empty-block / complex flow
- `AXnpAe8rDDFOvAKXAQOa`, `AXnpAe8rDDFOvAKXAQOb`, `AXnpAe8rDDFOvAKXAQOc` XWikiAction — returns inside complex nested try/catch/finally.
- `AW5-S6tI1Yj5qvzeRnoL` Package L530 — continue is branch's only statement → empty block.
- `AW-kEsoeDYXZF0Pw6gz4` ListClass L388 — continue is else-if branch's only statement → empty block + changes flow.

### java:S1640 (HashMap→EnumMap) — null key / order dependency
- `AW5-S4591Yj5qvzeRmu7` (L120), `AXFqzITY-w3IdlBFv6Fa` (L125) Right.java (security-authorization-api) ENABLED_RIGHTS/UNMODIFIABLE_ENABLED_RIGHTS — populated with a `null` EntityType wildcard key (`enableFor(null,…)`); EnumMap forbids null keys → NPE at class init.

### java:S1643 (String += → StringBuilder) — prepend / order-sensitive, not tail-append
- `AW5-S6Qs1Yj5qvzeRnJL` PasswordClass L342 — `s = "0" + s` zero-pad prepend inside `while (s.length()<2)`; loop reads intermediate length.
- `AW5-S6z-1Yj5qvzeRnuI` (L102), `AW5-S6z-1Yj5qvzeRnuJ` (L106) TOCGenerator — `number = seg + number` hierarchical prepend; append would reverse output.

### java:S3878 (varargs array) — recursion
- `AX8g9GlX4xa8fAuVggkA` DocumentStringUserReferenceSerializer L59 — empty `new Object[]{}` routes to the varargs overload; removing it recurses into the 1-arg method.

### java:S6201 (instanceof pattern) — flow-scope / same-file open PR
- `AYyD4sQEj2dtqk6dnyJA` SecurityReference L88 — cast is in the ternary `:` branch where the pattern var is not definitely assigned (`x != null && !(x instanceof Y)` prefix).
- `AYyD4w7vj2dtqk6dnyN2` ExtensionVersionFileRESTResource L218 — same file as an open agent PR (#5490); off-limits to avoid conflicts.

### java:S6204 (Stream.toList()) — unmodifiable list escapes to public API
- `AYyD4rNNj2dtqk6dnyHk` OsvResponseAnalyzer L85 — list stored in ExtensionSecurityAnalysisResult, re-exposed by a public getter.
- `AYyD4sSKj2dtqk6dnyJF` DefaultMacroRequiredRightReporter L59 — list passed to RequiredRightAnalysisResult, exposed by a public getter.
- `AYyD4v44j2dtqk6dnyNK` TemporaryAttachmentsScriptService L149 — ScriptService method, script-exposed escape risk.
- `AZ4iYuZ97PLUpMHUdKe6` CreateActionRequestHandler L496 — assigned to a field returned by a public getter.

### java:S6126 (text block) — resulting line >120 or trailing-whitespace not reproducible
- Line >120 (long `beginMetaData [[...]]` event lines): `AY974jGxKZk1650Dhx29`, `AY974jGxKZk1650Dhx2_`, `AY974jGxKZk1650Dhx3B` (IncludeMacroTest); `AY974pCqKZk1650DhyQA`, `AY974pCqKZk1650DhyQC` (DisplayMacroTest); `AY974jXQKZk1650Dhx30`, `AY974jXQKZk1650Dhx31`, `AY974jXQKZk1650Dhx32`, `AY974jXQKZk1650Dhx33` (DefaultWikiMacroTest).
- Trailing whitespace on data rows (chart table/CSV strings, text blocks strip it): `AY974oVLKZk1650DhyOe`, `AY974oVLKZk1650DhyOf` (TableCategoryDatasetBuilderTest); `AY974oU3KZk1650DhyOc`, `AY974oU3KZk1650DhyOd` (TablePieDatasetBuilderTest); `AY974oVaKZk1650DhyOg`, `AY974oVaKZk1650DhyOh` (TableTimeTableXYBuilderTest).
- Line >120 once un-concatenated (merged no-`\n` fragments / markup / DOCTYPE lines): `AY974mAwKZk1650DhyFT`, `AY974mAwKZk1650DhyFV` (PdfExportImplTest DOCTYPE); `AY974mCgKZk1650DhyGd`, `AY974mCgKZk1650DhyGe`, `AY974mCgKZk1650DhyGf` (InvitationInternalDocumentParameterEscapingFixerTest `{{info}}` markup); `AY974pdXKZk1650DhyQq`, `AY974pdXKZk1650DhyQr` (HierarchyMacrosPageTest `resolveObject*` merge); `AY974pcbKZk1650DhyQi` (NotificationRssDefaultPageTest); `AY974p0EKZk1650DhyRh` (RssMentionPageTest); `AY974rMZKZk1650DhyVl` (XMLScriptServiceTest DOCTYPE); `AY974qb3KZk1650DhyTg` (StackTraceLogParserTest:45).
- Trailing whitespace on content line (text blocks strip it): `AY974qb3KZk1650DhyTh`, `AY974qb3KZk1650DhyTj` (StackTraceLogParserTest — `"date - \tat "` / space before `\n`); `AY974peMKZk1650DhyQ1`, `AY974peMKZk1650DhyQ2` (NotificationMailDefaultHtmlTest — all-whitespace `"      \n"` rows, also >120).
- Valid conversion skipped for build-ROI (not a fix defect): `AY974l35KZk1650DhyAE` (HibernateStoreTest:98) — byte-safe, but shipping it means building oldcore's full test suite for one site.

### java:S7476 / java:S3706 — valid fixes skipped for build ROI (`xwiki-platform-search-solr-api`)
Not defects: the conversions are clean, but Solr submodules are slow to build and this was the only
reason to add the module to the reactor. Pick them up in a batch that already builds solr-api.
- `AZ5GtE2Xwxx8uFGPms3i`, `AZ5GtE2Xwxx8uFGPms3j` (ObjectSolrMetadataExtractorTest, S7476);
  `AZcwcqCH8IL3Wg1vzZyz`, `AZcwcqCH8IL3Wg1vzZy0`, `AZcwcqCH8IL3Wg1vzZy1`, `AZcwcqCH8IL3Wg1vzZy2`
  (AbstractSolrCoreInitializer, S7476); `AZ3geiByzLZemL-okY2L` (SolrIndexEventListener, S3706).
