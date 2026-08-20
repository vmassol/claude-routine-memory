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

### java:S2447 (null returned from a `Boolean` method) — the null IS the API contract
Same shape as platform's: `ListTool`, `SerializableXStreamChecker` and `Request` return `null` to mean
"undecided", which is not interchangeable with `false`.
- `AW6oRQpxoucddIov5uRZ`, `AW6oRQpxoucddIov5uRa`, `AW2vdi3HKOCjNAOnQHTz`, `AW2vdi3HKOCjNAOnQHT0`,
  `AWgZST5cUMkE2J58eTXi`, `AWgZST5cUMkE2J58eTXj`.

### java:S6876 (use `reversed()`) — the loop body mutates through the iterator
- `AZp8a1DYEzLQkZWCl-Gv` DefaultVersion#trimPadding L458 — the backward `ListIterator` calls
  `it.remove()`, which a for-each over `reversed()` cannot do. **This is the rule's only drop
  condition**: grep the body for `it.remove(`/`it.set(` and convert everything else.

### java:S1192 (duplicated literal) — a literal inside a vocabulary list
- `AYG0Ud0QouLKnFEkGft-` SVGDefinitions L87 — `"style"` appears three times because it is a member of
  three different allow-lists of SVG attribute/tag names; a shared constant would obscure the lists.


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

### java:S1192 (duplicated literal) — coincidental duplication inside vocabulary lists
- `AYG0Ud0QouLKnFEkGft-` SVGDefinitions L87 (`"style"`), `AYG0Ud0QouLKnFEkGft9` L99 (`"title"`) — the
  literals are entries of three different SVG/HTML name lists (attributes, elements, integration
  points); a shared constant hides the lists and the repetition is coincidental, not a shared concept.

### java:S1130 (remove unthrowable `throws`) — `src/main` methods with callers
The 24 **test** sites are all clean fixes; the 4 `src/main` ones are permanent drops — narrowing a
`throws` on a published method breaks callers that catch it.
- `AWgZSVQPUMkE2J58eTdz` MockitoComponentMocker L228; `AV4uHS0Q5jV1AdqTqB1L` DefaultJobManager L114;
  `AV2juG7VVSxcxmoV58Lz` AbstractBeanOutputFilterStream L68; `AV2juGiwVSxcxmoV58Cq`
  AbstractExtensionMojo L168.

### java:S125 (commented-out code) — a `TODO`/`FIXME` or a documented intent anchors the block
Roughly 80% of a `S125` pool is NOT dead code. Two shapes, both permanent drops: a block whose
neighbouring comment names the JIRA issue that will restore it, and a block that *documents* what the
code below it covers.
- Anchored by a `TODO`/`FIXME` (mostly XCOMMONS-3028 / XCOMMONS-1292): `AY-F49LFUnN6kAHHxlXP`
  AetherExtensionRepository L729; `AY-F49M0UnN6kAHHxlXe` AetherDefaultRepositoryManagerTest L236;
  `AY-F49NKUnN6kAHHxlXz` InstallJobTest L85; `AY-F47phUnN6kAHHxlTQ` DefaultHTMLCleanerTest L593;
  `AY-F47rQUnN6kAHHxlUA` StAXUtilsTest L66; `AYnfzRhr5_vnQ7oYSQpG`, `AYnfzRhr5_vnQ7oYSQpH`,
  `AYnfzRhr5_vnQ7oYSQpI` SafeXStream L42/96/105; `AV4uHSm75jV1AdqTqB0N` DefaultHTMLCleaner L144;
  `AV2juGhFVSxcxmoV58B3` XARMojo L568 (the comment explains WHY `split` is not used);
  `AY-F48s3UnN6kAHHxlWY`, `AY-F48s3UnN6kAHHxlWZ` JSONToolTest L240/257.
- Documentation of the `@Inject` declaration each assertion block covers, triplicated across the
  Jakarta/Javax/legacy copies of the same test: `AZRQnUDg6WWG1k8VL2F9`, `AZRQnUDg6WWG1k8VL2GA`,
  `AZRQnUDg6WWG1k8VL2GD`, `AZRQnUDg6WWG1k8VL2GG`, `AZRQnUDg6WWG1k8VL2GJ`, `AZRQnUDg6WWG1k8VL2GM`
  (JakartaComponentDescriptorFactoryTest); `AY-F47leUnN6kAHHxlSx`, `AY-F47leUnN6kAHHxlS0`,
  `AY-F47leUnN6kAHHxlS3`, `AY-F47leUnN6kAHHxlS6`, `AY-F47leUnN6kAHHxlS9`, `AY-F47leUnN6kAHHxlTA`
  (JavaxComponentDescriptorFactoryTest); `AY-F48hrUnN6kAHHxlVP`, `AY-F48hrUnN6kAHHxlVS`,
  `AY-F48hrUnN6kAHHxlVV`, `AY-F48hrUnN6kAHHxlVY`, `AY-F48hrUnN6kAHHxlVb` (legacy
  ComponentDescriptorFactoryTest).

### java:S2093 (try-with-resources) — `closeQuietly` in the `finally`, so converting changes behaviour
- `AWl4RiCkzMcy0S6oSn2B` InfinispanCacheFactory L105 — the `finally` is a genuine
  `IOUtils.closeQuietly(configurationStream)`, i.e. the one shape this rule normally converts, BUT
  `closeQuietly` deliberately swallows a close failure while try-with-resources would surface it
  through the enclosing `catch (IOException)` as an `InitializationException`. A behaviour change on
  an error path in a `src/main` cache component is over the mechanical bar.

### java:S5778 (one throwing call per `assertThrows` lambda) — the hoisted call IS the thrower
- `AZpzqATq_W9UTNvTGQYE` BlobPathTest L208 — the `IllegalArgumentException` comes from
  `BlobPath.relative("..", "bad/name")`, not from the `resolve()` it is passed to, so hoisting it out
  of the lambda moves the exception outside `assertThrows` and breaks the test. (The three sibling
  sites were clean.)

### java:S3358 (nested ternary) — FIXED, no longer dropped
The three `DefaultVersionRange` keys shipped as their own PR (the extraction is provably equivalent —
the two branches of each outer ternary are exact negations — but it stays a readability judgement, so
it was split off from the mechanical batch rather than dropped).

### java:S2629 (invoke conditionally) — WHOLE RULE: never remove the eager String
**Corrected — there is no clean shape.** "The explicit `x.toString()` is redundant because SLF4J calls
it itself" is WRONG in XWiki and cost a withdrawn PR (`xwiki/xwiki-commons#1888`): a job captures the
`LogEvent` with its **raw `Object[]`** and `SafeMessageConverter` XStream-serializes it into the job
log, where `XStreamUtils.isSerializable()` **defaults to true** — so the object is written as a full
graph, and `SafeArrayConverter.readBareItem()` turns any read failure into `null`. The eager String is
a deliberate snapshot. The rule is owned by the OKF's `conventions/logging.md` (**not**
`okf/sonarqube/`, which is why a Sonar sweep never reaches it) and the only resolution is
`@SuppressWarnings("java:S2629")` + the inline reason — see `AbstractInstallPlanJob` and
`DefaultJobProgress.onEndStepProgress()`. The 4 `AbstractInstallPlanJob` keys
(`AXNSaTSQLv0ks60bJ0DB`-`DE`) are suppressed in code, not dropped. The remaining sites all need an
`isXxxEnabled()` guard, which is a judgement call:
- `AYjgzRgXPYtryzrppJ6t`, `AYjgzRgXPYtryzrppJ6u`, `AYjgzRgXPYtryzrppJ6v` XMLUtils L97/111/124 — the
  argument is `ExceptionUtils.getRootCauseMessage(exception)`; one of the three is a `warn`, which is
  enabled anyway.
- `AXKD8jA6g7M8To5DTnWM`, `AXIpLV2-0IfHJsxwimTz`, `AXEX-V0v6U9Dwo1n47dT`, `AXEX-V0v6U9Dwo1n47dU`,
  `AXEX-V0v6U9Dwo1n47dV`, `AXEX-V0v6U9Dwo1n47dW` FailingTestDebuggingTestExecutionListener L56-64 —
  the arguments are `RuntimeUtils.run("top …"/"docker ps"…)` whose whole purpose is to be printed,
  and the block is already guarded by `isInCI()`.

### java:S899 / java:S4042 (ignored `delete()` result / use `Files#delete`) — delete in a `finally`
The two rules sit on the same lines, so one edit would clear both — but every commons site is a temp
file deleted in a `finally`, and `Files.delete` throws, masking the original exception and creating a
fresh `S1163`. The `offer()` variant is an unbounded queue (always `true`), so "doing something" with
the result is a design change.
- `AXt3f5KPleV-YJ_ZFWoB`/`AXt3f5KPleV-YJ_ZFWoC` ExtendedJarURLConnection L106;
  `AWgZSTiwUMkE2J58eTU0`/`AWgZSTiwUMkE2J58eTUz` FileAssert L117; `AWgZSTjGUMkE2J58eTU6`/
  `AWgZSTjGUMkE2J58eTU5` L135 and `AWgZSTjGUMkE2J58eTU8`/`AWgZSTjGUMkE2J58eTU7` L138
  ZIPFileAssertComparator; `AWgZSVLwUMkE2J58eTc7`/`AWgZSVLwUMkE2J58eTc6` XARMojo L521.
- `offer()` (design change): `AWgZSTH7UMkE2J58eTTB`, `AWgZSTH7UMkE2J58eTTC`
  DefaultExtensionJobHistory L124/131; `AWgZST6wUMkE2J58eTXz`, `AWgZST6wUMkE2J58eTX0`
  DefaultJobManager L208/216.

### java:S1163 (throw in `finally`) — the rethrow is the intent
- `AW6i52SPr36yrbKLYWHs` ExecutionContextRunnable L76 — the `finally` deliberately rethrows a context
  cleanup failure as a `RuntimeException`; "fixing" it means deciding to swallow it.

### java:S1144 / java:S1118 / java:S6204 / java:S1185 / java:S1171 commons singletons
- `AY-F47j8UnN6kAHHxlSf`, `AY-F47j8UnN6kAHHxlSg` ReflectionUtilsTest L113/142 — the "unused" private
  methods ARE the subject of the reflection test that looks them up by name.
- `AYRYmI-EY9WvGmTad5t0` AgentUtil L50 — the class is copied verbatim from CSS4J and already carries
  `@SuppressWarnings("checkstyle:HideUtilityClassConstructor")` plus a comment saying divergence from
  upstream is deliberately minimised.
- `AYyDaUxcjVDzm496Abgb` AbstractJobStatus L554 — the list escapes through a public (`@Deprecated`)
  getter, so an immutable `toList()` result would change the contract.
- `AZnoBG5_sgBBO9CiAWpB` DefaultCoreExtension L112, `AV4uHSsf5jV1AdqTqB0u`
  AbstractGenericComponentManager L108 — super-only overrides on public API; needs a per-site check of
  the parent's visibility before it is mechanical.
- `AV4uHSLB5jV1AdqTqBv_` DefaultExtensionSerializer L219 — the instance initializer is ~30 lines and
  moving it into a constructor is past the mechanical bar.

### java:S3012 (manual array copy) — the loop copies a SUFFIX, not the array
- `AZkVtQQYz-Gjyq6TVk6Y` LogUtils L196 — `for (int i = arguments.size(); i < defaults.length; ++i)`
  appends from an offset, so every suggested replacement (`Arrays.copyOfRange` + `Collections.addAll`,
  `subList`) reads worse than the loop. The rule is only clean on a whole-array copy.

### java:S1185 (remove super-only override) — comment / public-API removal
- `AV4uHSsf5jV1AdqTqB0u` AbstractGenericComponentManager L108 — a "Note: Ideally …" comment on the
  override explains why it exists.
- `AZnoBG5_sgBBO9CiAWpB` DefaultCoreExtension L112 (`setId`) — would take a public method off a public
  API class for one issue; not worth the Revapi risk.

### java:S1171 (instance initializer) — same file as an open agent PR
- `AV4uHSLB5jV1AdqTqBv_` DefaultExtensionSerializer L219 — file claimed by open PR #1875.

### java:S1144 (unused private method) — asserted BY NAME in the test's expected output
- `AY-F47j8UnN6kAHHxlSf` (L113 `privateParentMethod`), `AY-F47j8UnN6kAHHxlSg` (L142 `privateMethod`)
  ReflectionUtilsTest — both appear in the expected `getAllMethods` strings at L310-311, so they are
  used, just never called. Any "unused private member" inside a *reflection* test is suspect.

### java:S5786 (JUnit 5 visibility) — cross-package use / an `@Override` of a public method
- `AZ827Jbt5HXsydqHEg0y` DefaultHTMLCleanerTest — `HTMLUtilsTest` (another package) reads
  `DefaultHTMLCleanerTest.HEADER`, so the class cannot become package-private.
- `AZ827KKd5HXsydqHEg00` InstallJobTest — its `setUp()` is `@BeforeEach @Override` of a public parent
  method; reducing an override's visibility does not compile, so the file cannot be fully cleared.

### java:S3457 (`\n` → `%n`) — the produced string is asserted, so the separator is behaviour
- `AWgZSTp2UMkE2J58eTVu` UnifiedDiffBlock:140 and `AY-F47xFUnN6kAHHxlUC` ExtendedDiffDisplayerTest:141 —
  `UnifiedDiffBlock#toString()` emits a unified-diff header and the test rebuilds the *same* literal to
  assert against it. `%n` would make the diff text platform-dependent.

### java:S3457 (`toString()` shape) — deleting the call changes what the JOB LOG stores
`AXDOBHbO3-iMCxIZIEYv` RepositoryUtils:420 is **fixed, but not the way it first looked** — this is the
same trap as `java:S2629` above. My first fix simply deleted the `.toString()`; that reverted `64ba541`
(XWIKI-24665), **the direct parent of the branch**, which had added it on purpose. Corrected in-flight
by `816355e` to **`String.valueOf(repository.getDescriptor())`**, which clears the rule (it is not a
`toString()` call) while keeping the eager snapshot — the form `DefaultJobProgress` already uses. No
`@SuppressWarnings("java:S2629")` is needed when the call sits in a `catch` block: S2629 skips those.

### java:S3824 (computeIfAbsent) — guarded block does more than the put / concurrency
- `AWgZSU85UMkE2J58eTcM` DefaultBeanDescriptor:303 — the guarded block has an `else if` branch, so
  `computeIfPresent` is not an equivalent rewrite.
- `AXJ6kDt8oUIcIBrOdW4q` JobGroupPathLockTree:49 — the map is a `ConcurrentHashMap` and the mapping
  function would call `GroupedJobInitializerManager` while the bin lock is held. Concurrency change.
- `AY7OTFZKsA0XqdeA5Uak` AbstractInstallPlanJob:198 — same file as the withdrawn `S2629` fix; the
  guarded block is not a lone `put` and the class's log/plan bookkeeping is deliberately eager.

### java:S3457 (no need to call toString()) — log calls in `AbstractInstallPlanJob`
The whole remaining commons pool sits in one file, and it is the file whose eager `toString()` was
*added on purpose* (XWIKI-24665): the job captures the `LogEvent` with its raw `Object[]` and
XStream-serializes it into the job log, so the String is load-bearing. Two PRs have already been
withdrawn over this. Permanent drop: `AXNSaTSQLv0ks60bJ0DF` L400, `AXNSaTSQLv0ks60bJ0DG` L606,
`AXNSaTSQLv0ks60bJ0DH` L659, `AXNSaTSQLv0ks60bJ0DI` L697, `AXNSaTSQLv0ks60bJ0DJ` L700.

## xwiki-rendering

### java:S1118 (add private constructor) — public utility classes in the `wikimodel` public API
All six are `public final class XxxUtil` with only static members, in the exported
`org.xwiki.rendering.wikimodel.*` packages: adding a private ctor removes the implicit PUBLIC one →
revapi `java.method.visibilityReduced`.
- `AV2j0WlOpvRVEt3bvRma` WikiPageUtil, `AV2j0WmYpvRVEt3bvRnJ` ImageUtil, `AV2j0WmypvRVEt3bvRn1`
  WikiScannerUtil, `AV2j0WndpvRVEt3bvRpt` HtmlEntityUtil, `AV2j0WoGpvRVEt3bvRqD` WikiEntityUtil,
  `AV2j0WsMpvRVEt3bvRsp` XWikiScannerUtil.

### java:S2198 (comparison always true/false) — a dead range that mirrors the XML spec
`WikiPageUtil.isValidXmlNameStartChar(char ch, …)` ends its range table with
`(ch >= 0x10000 && ch <= 0xEFFFF)`, unreachable for a `char`. Deleting it is behaviour-preserving but
drops the supplementary-plane row of the XML 1.0 `NameStartChar` production the table transcribes,
and the real fix (take an `int` code point) is an API change. Both keys sit on the same line.
- `AZ_01MtBbzuOmnNi3w77`, `AZ_01MtBbzuOmnNi3w78` WikiPageUtil L310.

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

### java:S2093 (try-with-resources) — the `finally` restores state, it does not close
- `AZNziSnUEcK0YeraNzE4` AbstractInternalRenderingTest L160 — the `finally` does
  `MutableRenderingContext.pop()` + `executionContextManager.popContext()`. (The sibling
  `TestDataParser` L80 site WAS a real `reader.close()` and is fixed.)

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

### java:S3415 (swap assertion operands) — false positive / explicitly warned against in code
- `AYYHPo6DWD-yte4e8LeB` MacroContentSourceReferenceTest L43 — the arguments are ALREADY
  `(expected, actual)`; and that same file carries a `@SuppressWarnings("java:S5785")` whose comment
  explicitly warns that a later S3415 "swap these arguments" change would stop testing the `equals()`
  contract. `AYYHPo2jWD-yte4e8LeA` MacroContentWikiSourceTest L47 — both operands are constructed
  objects, so the swap is cosmetic only.

### java:S1130 (remove unthrowable `throws`) — `src/main` method other modules call
- `AZNziSnUEcK0YeraNzE5` AbstractInternalRenderingTest L288 — dropping `throws Exception` from a
  method in `xwiki-rendering-test`'s `src/main` breaks callers that catch it. The rule is only safe on
  a test method nothing calls.

### java:S3252 (static access via a derived type) — the derived type is XWiki's own `StringUtils`
- `AXYPy2qQq9YfDBN_rAfZ` SyntaxType:369, `AWy9lcQ0KTwBvn8qD2vO` IconTransformation:106 — both call an
  Apache Commons method through `org.xwiki.text.StringUtils`. See the platform section for the full
  reason; the rest of rendering's 34 were fixed.

### Rendering is NOT closed — a denylisted rule re-opened 32 sites
The "49-rule facet yields nothing" conclusion below held only for the *allowlist*. `S3252`, which the
OKF denylist described as backward-compat-bearing, turned out to be a pure qualifier change and paid
32 sites in two modules. Re-read a denylist reason against the rule's own definition before believing
a repo is closed.

A 49-rule facet cross-checked against this index produced exactly three keys not already denylisted,
rejected or dropped, and all three are non-starters: `S1845` (rename a field — denylisted shape),
`S5411` ×2 (boxed → primitive `boolean` — denylisted), `S5843` (reduce regex complexity 23 → 20, a
refactor). Budget a rendering allowlist PR at 0 until something regenerates.

## xwiki-platform

### java:S125 (commented-out code) — anchored by a TODO/FIXME, or a Sonar false positive
The rule's clean subset is an *unanchored* leftover (a disabled `verify(…)`/`when(…)` in a test, a
dead alternative implementation); 17 of the 28 fresh platform sites are drops and every reason is
visible in the flagged block:
- **A `TODO`/`FIXME` right above or below the block owns it** — `AW5-S-aW1Yj5qvzeRo2K`
  DefaultFlavorManager, `AW5-S8131Yj5qvzeRoQv` SecureGroovyCompilationCustomizer,
  `AW5-S5k51Yj5qvzeRm3M` WikiUIExtensionComponentBuilder, `AW5-S88e1Yj5qvzeRoSo`
  DomainWikiReferenceExtractor, `AYbMyekPEULH0Mn4tsKB` XWikiBlogNewsSource,
  `AYbrRz0gM5HAxHRCjFBd` XWikiBlogNewsCategoryConverter ("once it's fixed, uncomment the following
  line"), `AZOwlzyOY-cFm5FjhEI0` DefaultResourceReferenceEntityReferenceResolverTest (FIXME naming
  XWIKI-22699), `AZ85PRNJ7ctcaw4Skp-F` LegacyEventMigrationJob (a TODO below it says "see previous
  commented code").
- **The comment explains why the code is NOT run** — `AW5-S47s1Yj5qvzeRmvn` RightSet#size (why
  `Long.bitCount()` is avoided), `AW5-S-gg1Yj5qvzeRo3l` + `AZ7imjBN8lbwoM_tj_bk` RepositoryManager
  ("don't import website since…"), `AW5-S9Aa1Yj5qvzeRoS6` EntityResourceReferenceHandler ("in the
  future, modify this to return…"), `AY974nPBKZk1650DhyKG` DefaultAuthorizationManagerIntegrationTest
  L377 (the alternative mock the test deliberately does not use), `AY974p67KZk1650DhyR8`
  ProvidingTransactionRunnableTest L49 ("this should fail at compile time: …" documents a negative
  test case).
- **Prose Sonar mis-reads as code (false positive)** — `AW5-S5Fz1Yj5qvzeRmyK`
  R40001XWIKI7540DataMigration, `AXnpAdFGDDFOvAKXAQCb` ModelFactory.
- **Removing it would leave an empty block** (a fresh `S108`) — `AZGaGx8HPhpr_P5DN478`
  ExtensionIndexJob L202, the only statement of an `if (getRequest().isLocalExtensionsEnabled())`.

### java:S116 (field naming) — the one non-`private` site
- `AW5-S6ET1Yj5qvzeRm_Q` AbstractSkin `protected Skin VOID` — a `protected` field on a published
  class is a cross-module rename. The other 16 platform sites are all `private` and were fixed.

### java:S2447 (null returned from a `Boolean` method) — the null IS the API contract
Every site is a script service or bridge whose `Boolean` return means "not applicable / no answer"
(`null`) as opposed to `false`; returning `false` instead is a behaviour change on a public API and a
per-site product decision, not a cleanup. Same shape in commons (see that section).
- `AXnpAf7oDDFOvAKXAQhj`, `AW5_87iCV8VR4Ualy-ov`, `AW5_87iCV8VR4Ualy-ow`, `AW5-S8e31Yj5qvzeRoME`,
  `AW5-S5BQ1Yj5qvzeRmwh`, `AW5-S5BQ1Yj5qvzeRmwi`, `AW5-S9Tc1Yj5qvzeRoXO`, `AW5-S9Tc1Yj5qvzeRoXP`,
  `AW5-S9RU1Yj5qvzeRoW5`, `AW5-S9OD1Yj5qvzeRoWW`, `AW5-S9OD1Yj5qvzeRoWZ`, `AW5-S9OD1Yj5qvzeRoWa`.

### java:S3655 (`Optional#get()` without `isPresent()`) — presence proved by a guard Sonar cannot follow
- `AZCZDlD8LWAsJr2aywhx` EntityChannelScriptAuthorBot L77 — the value is guarded by a preceding
  `entityChannel.map(this::needsProtection).orElse(false)` stored in a boolean; making Sonar see it
  means restructuring the method. The two sites where the fix was a one-liner (`map(…).orElse(false)`
  and `Collectors.joining`) were fixed.

### java:S6912 (use `addBatch`/`executeBatch`) — a batching decision, not a cleanup
- `AZf1b0yb-3Rl8fL0EFUS`, `AZf1b0yb-3Rl8fL0EFUT` R35101XWIKI7645DataMigration L137/L140 — batching a
  migration's statements changes its failure granularity and error reporting.


### java:S5778 (one throwing call per `assertThrows` lambda) — same FILE as an open agent PR
- `AZ_Y1yTvsEtYd74_M78C`, `AZ_Y1yTvsEtYd74_M78E` ClassPropertyValuesResourceImplTest L137/L145 — the
  fix itself is clean (hoist `List.of("text")`), but the file is already touched by an open `S1130`
  agent PR, so editing it risks a merge conflict. Re-triage once that PR has merged; this is a
  timing drop, not a correctness one.

### java:S1130 (remove unthrowable `throws`) — not-a-test-method, recorded by SHAPE not by key
The platform pool is ~294 and splits cleanly, so triage it with the component path + the annotation
above the flagged line rather than key-by-key:
- **`/src/main/` (54) — permanent drop.** Narrowing a `throws` on a method other modules call breaks
  their `catch`.
- **A NON-`private`, non-annotated method inside a test class — drop.** It can be overridden by a
  subclass in another module (a `test-jar` consumer) that still declares the exception, and the
  in-module compile will not catch that. A `private` helper is NOT a drop (nothing can override or
  call it from outside the file); that distinction turned 22 "non-annotated" sites into 18 fixes.
- The annotated test-method sites are all fixed. The 4 surviving non-private drops:
  `AY974oDeKZk1650DhyN8` TempResourceActionTest L215 (`public void
  renderWithInvalidPathThrowsException()` — a public method with no `@Test`); `AY974q4wKZk1650DhyU6`,
  `AY974q4wKZk1650DhyU7` AbstractROMTestCase L135/140 (public getters on an abstract base test class);
  `AY974o4KKZk1650DhyPS` AbstractMoveJobTest L45 (`protected` method on an abstract base test class).

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

### java:S4201 (null check before instanceof) — module falls under its JaCoCo floor
- `AW5-S8LX1Yj5qvzeRoFX` DefaultUntypedRecordableEvent L75 — the fix is correct and compiles, but
  removing those covered instructions takes `xwiki-platform-eventstream-api` from 0.73 to 0.72 and
  fails `jacoco:check`. Retry only alongside tests that raise that module's coverage.

### java:S1905 (unnecessary cast) — the cast may be selecting an overload
- `AZzGqNNeXFD7Ud6sqSQJ` BaseObjectEventGenerator L105 — `(Map<String, Object>) properties` where
  `properties` is a `DocumentInstanceInputProperties` and `EntityEventGenerator.write` is overloaded.
  Dropping the cast could silently re-dispatch instead of being a no-op, so it is not mechanical.
  **General rule: an "unnecessary" cast on an argument of an OVERLOADED method is never a safe
  mechanical fix** — the compiler will not complain when it picks a different overload.

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

### java:S2589 (always-true expression) — dead DEFENSIVE checks and documented case analysis
The clean subset is a sub-expression made redundant by the *enclosing* branch (fixed). These are the
residue — every one is a deliberate defensive check or a documented exhaustive case analysis, and
removing it is a behaviour question, not a cleanup:
- Defensive null checks: `AW5-S62m1Yj5qvzeRnwQ` XWiki:6228, `AW5-S6yK1Yj5qvzeRntd` PdfExportImpl:368,
  `AW5-S6cN1Yj5qvzeRnYK` UndeleteAction:116, `AYMiXdX1XBPpoazW6ICL`/`AYMiXdX1XBPpoazW6ICM`
  DefaultTasksManager:244/256 (concurrent queue), `AZ-pzSZoUU8N_C5_mpMF` MyFormAuthenticator:317
  (CSRF path), `AW5-S5As1Yj5qvzeRmwQ` XWikiCachingRightService:249.
- Numbered 4-case truth tables whose comments ARE the documentation: `AW5-S9y71Yj5qvzeRooS`,
  `AW5-S9y71Yj5qvzeRooR`, `AW5-S9y71Yj5qvzeRooT` FileDeleteTransactionRunnable:138/143;
  `AZp0P6gbRAQTv1t4zCSU`, `AZp0P6gbRAQTv1t4zCST`, `AZp0P6gbRAQTv1t4zCSV`
  BlobDeleteTransactionRunnable:142/147.
- `AZYTQtnG6ci08M-1XWQc` DefaultModelBridge:423 — `if (!save)` is the first of several identical
  clone-once guards; clearing only the first breaks the symmetry that makes the block readable.

### java:S4973 (compare with equals) — a real-bug rule, not a cleanup (whole platform pool)
All 11 need a per-site semantic decision about whether `==` was meant as identity, so each is a JIRA
issue rather than a sweep: `AYrcilQ1bO8c88r9TGKa`-`d` CharacterDiffService:86/86/94/99 (`Difference.NONE`
sentinel), `AW5-S6YN1Yj5qvzeRnPF`/`G`/`H` XWikiDocument:5037/7193/9521 (`isHidden()`),
`AW5-S4oW1Yj5qvzeRmsg` ScriptClassLoaderHandlerListener:172, `AW5-S76z1Yj5qvzeRoB4`/`5`/`6`
DocumentSolrMetadataExtractor:281/285 (interned type constants).

### java:S1871 (duplicate branch) — the merged condition exceeds Checkstyle's complexity cap
- `AXnpAd1eDDFOvAKXAQIQ`, `AXnpAd1eDDFOvAKXAQIS`, `AXnpAd1eDDFOvAKXAQIR`
  ScopeNotificationFilterPreferencesGetter:95/97/99 — merging the four `return false` branches gives a
  `BooleanExpressionComplexity` of 10 (max 3), so the build rejects it.
- `AZbU8BJ4GHlYUfXgHgO7` RequiredRightsInfoUIExtension:266 — same, complexity 5.

### java:S3358 (nested ternary) — not hoistable / blows a Checkstyle metric
- `AW5-S6XA1Yj5qvzeRnND` AttachmentDiff:43 — the ternary is an argument of a `this(…)` delegating
  constructor call, so no statement can precede it; clearing it needs a static helper.
- `AW5-S9MC1Yj5qvzeRoWH` DefaultWikiDescriptorBuilder:221 — `save()` is already at exactly the
  `ExecutableStatementCount` cap (30); hoisting the ternary makes it 31 and fails the build.

### java:S2629 (invoke conditionally) — the platform pool needs `isXxxEnabled()` guards
**The whole rule is a drop** — see the commons section above; the "redundant `x.toString()`" shape is
NOT clean either. 26 of the 27 platform sites pass a genuine method call
(`ExceptionUtils.getRootCauseMessage(e)`, `object.getStringValue(…)`, `StringUtils.join(…)`,
`Messages.getString(…)`) and would need a level guard — a judgement call. `AW5-S7nD1Yj5qvzeRn8U`
DefaultQuery:299 is the tell that should have stopped both sessions: a comment above it already says
the stringification is deliberate *because the log is serialized into a job status*.

### java:S6126 (text block) — additional platform drops
- Merged line >120: `AYyD4wTKj2dtqk6dnyNe` MoveAttachmentDocumentInitializer:102,
  `AYyD4p_jj2dtqk6dnyGU` HibernateDataMigrationManager:332 (the `xsi:schemaLocation` pair merges to 144).
- `\r\n` line endings, which a text block cannot reproduce without escapes: `AY974jt8KZk1650Dhx4W`
  JavaIntegrationTest:292, `AY974juUKZk1650Dhx4Y` ScriptingIntegrationTest:248.
- Build ROI only (the conversion is byte-safe): `AY974ns7KZk1650DhyMP` DocumentSolrMetadataExtractorTest:713
  — one site would add the slow search-solr-api module to the reactor.

### java:S125 (commented-out code) — src/main is nearly all ANCHORED; test files are the clean subset
Anchored by a `TODO`/`FIXME` or by a comment explaining why the code is commented out:
`AW5-S9_r1Yj5qvzeRos3` FeedPlugin:735, `AW5-S50E1Yj5qvzeRm5j` Document:3206, `AW5-S6WL1Yj5qvzeRnMv`
XWikiRCSArchive:209, `AW5-S6fT1Yj5qvzeRnaP` CancelAction:60, `AW5-S6jq1Yj5qvzeRnfA` SaveAction:174,
`AW5-S6d91Yj5qvzeRnZb` XWikiServletURLFactory:448.
Explanatory leftovers documenting mock argument positions: `AY974qjEKZk1650DhyT3`,
`AY974qjEKZk1650DhyT4` SyndEntryDocumentSourceTest:155/157.
**False positive** (a descriptive prose comment Sonar read as code): `AXnpAfAcDDFOvAKXAQOz` EditForm:522.

### java:S1450 (field → local) — the field outlives the method Sonar sees it in
Sonar's "only used in one method" is about *syntactic* occurrences, not lifetime: a field written and
read inside one lifecycle method still exists so the component can be restarted or torn down, and
demoting it to a local silently changes object lifetime. All `src/main`.
- `AW5-S8FO1Yj5qvzeRoET` DefaultOfficeServer:86 `jodConverter` — assigned in three places across the
  start/stop path (`= null` in one method, rebuilt in another) and handed to `DefaultOfficeConverter`.
- `AW5-S79J1Yj5qvzeRoCr` / `AW5-S79J1Yj5qvzeRoCs` DefaultSolrIndexer:401/406 `indexThread` /
  `resolveThread` — the component's handles to its own daemon threads; losing them is a design change.
- `AW5-S6wR1Yj5qvzeRnro` StatsUtil:182 `cookieExpirationDate` is the ONE viable site of the four (a
  `private static` written and returned in the same method, nothing else reads it) — deferred, not
  dropped, because one issue does not pay for an oldcore reactor. Take it as a rider.

### java:S1121 (extract the assignment) — deferred, viable, needs a repository-server-api build
`AW5-S-g71Yj5qvzeRo3o` / `AW5-S-g71Yj5qvzeRo3p` XWikiRepositoryModel:572/575
(`repository.substring(0, index = repository.indexOf(':'))`). The rewrite is mechanical and
order-preserving — hoist each `indexOf` into its own `int` local before the `substring` that uses it —
but it is `src/main` and only worth doing as a rider on a reactor that already builds that module.

### java:S3252 (static access via a derived type) — the derived type is XWiki's own utility subclass
`org.xwiki.text.StringUtils` and `org.xwiki.localization.LocaleUtils` deliberately **extend** the
Apache Commons classes of the same simple name so that one import serves both the Apache helpers and
the XWiki additions. Sonar flags every inherited call made through them. "Fixing" one means swapping
the import for the base class — a style regression, and impossible without an FQN in any file that
also uses the XWiki-specific methods. Permanent drop for the whole shape; the other 85 platform
`S3252` sites (a genuine derived-type qualifier) were fixed.
- `StringUtils` (47): `AYjDs5BtPAk_qAE7Pepl`, `AXnpAd_3DDFOvAKXAQJQ`, `AW5-S5bz1Yj5qvzeRm1M`,
  `AW5-S5bz1Yj5qvzeRm1O` DefaultNotificationPreferenceModelBridge; `AW5-S5ev1Yj5qvzeRm1w`,
  `AW5-S5ev1Yj5qvzeRm1x`, `AW5-S5ev1Yj5qvzeRm1y`, `AW5-S5ev1Yj5qvzeRm1z` LogoAttachmentExtractor;
  `AW5-S9401Yj5qvzeRoo8`, `AW5-S9401Yj5qvzeRoo9`, `AW5-S9401Yj5qvzeRoo-`
  DefaultStringEntityReferenceSerializer; `AW5-S5TW1Yj5qvzeRm0W`, `AW5-S5TW1Yj5qvzeRm0Z`,
  `AW5-S5TW1Yj5qvzeRm0Y` DefaultWatchedEntitiesConfiguration; `AW5-S5eD1Yj5qvzeRm1l`,
  `AW5-S5eD1Yj5qvzeRm1m`, `AW5-S5eD1Yj5qvzeRm1n` DefaultNotificationEmailRenderer;
  `AW5-S7eg1Yj5qvzeRn8K`, `AW5-S7eg1Yj5qvzeRn8L`, `AW5-S7eg1Yj5qvzeRn8M` AbstractQueryFilter;
  `AX0whn23NVa5BHo6Y8yN`, `AX0whn23NVa5BHo6Y8yO` StaticListClassPropertyValuesProvider;
  `AW5-S7Sx1Yj5qvzeRn6v`, `AW5-S7Sx1Yj5qvzeRn6w` DistributionInternalScriptService;
  `AW5-S5_n1Yj5qvzeRm9j`, `AW5-S5_n1Yj5qvzeRm9k` ImplicitlyAllowedValuesPageQueryBuilder;
  `AW5-S7d31Yj5qvzeRn8C`, `AW5-S7d31Yj5qvzeRn8D` CountDocumentFilter; `AW5-S4Vs1Yj5qvzeRmnU`,
  `AW5-S4Vs1Yj5qvzeRmnV` AbstractClassPropertyValuesProvider; `AW5-S5Yr1Yj5qvzeRm03`,
  `AW5-S5Yr1Yj5qvzeRm04` DefaultNotificationFilterPreference; and one each `AYAqYBvjTBPgKLwK3BDS`
  IncludeMacroRefactoring, `AXnpAh1uDDFOvAKXAQzG` DefaultQuoteService, `AXnpAd-DDDFOvAKXAQJG`
  NotificationPreferenceScriptService, `AXnpAfjUDDFOvAKXAQfK` SecureDocumentConfigurationSource,
  `AXE7zQJN6SPgukNyCE31` FileUploadPlugin, `AW5-S8If1Yj5qvzeRoE6`
  DefaultUntypedRecordableEventDescriptor, `AW5-S7TU1Yj5qvzeRn62` DefaultVersionCheckConfiguration,
  `AW5-S5UM1Yj5qvzeRm0f` EventUserFilter, `AW5-S5Yi1Yj5qvzeRm01` WikiNotificationFilterDisplayer,
  `AW5-S5fe1Yj5qvzeRm16` AbstractWikiNotificationRenderer, `AW5-S5aO1Yj5qvzeRm1C`
  AbstractNotificationPreference, `AW5-S5gh1Yj5qvzeRm2R` InternalNotificationsRenderer,
  `AW5-S5_E1Yj5qvzeRm9T` DefaultDBListQueryBuilder, `AW5-S5_91Yj5qvzeRm9m` DefaultPageQueryBuilder,
  `AW5-S-VT1Yj5qvzeRo1L` StackTraceLogParser.
- `LocaleUtils` (3): `AW5-S62m1Yj5qvzeRnyd`, `AW5-S62m1Yj5qvzeRnyg`, `AW5-S62m1Yj5qvzeRnyl` XWiki.java.

### java:S3824 (computeIfAbsent) — not an equivalent rewrite (8 of the 35 platform sites)
The other 27 converted cleanly. Three distinct disqualifiers, all visible in the flagged block:
- **The mapping function can return `null`**, which `computeIfAbsent` does not store, so the current
  code's caching of a negative result is lost: `AW5-S4nz1Yj5qvzeRmsY` AbstractJSR223ScriptMacro:315
  (`scriptEngineManager.getEngineByName`).
- **The guarded block does more than the single `put`** — it also feeds a second collection/map, or
  needs a multi-statement body that would have to throw: `AW5-S6v_1Yj5qvzeRnrV`
  XWikiStatsStoreService:152, `AXnpAhXIDDFOvAKXAQwP` XClassBreakingQuestion:105,
  `AW5-S4591Yj5qvzeRmu9` Right:297, `AW8Kv7iBPTv0sE1_7izG` XWikiAttachmentRCSArchive:93.
- **The mapping function would call another component from inside the map operation**:
  `AW5-S6qr1Yj5qvzeRnlc` RightsManager:1263 (group service),
  `AYj3Kveh6Rs8CIfHrNru` CachedNotificationPreferenceModelBridge:149 (preference bridge, and its
  hint can be `null`).
- **The flagged `get()` is a save/restore of a context value, not a lazy init**:
  `AW5-S7bh1Yj5qvzeRn7x` DefaultWikiComponentMethodExecutor:162.

### java:S1872 (compare classes by name) — `instanceof` is not equivalent
`ApplicationStartedEvent.class.getName().equals(event.getClass().getName())` matches exactly one
class; `instanceof` also matches subclasses, and comparing `Class` objects instead of names changes
behaviour across class loaders (extension jars). A per-site semantic decision, not a cleanup.
- `AW5-S8FG1Yj5qvzeRoEQ`, `AW5-S8FG1Yj5qvzeRoER` OfficeServerLifecycleListener:84/86.
- **Correction:** `AW5-S7bh1Yj5qvzeRn71` DefaultWikiComponentMethodExecutor:189 was listed here and
  should not have been — `"void".equals(returnType.getName())` → `void.class.equals(returnType)` is a
  comparison against a *fixed known* class, not an `instanceof`, so it is exactly equivalent. Fixed.
  See [rules/java-S1872.md](rules/java-S1872.md).

### java:S1193 (instanceof in a catch → multi-catch) — empties a `catch`
The XWiki shape is `catch (Exception e) { if (e instanceof InterruptedException) { interrupt(); }
throw new XxxException(…, e); }`.
- `AW5-S5ct1Yj5qvzeRm1Y` LiveNotificationEmailListener:99 — here the `if` is the whole body, so the
  split leaves an empty `catch (IllegalArgumentException)`, i.e. a fresh S108/S1186. Permanent drop.
- **Correction:** `AYK2yKxsjX57U6EJhxnU`, `AYK2yKxsjX57U6EJhxnW` XarExtensionHandler:178/203 were
  dropped for "duplicates the wrapping `throw`" — that is a solvable problem, not a drop condition:
  extract the message into a constant in the same edit and the duplicated `throw` costs nothing.
  Fixed. See [rules/java-S1193.md](rules/java-S1193.md).

### java:S6916 (pattern match guard) — FALSE POSITIVE on an enum `switch`
Java 21 allows a `when` guard only on a *pattern* case label; `case BLOCKED, X when cond ->` does not
compile. Permanent drop for every enum-`switch` site.
- `AZqCQr1OWrST1JcpjTkl`, `AZqCQr1OWrST1JcpjTkm` ScopeNotificationFilter:101/107.

### java:S100 (method naming) — every platform site is a PUBLIC API method
All 24 open sites are `public`/`protected` methods of published oldcore classes (`XWiki` 7,
`XWikiAttachment` 4, `DefaultXWikiDocumentMerger` 4, `Package`/`PackageAPI` 4, `ScopeFactory` 3,
`PeriodFactory`, `Utils`), so the rename is a Revapi break. Same verdict as the denylisted `S115`.
- `AZ9R1gSKWw4wwGCX_PiV`, `AW5-S7Kr1Yj5qvzeRn5D`, `AW5-S7Kr1Yj5qvzeRn5E`, `AW5-S7Kr1Yj5qvzeRn5G`,
  `AW5-S7Kr1Yj5qvzeRn5I`, `AW5-S6tI1Yj5qvzeRnoN`, `AW5-S6tf1Yj5qvzeRnpK`, `AW5-S6tf1Yj5qvzeRnpM`,
  `AW5-S62n1Yj5qvzeRn2T`, `AW5-S62n1Yj5qvzeRn2V`, `AW5-S62n1Yj5qvzeRn2X`, `AW5-S62n1Yj5qvzeRn2Z`,
  `AW5-S6Vq1Yj5qvzeRnL_`, `AW5-S6Vq1Yj5qvzeRnMA`, `AW5-S6Vq1Yj5qvzeRnMB`, `AW5-S6Vq1Yj5qvzeRnMC`,
  `AW5-S6yZ1Yj5qvzeRntx`, `AW5-S6yi1Yj5qvzeRnt0`, `AW5-S6yi1Yj5qvzeRnt1`, `AW5-S6yi1Yj5qvzeRnt2`,
  `AW5-S6hc1Yj5qvzeRndD`, `AW5-S62m1Yj5qvzeRnx4`, `AW5-S62m1Yj5qvzeRnx5`, `AW5-S62m1Yj5qvzeRnx6`.

### Platform one-off rules triaged from the message and rejected (no source read)
- **`java:S106`** replace `System.out`/`System.err` by a logger (4) — the sites are a test-support
  capture helper and a CLI-style utility where printing to the console *is* the contract:
  `AW5-S9v11Yj5qvzeRong`, `AW5-S-Ve1Yj5qvzeRo1N`, `AW5-S-Ve1Yj5qvzeRo1O`, `AW5-S-L11Yj5qvzeRoxx`.
- **`java:S3078`** use an `AtomicInteger` (2) — changes the concurrency contract of
  `DefaultSolrIndexer`: `AW5-S79J1Yj5qvzeRoC2`, `AW5-S79J1Yj5qvzeRoC3`.
- **`java:S4348`** make the `Iterator` support multiple traversal (3) — a redesign of the mail
  iterators: `AW5-S5Ni1Yj5qvzeRmzq`, `AW5-S5Mm1Yj5qvzeRmzk`, `AW5-S5fA1Yj5qvzeRm11`.
- **`java:S6912`** use `addBatch`/`executeBatch` (2) — see the S6912 section above:
  `AZf1b0yb-3Rl8fL0EFUS`, `AZf1b0yb-3Rl8fL0EFUT`.
- **`java:S6019`** reluctant quantifier matching 0 repetitions (2) — a real regex bug needing a
  per-site decision about what the pattern was meant to match, i.e. a JIRA issue:
  `AXnpAeoLDDFOvAKXAQNA` XWikiAuthServiceImpl:548, `AXnpAe7ADDFOvAKXAQOS` XWikiServletURLFactory:1007.

### javascript:* — VENDORED third-party scripts in `xwiki-platform-web-war` (ANY rule, permanent)
`xwiki-platform-web-war` redistributes three scripts XWiki does not maintain; each carries its
upstream banner instead of the standard XWiki LGPL "See the NOTICE file" header. **Check the header
of every WAR JS file before editing it** — this drop applies to every rule, not just the ones listed.
- `js/xwiki/table/tablefilterNsort.js` (Guglielmi / de Valk / Eldenmalm)
- `uicomponents/widgets/validation/livevalidation_prototype.js` (LiveValidation 1.4, MIT)
- `js/xwiki/panelwizard/ieemu.js` (WebFX / Erik Arvidsson, "IE Emu")

Keys seen so far: `S6535` ieemu `AY1U1sPk0GHv9uFD3ja-`; `S6535` tablefilterNsort
`AY1U1sNB0GHv9uFD3jPT`, `AY1U1sNB0GHv9uFD3jPU`, `AY1U1sNB0GHv9uFD3jPV`, `AY1U1sNB0GHv9uFD3jPW`,
`AY1U1sNB0GHv9uFD3jPX`; `S1125` tablefilterNsort `AY1U1sNB0GHv9uFD3jQI`, `AY1U1sNB0GHv9uFD3jRB`,
`AY1U1sNB0GHv9uFD3jQq`, `AY1U1sNB0GHv9uFD3jQw`, `AY1U1sNB0GHv9uFD3jQ5`; `S4030`
`AY1U1sNB0GHv9uFD3jP6`; `S6644` `AY1U1sNB0GHv9uFD3jRl`; `S7762` `AZlyHalLlYnK6j8fkQai`;
`S6660` livevalidation `AY1U1sQy0GHv9uFD3jfV`; `S7759` livevalidation `AZlyHawolYnK6j8fkQdy`.

### javascript:S1125 (boolean literal in a comparison) — false positive on a non-boolean operand
The rule assumes the compared expression is a boolean. It is not always.
- `AY1U1sOk0GHv9uFD3jWK` `js/xwiki/create.js:108` — `if (targetName != false)` where
  `computeTargetPageName()` returns a serialized reference (`String`) **or** `false`. Dropping the
  literal (`if (targetName)`) silently changes behaviour for a page named `0` or `''`; keeping a
  strict `!== false` does not remove the literal. Permanent drop.

### javascript:S7773 (`Number.*` over the global functions) — the vendored keys
Superseded by the vendored-scripts entry above; kept for the exact key list.
- `js/xwiki/table/tablefilterNsort.js`: `AZlyHalLlYnK6j8fkQaP`,
  `AZlyHalLlYnK6j8fkQaQ`, `AZlyHalLlYnK6j8fkQaR`, `AZlyHalLlYnK6j8fkQaS`, `AZlyHalLlYnK6j8fkQaT`,
  `AZlyHalLlYnK6j8fkQaU`, `AZlyHalLlYnK6j8fkQaV`, `AZlyHalLlYnK6j8fkQaW`, `AZlyHalLlYnK6j8fkQaX`,
  `AZlyHalLlYnK6j8fkQan`, `AZlyHalLlYnK6j8fkQao`, `AZlyHalLlYnK6j8fkQar`, `AZlyHalLlYnK6j8fkQas`,
  `AZlyHalLlYnK6j8fkQat`, `AZlyHalLlYnK6j8fkQau`, `AZlyHalLlYnK6j8fkQav`, `AZlyHalLlYnK6j8fkQaw`,
  `AZlyHalLlYnK6j8fkQax`, `AZlyHalLlYnK6j8fkQay`, `AZlyHalLlYnK6j8fkQaz`.
- `uicomponents/widgets/validation/livevalidation_prototype.js`:
  `AZlyHawolYnK6j8fkQdz`, `AZlyHawolYnK6j8fkQd0`.

### java:S4165 (useless assignment) — hides a probable bug / the call must stay
- `AW5-S6wR1Yj5qvzeRnrm` StatsUtil:410 — `VisitStats newVisitObject = visitObject; … visitObject =
  newVisitObject;` is a self-assignment whose sibling block above does `visitObject.clone()`.
  Removing the flagged line papers over a likely missing `clone()`; that is a JIRA issue.
- `AW5-S4oW1Yj5qvzeRmsf` ScriptClassLoaderHandlerListener:173 — the right-hand side
  `createOrExtendClassLoader(...)` has side effects, so only the `cl =` prefix could go, which reads
  worse than the current code.

### java:S3400 / java:S4165 — the judgement-call PR was CLOSED without merging. Whole rules: DROP
PR #6179 carried exactly these three and was closed silently (no comment) while its mechanical
sibling #6178 merged the same hour. So in xwiki-platform, replacing a documented `private`
constant-returning method with a constant, and deleting a "redundant" defensive re-assignment, are
both unwanted — the reviewer's answer, not a guess. Treat both rules as permanent drops here.
- `AW5-S6zO1Yj5qvzeRnuA` Scope:162 (`getGlobalPattern()`), `AW5-S6ls1Yj5qvzeRngN`
  HibernateDataMigrationManager:346 (`getLiquibaseChangeLogFooter()`),
  `AYZ7IjPH9c9uyqlx3uGB` XWikiRightServiceImpl:781 (`hasDenyRights()`) — java:S3400.
- `AW5-S6s91Yj5qvzeRnnN` DocumentInfo:141 — java:S4165 (the other two S4165 sites are above).

### java:S3415 (swap assertion operands) — `null` on one side IS the test
Same shape as the commons/rendering entries above: `assertNotEquals(new X(...), null)` deliberately
exercises `x.equals(null)`, and swapping short-circuits inside `Objects.equals`. **This is the rule's
only common drop condition** — see [rules/java-S3415.md](rules/java-S3415.md); 15 of 18 platform sites
were clean, so do NOT treat the rule as a drop pool.
- `AYUsLksItp4wiQKQFvG-` CodeMacroSourceTest:53, `AXG6od5sUBz12AiapMr8` DefaultEventTest:47 (which
  asserts both directions on consecutive lines — the point of the test),
  `AW5-S8qr1Yj5qvzeRoOB` DocumentRenamingEventTest:54.

### javascript:S7781 (`replaceAll`) — third-party file
`js/xwiki/panelwizard/ieemu.js` is IE Emu, © Erik Arvidsson / WebFX — XWiki redistributes it rather
than maintaining it, like the other vendored WAR scripts. Permanent drop.
- `AZlyHat4lYnK6j8fkQdN`, `AZlyHat4lYnK6j8fkQdO`, `AZlyHat4lYnK6j8fkQdP`, `AZlyHat4lYnK6j8fkQdQ`.

### javascript:S6660 (`if` alone in an `else` block) — merging would re-indent a long block
Mechanically correct, but the dedent moves every line of a 30+-line `if/else if` chain, so the diff
dwarfs the fix for a cosmetic rule. The two short sites in `dashboard.js`/`suggest.js` shipped.
- `AY1U1sR60GHv9uFD3jkv` suggest.js:902, `AY1U1sR60GHv9uFD3jk2` suggest.js:987.

### javascript:S2201 (unused return value) — the `reduce()` IS the sequencing
`suggestAttachments.js:425` threads a promise chain through the accumulator to upload files
sequentially (per the comment naming XWIKI-13473), so "use the result" is a design decision.
`actionButtons.js:44` is in a file an open agent PR already claims.
- `AZIFl7cT2p5gib56Ksjp`, `AZIFl7Su2p5gib56Ksjo`.

### javascript:S1125 (boolean literal) — false positive in these scripts
Confirmed on two more sites: the flagged operand is a `String`-or-`false` return value, not a boolean,
so the "redundant" comparison is doing real work. Treat the rule as a drop pool in the WAR scripts.
- `AY1U1sK70GHv9uFD3jM3` xwiki.js:533, `AY1U1sK70GHv9uFD3jM9` xwiki.js:725.

### javascript:S6637 (unnecessary `.bind(this)`) — DEFERRED, not dropped
Six sites, all in `uicomponents/dashboard/dashboard.js` (lines 368, 376, 481, 489, 525, 636). Each
needs its own check that the callback truly does not use `this`, and per
[rules/javascript-S6637.md](rules/javascript-S6637.md) the flagged text is never unique. A viable next
batch, not a drop — do not skip these keys, triage them.
- `AY1U1sV50GHv9uFD3jwm`, `AY1U1sV50GHv9uFD3jwo`, `AY1U1sV50GHv9uFD3jw0`, `AY1U1sV50GHv9uFD3jw2`,
  `AY1U1sV50GHv9uFD3jw7`, `AY1U1sV50GHv9uFD3jxQ`.


### java:S3457 (`No need to call "toString()"`) — the eager String is load-bearing (see [rules/java-S3457.md](rules/java-S3457.md))
A log argument is XStream-serialized into the job log, so an explicit `toString()` is normally a
deliberate snapshot. **12 of these 16 already carry a call-site comment saying exactly that**, which
makes the universal "a comment explains it" condition the cheap classifier for the whole shape. The
four without a comment are drops on their own terms: an `Object[]` query row (`result[2]`, type
unknown at compile time), a `java.net.URI`, and a `Version` from an extension. The clean subset of this
shape is only the convention's "pass the object" list (a `File`, an enum, a model reference, …) — those
two sites shipped in #6200. Sites: `MovedAttachmentListener`, `DatabaseMailStatusStore`,
`MailSenderPlugin` ×3, `MailSenderPluginApi` ×3, `JGroupsNetworkAdapter` ×2, `XWikiHibernateBaseStore`,
`DefaultQuery`, `ExtensionVersionFileRESTResource`, `R140600000XWIKI19869DataMigration`,
`R4359XWIKI1459DataMigration` ×2.
  - `AZ_duy1yaa4LEs-MEU4g`, `AZ_XP-5FkxIS6rT1tShz`, `AZ_XQFXTkxIS6rT1tSh7`,
    `AZ_XQFXTkxIS6rT1tSh8`, `AZ_XQFXTkxIS6rT1tSh9`, `AZ_XQFW8kxIS6rT1tSh4`,
    `AZ_XQFW8kxIS6rT1tSh5`, `AZ_XQFW8kxIS6rT1tSh6`, `AZ_XP9aNkxIS6rT1tShw`,
    `AZ8-G6caE9D-RjRrnZsi`, `AYQ5rjEL-8V8J8eaPoCz`, `AYMiXeniXBPpoazW6ICP`,
    `AXnpAikEDDFOvAKXAQ4f`, `AXnpAikEDDFOvAKXAQ4g`, `AW5-S6l11Yj5qvzeRngV`,
    `AW5-S7nD1Yj5qvzeRn8a`

### java:S3457 (`%n` should be used in place of `\n`) — behaviour change on Windows
Same verdict as commons and rendering, now confirmed for all 8 platform sites: the produced string is
either asserted by a test (`DefaultAuthenticationFailureManagerTest` ×3, `CssSkinExtensionPluginTest`,
`R140600000XWIKI19869DataMigrationListenerTest`) or is content compared downstream
(`XWikiRestExceptionMapper`, `SxDocumentSource` — a CSS comment, `LogCaptureValidator`).
  - `AY974nHjKZk1650DhyJU`, `AY974qqWKZk1650DhyUQ`, `AY974nHHKZk1650DhyJQ`,
    `AW5-S4ep1Yj5qvzeRmq9`, `AY974nHHKZk1650DhyJP`, `AY974nHHKZk1650DhyJR`,
    `AW5-S-Cd1Yj5qvzeRot3`, `AW5-S-Vp1Yj5qvzeRo1T`

### java:S3457 (`String contains no format specifiers` on a `Formatter`) — no cheap equivalent
All 9 in `DatabaseKeywordSearchSource`, which builds an HQL query through a `java.util.Formatter`.
`Formatter` has no `append`, so dropping the pointless `f.format("<literal>")` means going through
`f.out()`, which throws `IOException` — a refactor, not a cleanup. (The *other* no-format-specifier
shape, an SLF4J `{}` inside a `String.format`, IS a clean fix — see the rule file.)
  - `AZeKdlNsdMqt9rTWYu_v`, `AZeKdlNsdMqt9rTWYu_w`, `AZeKdlNsdMqt9rTWYu_x`,
    `AZeKdlNsdMqt9rTWYu_y`, `AZeKdlNsdMqt9rTWYu_z`, `AZeKdlNsdMqt9rTWYu_0`,
    `AZeKdlNsdMqt9rTWYu_1`, `AZeKdlNsdMqt9rTWYu_3`, `AZeKdlNsdMqt9rTWYu_4`

### java:S3457 (format specifiers instead of concatenation) — a test asserts the RAW log message
- `AXnpAiy-DDFOvAKXAQ57` LoggingScriptService L207 — `warn("[DEPRECATED] " + message)`. The SLF4J
  parameterized form renders identically but changes `LogEvent.getMessage()`, and
  `LoggingScriptServiceTest#deprecate` asserts that exact concatenated string. Tried, caught by the
  build, reverted.

### javascript:S7721 (move functions to the highest possible scope) — a restructuring, not a cleanup
Hoisting a function out of a `Class.create` body or an IIFE moves it out of the scope that makes it
private in these Prototype-era WAR scripts, and the diff is a file reorganisation. 61 keys (38 of them
in files no other PR claims); permanent drop for the whole rule in this codebase.
  - `AZ-LGl-7RQLNEUYjlwfW`, `AZpXUsyKZxcWRO8GXhsN`, `AZpXUsyKZxcWRO8GXhsO`,
    `AZpXUsyKZxcWRO8GXhsP`, `AZnonzjIpn1CLOvpJF2Y`, `AZlyHcCklYnK6j8fkQgY`,
    `AZlyHcCklYnK6j8fkQgZ`, `AZlyHaSUlYnK6j8fkQXN`, `AZlyHaSUlYnK6j8fkQXS`,
    `AZlyHaSUlYnK6j8fkQXT`, `AZlyHaSUlYnK6j8fkQXU`, `AZlyHaqrlYnK6j8fkQbo`,
    `AZlyHaNzlYnK6j8fkQWi`, `AZlyHat4lYnK6j8fkQdK`, `AZlyHat4lYnK6j8fkQdM`,
    `AZlyHaUzlYnK6j8fkQXZ`, `AZlyHaU0lYnK6j8fkQZ2`, `AZlyHaU0lYnK6j8fkQZ4`,
    `AZlyHaU0lYnK6j8fkQZ5`, `AZlyHa52lYnK6j8fkQew`, `AZlyHa8nlYnK6j8fkQfZ`,
    `AZlyHa8nlYnK6j8fkQfa`, `AZlyHa8nlYnK6j8fkQfb`, `AZlyHbAwlYnK6j8fkQgJ`,
    `AZlyHbAUlYnK6j8fkQgC`, `AZlyHbAUlYnK6j8fkQgD`, `AZlyHbAUlYnK6j8fkQgE`,
    `AZlyHbIxlYnK6j8fkQgP`, `AZlyHbADlYnK6j8fkQfz`, `AZlyHbADlYnK6j8fkQf2`,
    `AZlyHbADlYnK6j8fkQgA`, `AZlyHa6NlYnK6j8fkQe2`, `AZlyHa-ZlYnK6j8fkQfj`,
    `AZlyHa-ZlYnK6j8fkQfl`, `AZlyHa88lYnK6j8fkQfc`, `AZlyHa88lYnK6j8fkQff`,
    `AZlyHa88lYnK6j8fkQfg`, `AZlyHa88lYnK6j8fkQfh`, `AZlyHa8GlYnK6j8fkQfU`,
    `AZlyHa1glYnK6j8fkQeG`, `AZlyHa1glYnK6j8fkQeL`, `AZlyHa1glYnK6j8fkQeM`,
    `AZlyHa1glYnK6j8fkQeN`, `AZlyHa1glYnK6j8fkQeO`, `AZlyHa1glYnK6j8fkQeT`,
    `AZlyHa1ClYnK6j8fkQeD`, `AZlyHa1ClYnK6j8fkQeE`, `AZlyHa0zlYnK6j8fkQeC`,
    `AZlyHa49lYnK6j8fkQeq`, `AZlyHa5TlYnK6j8fkQer`, `AZlyHa5TlYnK6j8fkQet`,
    `AZlyHa5TlYnK6j8fkQeu`, `AZlyHa8WlYnK6j8fkQfW`, `AZlyHa-4lYnK6j8fkQfp`,
    `AZlyHa_ilYnK6j8fkQft`, `AZlyHa_ilYnK6j8fkQfw`, `AZlyHau1lYnK6j8fkQdi`,
    `AZlyHau1lYnK6j8fkQdj`, `AZlyHau1lYnK6j8fkQdk`, `AZlyHau1lYnK6j8fkQdl`,
    `AZlyHau1lYnK6j8fkQdm`

### javascript:* — VENDORED third-party scripts, keys seen on this pass (permanent, ANY rule)
Addendum to the vendored-scripts section above, from a full `javascript:` allowlist pull:
`tablefilterNsort.js`, `livevalidation_prototype.js`, `ieemu.js`. The module pom's
`license-maven-plugin` excludes list (`accordion.js`, `ieemu.js`, `tablefilterNsort.js`,
`livevalidation_prototype.js`) is the authoritative enumeration — cheaper than reading headers.
  - `AZlyHat4lYnK6j8fkQdK`, `AZlyHat4lYnK6j8fkQdL`, `AZlyHat4lYnK6j8fkQdM`,
    `AZlyHalLlYnK6j8fkQaJ`, `AZlyHalLlYnK6j8fkQaK`, `AZlyHalLlYnK6j8fkQaM`,
    `AZlyHalLlYnK6j8fkQaN`, `AZlyHalLlYnK6j8fkQaY`, `AZlyHalLlYnK6j8fkQaZ`,
    `AZlyHalLlYnK6j8fkQaa`, `AZlyHalLlYnK6j8fkQab`, `AZlyHalLlYnK6j8fkQac`,
    `AZlyHalLlYnK6j8fkQae`, `AZlyHalLlYnK6j8fkQaf`, `AZlyHalLlYnK6j8fkQag`,
    `AZlyHalLlYnK6j8fkQaj`, `AZlyHalLlYnK6j8fkQak`, `AZlyHalLlYnK6j8fkQal`,
    `AZlyHalLlYnK6j8fkQam`, `AZlyHawolYnK6j8fkQd1`, `AZlyHawolYnK6j8fkQd2`,
    `AY1U1sQy0GHv9uFD3jfc`, `AY1U1sNB0GHv9uFD3jPp`, `AY1U1sNB0GHv9uFD3jQQ`,
    `AY1U1sQy0GHv9uFD3jfi`, `AY1U1sQy0GHv9uFD3jfk`, `AY1U1sQy0GHv9uFD3jfn`,
    `AY1U1sQy0GHv9uFD3jfs`, `AY1U1sQy0GHv9uFD3jfu`, `AY1U1sNB0GHv9uFD3jPD`,
    `AY1U1sNB0GHv9uFD3jPK`, `AY1U1sNB0GHv9uFD3jPe`, `AY1U1sNB0GHv9uFD3jPf`,
    `AY1U1sNB0GHv9uFD3jPi`, `AY1U1sNB0GHv9uFD3jPl`, `AY1U1sNB0GHv9uFD3jPn`,
    `AY1U1sNB0GHv9uFD3jPr`, `AY1U1sNB0GHv9uFD3jPw`, `AY1U1sNB0GHv9uFD3jQX`,
    `AZv8Dn_JFzGovcQ6Xjte`, `AY1U1sNB0GHv9uFD3jQg`, `AY1U1sNB0GHv9uFD3jQ8`,
    `AY1U1sNB0GHv9uFD3jQ-`, `AY1U1sNB0GHv9uFD3jRA`, `AY1U1sNB0GHv9uFD3jRW`,
    `AY1U1sNB0GHv9uFD3jRb`, `AY1U1sNB0GHv9uFD3jRg`, `AY1U1sNB0GHv9uFD3jRq`,
    `AY1U1sNB0GHv9uFD3jRr`, `AY1U1sNB0GHv9uFD3jRu`, `AY1U1sNB0GHv9uFD3jRv`
