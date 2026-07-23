# Dropped issues — analyzed, decided NOT to fix (skip these on future runs)

Per Vincent's override: record every issue analyzed but not fixed, with the reason, so future runs
SKIP them instead of re-triaging. Keyed by SonarCloud issue key (stable across scans until the code
changes). When a listed file is later edited, its keys may go stale — that is harmless (the key just
won't match an open issue). Group by rule; keep the reason to one line. This is a skip-index, not run
history — merge/trim in place, don't append dated anecdotes.

## xwiki-platform

### java:S1118 (add/hide private constructor) — instantiated factories & abstract bases
- `AW5-S6zG1Yj5qvzeRnt9` DurationFactory — instantiated via `new DurationFactory()` in XWikiCriteriaServiceImpl.
- `AW5-S6yZ1Yj5qvzeRntv` PeriodFactory — instantiated via `new PeriodFactory()`.
- `AW5-S6zW1Yj5qvzeRnuB` RangeFactory — instantiated via `new RangeFactory()`.
- `AW5-S6yi1Yj5qvzeRnty` ScopeFactory — instantiated via `new ScopeFactory()`.
- `AW5-S5WZ1Yj5qvzeRm0r` AbstractNode — abstract base designed for extension (private ctor breaks subclasses).
- `AW5-S49G1Yj5qvzeRmv5` AbstractSecurityConfiguration — abstract base designed for extension.

### java:S1144 (unused private method) — Hibernate-mapped
- `AX-N4fWt2gSgI3Rl_5uE` XWikiDocument.getOriginalMetadataAuthorReference — mapped `<property name="originalMetadataAuthorReference">` in every hbm.xml.
- `AX-N4fWt2gSgI3Rl_5uF` XWikiDocument.setOriginalMetadataAuthorReference — same Hibernate mapping.

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
