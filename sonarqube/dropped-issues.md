# Dropped issues — analyzed, decided NOT to fix (skip these on future runs)

Per Vincent's override: record every issue analyzed but not fixed, with the reason, so future runs
SKIP them instead of re-triaging. Keyed by SonarCloud issue key (stable across scans until the code
changes). When a listed file is later edited, its keys may go stale — that is harmless (the key just
won't match an open issue). Group by rule; keep the reason to one line. This is a skip-index, not run
history — merge/trim in place, don't append dated anecdotes.

Sections are per REPO — issue keys are unique per SonarCloud project, so a key from
`org.xwiki.commons:xwiki-commons` never collides with a platform one. Repos with no drops yet are
listed as such rather than omitted, so a future run knows the absence is real and not an oversight.

## xwiki-commons

(A 37-site `java:S7158` sweep, a 34-site `java:S6201` sweep of `xwiki-commons-xml` +
`xwiki-commons-filter-xml`, and a 28-site `java:S7476`+`java:S3706` sweep were each analyzed in full
and every site was fixed.)

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

### java:S7476 / java:S3706 — module `xwiki-commons-extension-api` is red on master
The fixes themselves are valid, but the module fails its own `revapi` check on master (JSpecify
`@Nullable` migration, see `learnings.md` → Building / verifying), so it cannot be shipped green.
Re-try once master's revapi is fixed.
- `AZ3fPeNCs6Hfl3oG5jGp` (AbstractExtensionHandlerTest, S3706); `AZcvxNEwTKQ4AEDhZogt`,
  `AZcvxNEwTKQ4AEDhZogu`, `AZcvxNEwTKQ4AEDhZogv`, `AZcvxNEwTKQ4AEDhZogw`, `AZcvxNEwTKQ4AEDhZogx`,
  `AZcvxNEwTKQ4AEDhZogy`, `AZcvxNEwTKQ4AEDhZogz` (DefaultCoreExtensionScanner, S7476).

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

### java:S5785 (use dedicated JUnit assertion) — equals()/hashCode() contract tests, REJECTED in review
All 30 open rendering S5785 issues sit in dedicated equality-contract test methods. A PR converting
them (`xwiki/xwiki-rendering#389`) was CLOSED on review: `assertTrue(a.equals(b))` makes it explicit
which object's `equals` is exercised, and converting makes that depend on JUnit internals + argument
order (and invites a later `java:S3415` swap that would break it). See `rules/test-code.md`. Do not
retry these.
- LinkStateTest (4): `AZjxN4MMydzFhytemtU1`, `AZjxN4MMydzFhytemtU2`, `AZjxN4MMydzFhytemtU3`, `AZjxN4MMydzFhytemtU4`.
- MacroContentSourceReferenceTest (5): `AYYHPo7DWD-yte4e8LeC`, `AYYHPo7DWD-yte4e8LeD`, `AYYHPo7DWD-yte4e8LeE`, `AYYHPo7DWD-yte4e8LeF`, `AYYHPo7DWD-yte4e8LeG`.
- MacroIdTest (14): `AXj0YJhnfft_IP7n5aq_`, `AXj0YJhnfft_IP7n5arA`, `AXj0YJhnfft_IP7n5arB`, `AXj0YJhnfft_IP7n5arC`, `AXj0YJhnfft_IP7n5arD`, `AXj0YJhnfft_IP7n5arE`, `AXj0YJhnfft_IP7n5arF`, `AXj0YJhnfft_IP7n5arG`, `AXj0YJhnfft_IP7n5arH`, `AXj0YJhnfft_IP7n5arI`, `AXj0YJhnfft_IP7n5arJ`, `AXj0YJhnfft_IP7n5arK`, `AXj0YJhnfft_IP7n5arL`, `AXj0YJhnfft_IP7n5arM`.
- ResourceReferenceTest (3): `AXInk9JjfCbQNqY0fR2-`, `AXInk9JjfCbQNqY0fR28`, `AXInk9JjfCbQNqY0fR29`.
- SyntaxTest (2): `AXInk9L3fCbQNqY0fR3g`, `AXInk9L3fCbQNqY0fR3h`.
- SyntaxTypeTest (2): `AXYPy2uwq9YfDBN_rAfb`, `AXYPy2uwq9YfDBN_rAfc`.

## xwiki-platform

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
