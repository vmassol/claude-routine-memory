# Pool state (VOLATILE — re-query before trusting any number here)

Where each rule's issues were last observed, how they cluster, and the drop rate actually seen. This
is the part of the old `rules/*.md` files that must NOT live in the OKF, because it is stale within
days. **The fix itself, and the drop conditions, are in the OKF** —
[`xwiki/okf/sonarqube/`](https://github.com/xwiki/xwiki-dev-llm/tree/master/xwiki/okf/sonarqube).

Every count below is a *last-seen* observation, not a fact. Confirm with
`issues/search?...&rules=java:SXXXX&ps=1` → `total` before planning a batch.

## The standing shape of the pools

- **A rule on the OKF DENYLIST can still be the run's best pool — re-read the denylist reason against
  the rule's own definition.** `S3252` was listed as "static-access … usually backward-compat-bearing
  public API"; the rule actually only asks you to qualify a static member with the class that declares
  it, which changes no declaration at all. One `api/rules/show?key=java:SXXXX` call (name + compliant
  example) settles it, and it re-opened 123 clean CRITICAL sites across three repos on a day when the
  whole classic allowlist read platform 61 / commons 19 / rendering 6, nearly all already dropped.
  This is the second denylist entry found wrong after `S1117`; both were rules that *sound* like
  renames but are not.
- **The mechanical pool is usually drained in PLATFORM but untouched in the SIBLINGS.** Platform's
  whole classic allowlist can total under 60 with nearly all of it already in `dropped-issues.md`,
  while commons and rendering still hold hundreds. On a multi-repo run, spend the effort in
  commons + rendering and treat platform as the small remainder. **The same rule in commons/rendering
  is the single most reliable lever** when a rule reads 0 in platform.
- **The classic allowlist is now EXHAUSTED in all three repos.** The `xwiki-commons-extension` tree
  — the last dense pool — was swept in one 99-site PR (54 S6201 + 16 S1066 + 8 S6208 + 7 S7476 +
  3 S6485 + 3 S1488 + 2 S1126 + 2 S1604 + 1 each S1612/S3706/S6204/S1068). A 4-module `-pl` reactor
  (extension-api, -maven, -handler-jar, -repository-maven) builds it in one shot. What is left of
  the 39-rule allowlist reads: commons ~15 (scattered singletons + the 4 crypto-cipher and 4
  legacy-classloader permanent drops), rendering **1** (a dropped key), platform **18** and nearly
  all of those are already in `dropped-issues.md`.
- **The next generation of pools is the TEST-CODE rules** and they are proving to be the best fodder
  since the classic allowlist dried up: `S9016`, `S6068` and `S8714` are all zero-risk (test sources
  only, the module's own suite is the complete verification) and all three are scriptable. Rendering
  has no volume at all — its only viable fodder is a handful of one-off mechanical rules
  (S2209/S3012/S3024/S1264 gave 9 in one PR; S1117/S2094/S4719/S8714 gave 5 in another).
  **Platform `S8714` is now spent too**: all 49 went in one PR (21 files, 12 modules, 0 drops), so
  the whole test-code generation (`S9016`, `S6068`, `S8714`) is drained in every repo. What is left
  project-wide is the *long tail of one-off mechanical rules*, which is still worth ~20 per repo-sweep.
- **When the deep pools are gone, a repo's LONG TAIL of one-per-rule mechanical singletons is still a
  viable batch**: commons yielded 17 in one PR from seven rules (`S4719` 9, `S1905` 3, then one each of
  `S2147`/`S1596`/`S1481`/`S1264`/`S1126`) and rendering 4 from three rules. Pull the whole
  distribution facet, take every rule whose *message* is a one-line mechanical instruction, and read
  the ~15 sites; the drop rate on this shape has been 0.
- **Classifying an unswept rule costs one turn:** batch one `ps=2` query per candidate rule key and
  read only `message`. Ten rules per turn; then check the OKF denylist before committing.
- **Query a whole ~28-rule allowlist in ONE call per repo and group by module** (`&rules=a,b,c…&ps=500`)
  instead of rule-by-rule. Three calls give the entire cross-repo plan, and a repo whose allowlist
  totals 20-30 is finishable in a single PR rather than needing target selection.
- **oldcore's dense count is a TRAP for the small clean rules** (S1118/S1144/S1185/S1192/S3626/S6204/
  S1068). oldcore is heavily scanned, so its EASY hits are long fixed and a high count there means
  *residue*, not cheap fixes. One run found 26 oldcore "mechanical" issues across 9 rules and nearly
  all were drops. Triage a few candidates with targeted greps before committing a build to it.
  **S6201 is the exception** — oldcore S6201 has been ~98% clean fodder.
- **oldcore is NOT the build-ROI blocker it was thought to be**, though: a 5-module platform reactor
  including oldcore, feed-api, filter-stream-xar, search-solr-api and tool-provision-plugin ran 7:25
  with tests (206 suites, green) on a cold `~/.m2`. Don't drop oldcore *sites* purely for "its test
  suite is huge" — only on ROI when the whole reactor would exist for 1-2 fixes.
- **Leaf-module residue is now ALSO drop-heavy** for the small clean rules. When the allowlist total is
  small, expect a low clean yield even from leaf modules.
- `xwiki-platform-legacy-*` modules are a reliable untouched fallback (cleanup waves skip them; add
  `-Plegacy`, they build fast).

## Per-rule last-seen state

| Rule | Last-seen pool & clustering | Observed drop rate |
|---|---|---|
| **S6201** instanceof | **Drained**: the last 54 (the whole `xwiki-commons-extension` tree) went in one PR. Commons keeps only the 4 permanently-dropped crypto-cipher sites; rendering 0; platform 2 (both dropped). Concurrent sessions routinely leave 9-10 feature-module PRs open at once. | ~95-100%: **54/54** in the extension tree, **0/53** on an earlier 5-module commons batch, **42/42** across 26 non-extension files. Drops are coverage or flow-scope, not correctness. The `equals()`-with-explicit-local shape is ~a third of any pool and is a pure delete-the-declaration edit. |
| **S6068** Mockito `eq()` | **Drained in all three repos** (platform 102 → 35 → 17 → 0, over three passes; the last 17 were thin-spread one-to-three per module across 11 modules). Commons 4 done, rendering 0. | **0 drops (106/106).** Fully scriptable and test-only, so the module tests are the whole verification. Key the edit to the whole STATEMENT (scan up/down to the `;`), never the flagged argument. |
| **S9016** nested mock creation | Platform 101 → 49 → **4** (one 45-site PR across 26 modules; the pool was thin-spread — 1-4 per module — and a 26-module platform reactor still builds in 18 min). Commons 6 (1 extension-api + 5 store-blob-s3) still unswept. Rendering 0. | **~8%: only the type-inferred `mock()` form drops.** 39 of the 45 were scripted off `textRange`; 6 of the 7 `mock()` sites were still recoverable by hand (the stubbed getter's return type was already imported). The one real drop needed a *new foreign* import (Hibernate `Configuration`) — that is the drop line for this rule. |
| **S8714** try/catch/fail→`assertThrows` | **Drained in all three repos.** The platform 49 went in one PR (security-authorization-api 11, rest-server 8, export-pdf-default 7, model-api 7, oldcore 5, export-pdf-api 4, livedata-livetable 2, + 5 singletons = 21 files / 12 modules). Regenerates slowly. | **0 drops (62/62 across the three repos).** Three shapes, all safe: single-call try (~90%), `catch { fail() }` → `assertDoesNotThrow`, and a multi-statement try where only the last call throws (hoist the setup out, effectively-final locals still work in the lambda). |
| **S4201** null check before `instanceof` | Commons 4 (crypto-pkix 2, extension-api 2) — drained. Not yet counted in platform/rendering. | **0 drops (4/4).** Pure one-line delete of `x == null \|\|`; every site was the `equals()`/validate shape. |
| **S1192 / S1121 / S1153 / S1659 / S6353 / S1450 / S2133 / S1144** commons long tail | One-to-three each; 19 issues over 11 rules in one PR across 16 files / 14 modules (whole-repo build). | ~30% drops, all for a *reason visible in the site*: a duplicated literal inside a vocabulary list, an "unused" private method asserted by name in a reflection test, a `throws` on a `src/main` method. |
| **S5786** JUnit5 visibility | Commons 5 → 3 fixed. Class-level flags expand to several method strips per file. | **40% drops.** Two hard blockers: a test class read from ANOTHER package (constants like `X.HEADER`), and a `@BeforeEach @Override` of a public parent method — reducing an override's visibility does not compile. Check both before committing to a file. |
| **S3358 / S1130** rendering leftovers | Rendering's last mechanical fodder: 1 + 2 = 3 (plus one S1185) in one PR. | 1 drop of 3 on `S1130`: a `src/main` method other modules call. Test methods are safe. |
| **S4719** charset name → `StandardCharsets` | Commons 9 — **drained** (filter-test 5, crypto-common 2, component-default 1, legacy-classloader-api 1). Not yet counted in platform/rendering. | **0 drops (9/9).** Two shapes: a `private static final String X = "UTF-8"` used only as a charset (retype the constant to `Charset` — one-line diff, no call-site change), and a literal/foreign-API call site (`IOUtils.toString(byte[], String)` → `new String(bytes, UTF_8)`, which then orphans the constant and the `IOUtils` import). |
| **S1905** unnecessary cast | Commons 3 — drained. | 0 drops (3/3). Includes the `(List<String>) null` in a conditional (the other branch already fixes the poly-expression type) and a `(G)` cast on a raw return type. |
| **S2147 / S1596 / S1481 / S1264 / S1126** commons singletons | One each; all done. `S1450` 1, `S1121` 1, `S1192` 5, `S1144` 4, `S3012` 2, `S5786` 5 remain. | 0 drops. Ideal riders on a batch you are already building. |
| **S1450 / S1121 / S2589** rendering one-offs | The last mechanical rendering fodder: 1 + 1 + 2 = 4 in one PR (ParametersPrinter field→local, TagStack `put(k, v = new …)`, and two provably-negated conditions). | 1 drop of 3 on `S2589`: a `queueItem != null` guard in a concurrent-queue poll loop is dead but reads as deliberate defensiveness — leave those. |
| **S2209 / S3012 / S3024 / S1264** rendering one-offs | Rendering's only remaining mechanical fodder once the allowlist is dry: 4 + 2 + 2 + 1 = 9 in one PR. `S2209` was 4 `this.PARAMETERS_PRINTER` reads of a `protected static` parent field in syntax-xwiki21. | 0 drops (9/9). |
| **S6397** regex `[x]`→`x` | Swept: platform 9 (8 of them one transliteration table in `XWiki.java`) + rendering 8 (the same table duplicated in `XWikiSerializer2`). Comment-free one-token edit, on par with S7476 for safety. | **0 drops (16/16).** |
| **S6485** `new HashMap<>(n)` | Swept in oldcore (9), rendering (3), then **all 20 remaining platform sites** (13 modules — a wide but cheap reactor) and **10 commons sites**. Commons keeps ~3 in the extension tree. Good *rider* on a reactor you already build. | **0 drops (42/42).** Raw (`new HashSet(n)`) and explicitly-parameterised (`new HashSet<String>(4)`) constructors convert too — target typing infers the factory's type argument. |
| **S6208** comma-separated case labels | Rendering 4 (`TagStack` 3 — one group has 32 labels, `BlockNavigator` 1), commons ~10, platform 0. Pure syntax; the only care is wrapping a long label list and leaving an intervening `// fallthrough` group alone. | 0 drops (4/4). |
| **S6204/S6211** `.toList()` | ~100 open, rarely PR-touched, thin-spread; the 3 livedata modules once held 25; commons had 11 non-extension. Go-to pivot when the small families are all drained. | ~10% (10/11 in commons). Near-0 in tests; the drops are escape analysis — a **public/deprecated getter** returning the list. Private locals passed to a constructor whose *sibling* argument is already `Collections.emptyList()` are safe. |
| **S7158** `isEmpty()` | **Drained in all three repos** by one multi-repo run (platform 37, commons 37, rendering 40 = 114). Regenerates. When it comes back: commons `xml`/`extension-api`/`repository-api`, rendering `wikimodel` whitespace filters + `syntax-xwiki20` printers, platform oldcore. | **0 drops in 114 sites.** Nothing about this rule disqualifies a site. |
| **S7476** `///` banner | 34 platform / 27 commons on first sweep. Comment-only, the safest rule there is. | 0 drops in 48 applied. |
| **S3706** `.stream().forEach()` | Part of the proven second-generation pivot trio with S7476 and S2130. | 0 drops (19/19 platform+commons). |
| **S2130** `valueOf`→`parseX` | Same trio. One sweep of the three cleared 48 platform + 28 commons. | 0 drops (11/11). |
| **S8924** Mockito static import | Test-only, fully scriptable. Platform 27 sites / 13 files / 9 modules; commons 15 sites in just 2 files. Best fallback when the classic allowlist is 100% dropped. | **0 drops (42/42).** |
| **S1124** modifier order | Usually ZERO open PRs (cleanup waves skip it). Dense in oldcore + many leaf modules. Unswept in commons/rendering while drained in platform; rendering ~24. Clusters many-per-file (a constants block gave 13 in one file). | 0 drops (24/24 in rendering, one ~20-line script). |
| **S1640** EnumMap | ~35+ project-wide, concentrated in notification / ratings / security modules and their **test** files (test-assertion maps are the safest sites). | Low; main-code maps need the null-key check. |
| **S1643** `+=`→StringBuilder | Small pool (~9), mostly oldcore. More surgery than its siblings. | Moderate — prepend and read-between-appends shapes. |
| **S1066** nested `if` | Density NOT reliably oldcore-concentrated (sometimes ~70 in oldcore = one batch; other runs thin-spread across ~16 modules). Structural, so cleanup waves never touch it, and it regenerates. Commons 25 → 16, rendering 2 → 1. | ~25% overall but **0/10 in commons+rendering** — the single-line `//` comment between the two `if`s is recoverable, and `} else if (A) { if (B) }` with no trailing `else` merges fine. |
| **S1118/S1144/S1185** dead code | Deep MAJOR pools, near-always zero open PRs. oldcore is the densest source and is fair game even with an open oldcore PR for a *different* rule. Outside oldcore the pool is 1-2 per repo. | **Frequently 100% drops.** One S1185 datapoint worth keeping: removing a super-only *public* override from a public class (`XWikiSerializer2.onNewLine`, wikimodel) passed `revapi:check` — Revapi does not treat a method that moves to the superclass as removed. So "public API" alone is not a reason to drop S1185; a comment on the override still is. The S1185 pool is dominated by `com.xpn.xwiki.plugin.*` classes; S1118 by instantiated factories, abstract bases and public nested holders. Triage the class kind before committing a build. |
| **S1068/S1481/S1854** unused | ~100+ open; sometimes ~45 in oldcore alone, else thin-spread. | ~10-15% residue. |
| **S1192** duplicated literal | Fluctuates, sometimes under 20 project-wide — then a 20-50 batch is impossible; mix simplification rules instead. oldcore often holds 20+ alone. | Moderate (stale / substring / coincidental-semantics). |
| **S2093** try-with-resources | Low-yield in XWiki. | **Near-100% drops** — the `finally` is a state restore, not a close. |
| **S6126** text block | ~90 open in platform, rarely PR-touched. **Dominated by TEST files** (expected-output strings in rendering/macro `*Test.java`); dense test modules seen: `rendering-macro-include` ~15, `display-macro` ~8, `rendering-wikimacro-store` ~8. Rendering's 13 are all in `xwiki-rendering-wikimodel` parser tests. | **~30-40% in platform; 100% in the rendering `wikimodel` parser tests** (13/13 dropped — wiki-table/heading fixtures carry trailing spaces on nearly every content line, the XHTML one uses `\r\n`, and the `ListBuilderTest` indent ladder has no line at the baseline so the minimum-indent strip eats one space). Treat the whole rendering S6126 pool as closed. |
| **S3878** varargs array | Spread across oldcore + legacy; commons `logging-*` holds ~30. | Commons `logging-*` is **100% drops** (deliberate disambiguation). |
| **S5785** assertEquals | Rendering has held ~30. | High in `equals()` contract tests — suppress those instead. |
| **S5786** JUnit5 visibility | Deep once syntax/simplification/unused are drained. Open PRs usually sit on sibling modules. | Low. |
| **S5361** `replaceAll`→`replace` | Rendering 8, all in the `XWikiSerializer2` transliteration table (the S6397 twin — same table, different rule). | 0 drops (8/8). Watch the ONE escaped-metacharacter site: `replaceAll("\\\\.", "")` must become `replace(".", "")`, **not** `replace("\\\\.", "")`. |
| **S1611 / S1161 / S1125 / S1612 / S1488** small mechanical | Individually 1-5 per repo, so only worth doing as *riders* on a batch you are already building. Commons yielded 5+3+2+2+1 = 13 in one pass alongside S1124/S6208/S1640 (1+2+2 more). | **0 drops (18/18).** |
| **S1604** anon class→lambda | ~20 project-wide, one-or-two per file across many modules. | Low, EXCEPT `AccessController.doPrivileged(new PrivilegedAction<T>(){…}, acc)` — **permanent drop**, the lambda is an ambiguous reference (see `dropped-issues.md`). |
| **S3024** concat inside `append` | Swept: platform 16 (oldcore 13, of which `XWikiGroupServiceImpl` alone held 10 — a query-builder class is the reliable shape; + repository-server-api, skin-skinx, zipexplorer). | **0 drops (16/16).** Chained `append` preserves left-to-right evaluation, so a `count++` fragment is safe. Use a `char` literal for one-character fragments. |
| **S4719** charset name → `StandardCharsets` | Swept: commons 9, then **platform 15** across 12 modules (oldcore 3, skin-skinx 2, + singletons). | **0 drops (24/24)** but **3 of 15 needed dead-`catch` removal** (see learnings): the shape to watch is a flagged call inside `try { … } catch (UnsupportedEncodingException)`. Retyping a *private* `String` charset constant to `Charset` clears several keys in one line. |
| **S6353** `[0-9]` → `\d` | Swept: platform 8 (oldcore 4, chart-macro 2, url-scheme-standard 2). Two keys per line is common. | **0 drops (8/8).** Equivalent without `UNICODE_CHARACTER_CLASS`, which XWiki never sets. |
| **S1659** one declaration per line | Swept: platform 8, ALL in oldcore (`XWiki.java` 5, DBListClass 2, StaticListClass 1). | **0 drops (8/8).** Pure line-splitting; the densest sites are `String a = "", b = "", …` preamble blocks in long legacy methods. |
| **S2133** object built only for `getClass()` | Swept: platform 3, all one oldcore test (`CommentEventGeneratorListenerTest`). | **0 drops (3/3).** `any(x.getClass())` → `any(X.class)`; the local and its `Event` import go too. |
| **S1130** unthrowable `throws` | 294 → 155 → 81 → then **24 more the very next scan** (5 files / 5 modules, all files the previous PR had already touched). **This rule regenerates inside the files you just fixed** — see the cascade note below. Regenerated again to 66 and **8 more shipped** (all `@Test` methods of one file, `MessageStreamTest` in `legacy-messagestream-api`); platform now keeps **54, all `src/main`** (permanent). The regeneration is one FILE at a time — re-query it every run, it is nearly free. Commons 28 = 24 test (done) + 4 `src/main`; rendering 1 (`src/main`). | **0 drops on annotated test methods AND on `private` helpers (284/284 across two repos)**; `src/main` is a permanent drop, and a non-annotated **non-private** method inside a test class is a drop too (a `test-jar` consumer in another module can override it). The compiler is the whole verification. |
| **S1128** unused import | Platform 9 — swept (4 test files, 3 oldcore `src/main`, 2 others). Thin everywhere; a perfect rider on any reactor you already build. | **0 drops (9/9).** Check the simple name with a word boundary across the whole file first: the remaining hits are almost always prose in comments/strings, but a `{@link X}` in Javadoc counts as a use. |
| **S1117** local hides a field | **Was rejected as a rule; it is NOT — see the scope proof below.** Commons 15 (5 test files / 5 modules) swept in one PR. Platform's **107 are now swept too**, in two PRs split on `src/test` (58, 26 files / 19 modules — oldcore held 34 of them, `XWikiDocumentMockitoTest` alone 19) vs `src/main` (49, 14 files / 5 modules — oldcore 44, `XWikiDocument` 21). Both halves fit one 22-module reactor. | **0 drops (122/122 across two repos).** The local shadows the field from its declaration to the enclosing block's closing brace, so *every* bare occurrence in that range is provably the local. Rename only inside that computed range. `src/main` is NOT a drop condition here — a local cannot escape its method, so nothing is API-bearing, and **all three PRs of this sweep merged within two hours with no review comment**, `src/main` half included. Keep splitting (it lets the test half land first) but do not drop production-code sites for fear of the review. |
| **S116** field naming | Commons 2 (one abstract test class). **Platform 17 — swept: 16 fixed, 1 dropped** (oldcore held 10 of them: `XWiki` 5 lock/context fields, `XWikiContext` 3, `XWikiAttachment`, `XWikiPluginManager`; plus `UsersCacheTest` 4 and two singletons). | **0 drops on `private` fields (16/16), 1 drop on the single `protected` one.** `private` is the whole test — the field cannot escape the compilation unit, so `src/main` is NOT a drop condition (oldcore's `XWikiContext`/`XWikiAttachment` fields renamed cleanly). Print every occurrence before editing: they are nearly all `this.X`, and the setter parameter usually already carries the new name (`this.engine_context = engineContext` → `this.engineContext = engineContext`). Grep the name repo-wide once to rule out a reflective/Hibernate-mapping use. Not the same rule as the denylisted **S115** (`static final` constants). |
| **S125** commented-out code | Commons 36, of which only **7** were real (the `dumpCert` debug leftovers in `BcX509CertificateGeneratorFactoryTest`). Platform/rendering not yet counted. Comment-only, so it is as safe as S7476 *once you have triaged the site*. | **~80% drops** — see `dropped-issues.md`: a `TODO`/`FIXME` naming a JIRA issue, or a block that documents what the code under it covers. Only a debug helper with no anchor is a genuine fix. |
| **S5778** `assertThrows` lambda | **Platform was the real pool: 53**, and 37 of them sat in `xwiki-platform-model-api`'s `*ReferenceTest` family alone (`new XReference(new EntityReference(…))` — one shape, ~70% of the pool); the rest are 1-3 per module over 11 modules. 51 shipped in one PR. Commons 4 → 1 left (a recorded drop), rendering 1 (dropped). | **0 real drops (51/51)**; the 2 not shipped were dropped only for a same-FILE open PR. The discriminator is unchanged — read the call you are about to hoist; if IT is the thrower, drop. In practice the hoisted expression is a *fixture* (a valid `EntityReference`, a `WordBlock`, a `DefaultParameterizedType`) and the thrower is the constructor under test, so the whole family converts. Block-bodied lambdas (`() -> { setup; call; }`) convert too and become expression lambdas — that is a bonus, not a drop. |
| **S2093** try-with-resources | Rendering 2 → 1 fixed. | **50%** here, and the discriminator is one look at the `finally`: a real `close()` converts, a state *restore* (`pop()`, `setX(previous)`, `release()`) never does. |
| **S3415** swap operands | — | **Default DROP.** |
| **S117** local-variable/parameter naming | **Platform 45 in just 2 modules** — 36 in oldcore `src/main` (BaseClass 9, XWiki 6, BooleanClass 5, XWikiHibernateStore 4, + 12 singletons over 9 files) and 9 in one notifications-filters test. Commons **2** (one extension-api test), rendering **0**. Swept. | **0 drops (47/47).** Splits cleanly into *locals* (28, pure mechanical) and *method parameters* (17, public legacy APIs → own PR). Not to be confused with the rejected **S1117** (shadowing rename). |
| **S3012** manual array copy | Platform 5 — **drained** (feed-api, model-api, oldcore, rest-server, webjars-api, one each). Commons 1 (a recorded drop). | **0 drops (5/5).** Four were whole-array → `Collections.addAll(list, array)`; one copied a prefix → `list.addAll(Arrays.asList(a).subList(0, n))`. `Collections`/`Arrays` is usually NOT yet imported. |
| **S2589** condition always true | Platform 21 → **7 fixed, 14 permanent drops**. The clean sites cluster in one file (`DefaultNotificationCacheManager` gave 4). | **~65% drops.** Clean = a sub-expression made redundant by the PRECEDING branch (`else if (count && !composite)` after `if (count && composite)` → `else if (count)`), a null check after an `instanceof` pattern binding, or a null check inside a block already guarded by `instanceof`. Drop every dead *defensive* null check and any numbered case-analysis block whose comments document the cases. |
| **S1871** duplicate branch | Platform 8 → **4 fixed, 4 blocked**. Thin-spread (notifications-filters, legacy-notifications-filters, oldcore, ratings). | **50%, and the discriminator is mechanical**: count the `&&`/`||` in the MERGED condition — Checkstyle `BooleanExpressionComplexity` caps it at 3. Two-branch merges (`!a \|\| b`, `(A && B) \|\| (C && D)`) pass; a four-branch `return false` chain came out at 10 and was rejected by the build. |
| **S3358** nested ternary | Platform 10 → **8 fixed** (the last one, `XWiki#getServerURL`, converted to a guard clause), 1 not hoistable, 1 Checkstyle-blocked. Commons 3 (an open PR). | **~30% drops.** Extraction is behaviour-preserving when the outer condition becomes a guard clause. Two drop shapes: the ternary is an argument of a `this(…)` delegating constructor (nothing can precede it), and the enclosing method is at the `ExecutableStatementCount` cap (30) so hoisting adds a statement too many. |
| **S125** commented-out code (platform) | **Fully triaged now**: after the earlier 9 fixed / 12 dropped, the remaining 28 fresh keys went **11 fixed / 17 dropped** — every drop is recorded by key. Regenerates slowly. | **`src/main` is nearly all anchored; TEST files are the clean subset.** Every `src/main` hit was introduced by a `TODO`/`FIXME` or by a comment saying why the code is commented out. The clean shape is a test-file leftover with no anchor — disabled `// verify(…)` pairs (5 in one file) and `//Type x = new Type(…)` declarations. Also expect the odd false positive (prose Sonar reads as code). |
| **S2629** invoke conditionally | Platform 27 — **effectively a drop pool** (scheduler-api 8, oldcore 4, tool-jetty-listener 4). Commons has an open PR on its only clean file. | **100% drops.** The "redundant `x.toString()`" shape this table once called clean is NOT — on a logging call the eager String is load-bearing (see `dropped-issues.md`). Everything else needs an `isXxxEnabled()` guard. Treat the whole rule as a drop pool. |
| **S3824** computeIfAbsent | Commons 8 → **5 fixed** (`InternalFilterTest` 3, `ResourceLoader`, `URLTool`). **Platform 35 → 27 fixed in one PR** (18 modules, thin-spread: oldcore 14 then 1-2 each; the whole reactor built in 15:49). Regenerates from ordinary lazy-init code. | **~25% drops, and every one is visible in the flagged block** — no whole-file read needed. Clean = the guarded block is exactly one `put` of a value that cannot be `null`. Four drop shapes: the mapping function can return `null` (`getEngineByName` — `computeIfAbsent` would stop caching the negative result), the block feeds a second collection/map as well, the mapping function calls another component from inside the map operation, and the flagged `get()` is a save/restore of a context value rather than a lazy init. `XWikiContext` extends `Hashtable`, so `computeIfAbsent`/`putIfAbsent` work on it directly (with a cast on the result). |
| **S3457** two distinct shapes | Commons 8 → 1 fixed, 2 permanent drops, 5 blocked by an open PR. Platform 41 untriaged. | **Read the message, not just the rule — and NEITHER shape is a free edit.** For "No need to call `toString()`" on a LOG call write `String.valueOf(x)`; a bare deletion changes what the XStream-serialized job log stores and has now cost two commons PRs. "`%n` should be used in place of `\n`" is a behaviour change — always a drop when the produced string is asserted or compared. |
| **S4973** compare with equals | Platform 11, in 4 files. | **100% drops — a real-bug rule.** Each site needs a semantic decision about whether `==` was meant as identity (sentinel constants, interned type strings, boxed getters). JIRA issues, not a sweep. |
| **S3252** static member via a derived type | **Was on the OKF denylist and should not have been** (see below). 175 project-wide: platform 135, rendering 34, commons 6 — **123 fixed in one three-repo sweep** (platform 85 / 25 files / 14 modules, rendering 32 / 3 files / 2 modules, commons 6 / 5 files / 2 modules). What is left is the 52-site `org.xwiki.text.StringUtils` shape (a permanent drop). CRITICAL severity, zero open PRs, and it regenerates from ordinary refactoring. **Outcome: all three PRs merged within ~35 minutes with no review comment**, `src/main` and oldcore included — the denylist entry, not the fix, was the problem. | **0 drops on the 123 qualifier sites.** The pool splits on one cheap test — the token *before* the flagged member: if it differs from the declaring class's simple name it is a pure qualifier swap (mechanical); if it is the SAME simple name the "fix" is an import swap to the base class, which in XWiki means abandoning `org.xwiki.text.StringUtils` / `org.xwiki.localization.LocaleUtils` — always a drop. |
| **S2129 / S1158** needless wrapper instantiation | Platform 5 + 3, thin-spread in oldcore (`Package` holds one of each on the SAME two lines) plus lesscss-default. Absent from commons/rendering. | **0 drops (8/8).** Three shapes: `new Boolean(x).toString()` → `Boolean.toString(x)` (clears both rules at once), `new Boolean(true)` → `Boolean.TRUE`, and `Integer.valueOf(a).compareTo(b)` → `Integer.compare(a, b)`. `new String(s)` in a `cloneResult(String)` override is safe too — `String` is immutable — but that one drew the sweep's only review question, and the answer was a COMMENT, not a revert (see learnings). |
| **S6395** unnecessarily grouped subpattern | Platform 4, all in ONE `private static final Pattern` in `DefaultWikiMacroRenderer` (2 keys per line). | **0 drops (4/4)** — `(?:["'])` → `["']`. Unlike the denylisted `S6035`, the pool is a `Pattern` constant, not a `public static final String`, so there is no Revapi `constantValueChanged` break. Check that before writing the rule off. |
| **S2388** ambiguous inner-class call | Platform 3, all three nested descriptors of one livedata class. | **0 drops (3/3)** provided the nested class does NOT itself declare the method — then `super.setId(id)` and the bare call resolve to the same inherited method. Grep the nested class for the method name before batching; if it overrides it, `super.` changes the target. |
| **S3400** method returning only a constant | Platform 3 (all oldcore, all `private` with one caller). | **0 fixable — REJECTED by review.** The two shipped sites were closed unmerged (PR #6179, no comment) while the mechanical sibling PR merged. Trading a documented method for a constant is unwanted in this codebase; the same verdict took `S4165` with it. Both are permanent drops — see `dropped-issues.md`. |
| **S6876 / S6877** manual reverse iteration | Platform 2 + 2 — **all fixed** (`ExtensionIndexJob`, `RootClassLoaderTranslationBundle`; `AbstractImportMojo`, `PackageMojo` in the packager plugin). Commons 1 (a drop), rendering 0. | **0 drops on the 4 platform sites.** Both rules are the same fix — a for-each over `List#reversed()` replacing either a backward `ListIterator` walk (S6876) or a `Collections.reverse()` copy-mutation (S6877). The single drop condition is a body that mutates through the iterator (`it.remove()`/`it.set()`). Watch the orphaned `java.util.ListIterator` import. |
| **S3655** `Optional#get()` without `isPresent()` | Platform 3 → 2 fixed, 1 dropped. | **~33%.** Clean when the fix is a one-liner: `isPresent()` + a second `get()` on the same expression → `map(pred).orElse(false)`, and `stream…reduce((a,b) -> a + sep + b).get()` → `collect(Collectors.joining(sep))`. Drop when presence is established by an earlier statement Sonar cannot follow (a boolean computed from `map(…).orElse(false)`). |
| **S4030** collection populated but never read | Platform 2 — both fixed (`DefaultIOService`, `ColumnsDashboardRenderer`). | **0 drops (2/2)**, and the check is one grep: if every occurrence of the local is its declaration plus `add(…)` calls, delete both. Remove the enclosing `else` when the `add` was its only statement, or the fix trades S4030 for S108. Removes covered instructions, so it is a (small) JaCoCo-ratio risk. |
| **S2447** null returned from a `Boolean` method | Platform 12, commons 6 — **all dropped, permanently.** | **100% drops.** The `null` is the API contract ("no answer" ≠ `false`) on script services and bridges. Keys recorded in `dropped-issues.md`. |


**After the JavaScript sweep (platform 43 JS + 6 Java, commons 0, rendering 0):**

- **The JAVA mechanical pool in platform is now genuinely empty.** The ~85-rule allowlist returns 270
  keys of which only 59 are not already in `dropped-issues.md`, and 54 of those are the permanent
  `S1130` `src/main` residue (recorded by shape, so they keep reading as "fresh" — cross-check the
  path, not just the key). Everything the distribution facet holds beyond the allowlist is a rename
  (`S100` 24, all public API), a design change (`S110` 39, `S107` 27, `S4348`, `S3078`, `S6912`) or a
  real-bug rule (`S1872` residue, `S6019`). Total Java yield this run: **6**.
- **`javascript:` is the pool now, and it is real: ~880 open in platform, zero in commons/rendering.**
  Almost all of it sits in `xwiki-platform-web-war/src/main/webapp/resources/`. First sweep took
  `S7773` (38 of 60) + `S6353` (5 of 5) = **43 in one PR, 0 drops beyond the vendored files**. What is
  left, roughly in descending safety: `S6397` 7, `S1125` 8, `S1121` 25, `S2814` 32, `S3504` 14 (all
  untriaged); then the judgement-heavy bulk — `S7765` 90 (`indexOf`→`includes`, needs the receiver
  type per site), `S6582` 76 (optional chaining — differs on falsy-but-not-nullish), `S1848` 76
  (**false positive on Prototype**: `new Ajax.Request(…)` IS the side effect), `S7721` 61,
  `S7773` residue, `S4138` 55 (`for`→`for-of`, breaks on mutation/sparse arrays), `S7741` 37
  (`typeof x === 'undefined'` → ReferenceError on an undeclared variable), `S7740` 33, `S7761` 26.
- The `xml:S1135` (TODO) pools — platform 54, commons 14, rendering 6 — are TODO comments, i.e. the
  same non-starter as `java:S1135`.

## Candidates not yet swept

- **Commons rules triaged and REJECTED in one pass** (read the message, not the code): `S1319`
  return-an-interface, `S2326` unused type parameter, `S1150` implement Iterator not Enumeration,
  `S1210` override equals for compareTo, `S4968`, `S4929`, `S2157` — all public-API shape changes;
  `S3077`, `S2274`, `S2445`, `S2442` — concurrency semantics; `S2112` URL→URI, `S8688` `now()` time
  zone, `S4517` signed byte, `S2885` static→instance `SimpleDateFormat`, `S1163` throw-in-finally,
  `S6876` manual reverse iteration — behaviour changes; `S1075` hardcoded URI, `S3398` move method
  into inner class, `S5413`, `S119`, `S131` add a default case — design/refactor calls; `S3011`
  `setAccessible` (that IS the test framework's job).

- **A *naming* rule outlives every transform rule, and it is where the volume now is.** `S117` paid
  after the test-code generation dried up; `S1117` then paid 107 in platform in a single run, split
  into two PRs. The reason is structural: cleanup waves reformat and simplify but never rename, so a
  naming rule accumulates untouched. When the facet looks dead, sort it for rules whose message starts
  with "Rename" before concluding the repo is closed. Platform's remaining naming pool is **S116**
  (17, fields — cross-module, so harder).
- **`S1117`** "rename this local variable which hides the field declared at line N" — **the earlier
  rejection was wrong and is withdrawn.** The hazard is real (a missed occurrence re-binds to the field
  and still compiles) but it is fully removed by scoping: Java shadows the field from the local's
  declaration to the enclosing block's closing brace, so every bare occurrence in that computed range
  *is* the local and nothing outside it is. Compute the range by brace-matching forward from the
  declaration and substitute `(?<![\w$.])name(?![\w$])` only there — the `.` in the look-behind is what
  spares `this.name` and `x.name`. Three cheap assertions make it safe to batch: the flagged line really
  is a declaration, the NEW name occurs zero times in the range beforehand, and the name never appears
  inside a string literal in the range. Commons' 15 went in one PR with 0 drops, including 8 sites where
  the local and the field have the **same type** (so the compiler would not have caught a partial
  rename). Platform's 107 are the obvious next pool.
- **`S3252`** "use static access with X for Y" (rendering 34, 30 of them in two
  `xwiki20/reference/*TypeSerializer` classes) — **rejected, and the OKF denylist entry is right for
  this pool**: the code says `XWiki20LinkReferenceParser.ESCAPE_CHAR` where the constant is declared
  on the parent `GenericLinkReferenceParser`. Renaming the qualifier is behaviour-neutral but *loses
  the intent* (which parser's escape char this is) and churns imports — a style argument a reviewer
  would push back on, not a cleanup.
- **`S2925`** `Thread.sleep()` in a test (rendering 3, all `LinkCheckerTransformationTest`) —
  **rejected**: replacing the sleeps means introducing Awaitility and rewriting the test's timing
  contract, well past the mechanical bar.
- **`S3457`** `\n` → `%n` in a `String.format` — **read the whole message first**: rendering's site is
  one `format` call of a multi-part exception message whose sibling `format` keeps `\n`, so "fixing" the
  flagged one produces inconsistent output rather than platform-correct output.
- **`S8924`** Mockito static import, **`S5786`** JUnit5 visibility — the remaining unswept test-code
  rules. Commons `S5786` is 5 singletons in 5 different modules (a bad reactor-to-fix ratio unless
  each file expands to many removals).
- **`S6878`** "use the record pattern instead of this pattern match variable" — **rejected**:
  deconstructing a record into its components inside an `equals()` collides with the record's own
  accessor names and makes the method harder to read. A style judgement, not a cleanup.
- **`S5976`** "replace these N tests with a single Parameterized one" — **rejected**: a test-design
  decision, well over the 15-minute mechanical bar.
- **`S4973`** "compare Strings/boxed types with equals()" — read the site first: in XWiki the flagged
  `==` is sometimes a deliberate **identity** check (e.g. "is this the very same content object we
  were given?") with a comment saying so. Drop those.
- **`S2094`** empty class — split verdict. Fair game when the class is empty, `abstract`, in an
  `internal` package and has **zero references** in the repo (grep to prove it). Drop when the Javadoc
  says the emptiness is required (`FootnoteMacroParameters`: "the rendering engine requires specifying
  a class for parameters").
- **`S6213`** "rename this method/variable to not match a restricted identifier" (`record`, `yield`,
  `var`) — **rejected**: renaming a method or field is an API change, and the pool sits on public
  `record(…)` methods (`*QuestionRecorder`, `ExtensionJobHistory`). Not a mechanical fix.
- **`S4144`** "implementation is identical to method X" — **rejected**: deduplicating two methods that
  legitimately mean different things is a design decision, not a cleanup.
- **`S6035`** "replace this alternation with a character class" (`"^(?:d|F)ownload:.*"` → `"^[dF]…"`)
  — **rejected when the regex is a `public static final String`**: the value of a compile-time
  constant changing is a Revapi `java.field.constantValueChanged` break even though the regex is
  equivalent. Only worth doing on a private/local pattern.
- **`S3824`** `Map.get()`+condition → `computeIfAbsent` — check the guarded block first; when it does
  more than the single `put` (the rendering `AbstractXHTMLImageTypeRenderer` site also touches
  `CLASS`), `computeIfAbsent` is not an equivalent rewrite.
- **`javabugs:S2259`** "Fix this access that will throw a NullPointerException" (platform 127,
  rendering 94, commons 16 — the last never-triaged big pool) — **rejected as a sweep**. 100%
  `src/main`, and rendering's 94 cluster in the chaining renderers (`XWikiSyntaxChainingRenderer` 18,
  `PlainTextChainingRenderer` 9, `XHTMLChainingRenderer` 8) where the "null" comes from the chaining
  indirection Sonar's dataflow cannot follow. Each site needs a real per-site dataflow argument and a
  behaviour change; that is a JIRA issue, not a Sonar sweep.
- **`java:S899` + `java:S4042`** ignored `File.delete()` result / use `Files#delete` — the two rules
  sit on the SAME lines (commons: 5 shared sites), so one edit would clear ~10 issues. **Rejected
  anyway**: every commons site is `tempFile.delete()` in a `finally`, and `Files.delete` throws, which
  both masks the original exception and creates a fresh `S1163` (throw in `finally`). The `offer()`
  variant of S899 (unbounded queue, always `true`) is a design change too.
- **`java:S1948`** "make this non-static field transient or serializable" (commons 14) — **rejected**:
  the inverse of the denylisted `S2065`, and equally load-bearing — adding `transient` changes what
  gets persisted.
- **`java:S1163`** throw in `finally` (commons 1, `ExecutionContextRunnable`) — **rejected**: the
  `finally` deliberately rethrows a cleanup failure as a `RuntimeException`; fixing it means deciding
  to swallow it.
- **`java:S2386`** "make this member protected" (commons 8, rendering 3) — **rejected**: reduces the
  visibility of a public static member → Revapi break, same shape as the denylisted `S5993`.
- **`java:S117`** was the pool that paid after the test-code generation dried up, and it shows where
  to look next: **a *naming* rule can hide a dense mechanical pool long after the transform rules are
  spent**, because cleanup waves never rename anything. The same shape is still unswept elsewhere —
  platform `S116` (17, fields — API-bearing, harder) and `S1117` (107, the rejected shadowing rename).
- **A rule can regenerate inside the files your last PR touched.** Platform's S1130 came back with 24
  issues in exactly the 5 files the previous run had fixed — the cascade described under *Batch mode*
  in `learnings.md`. So after shipping a signature-narrowing batch, re-query the same rule next run
  before assuming it is drained.
- **Platform rules triaged from their MESSAGE and rejected** (one `ps=2` query each, no source read —
  do not re-triage): `S2177` rename-shadowing-a-private-parent-method, `S5738` don't-use-a-
  deprecated-for-removal API, `S1206` add `hashCode()`, `S2139` log-or-rethrow, `S2176` rename this
  class, `S6880` if/else → switch expression, `S1452` remove a generic wildcard, `S2142` re-interrupt,
  `S1181` catch `Exception` not `Throwable`, `S8786` simplify a super-linear regex, `S135` reduce
  break/continue count. All are renames, behaviour changes or design work.
- **Platform candidates read but NOT swept this run** (messages only, no source read — a next run
  should start here rather than re-pulling the facet): `S108` empty block (84 — each site needs a
  fill/remove/comment judgement, so not a batch), `S3358` nested ternary (10, extract-to-statement —
  viable), `S6126` text block (39, ~30-40% drops), `S2093` try-with-resources (11, never triaged in
  platform), `S1871` duplicate branch (8), `S1185` useless override (14), `S1118` (7).
  `javabugs:S2190` (9, non-terminating recursion) and `javabugs:S6416` (6) are real-bug rules needing
  per-site dataflow — JIRA issues, not a sweep.
- **The `javascript:` rules are the one genuinely unswept generation left in platform** (`S7765` 90,
  `S1848` 76, `S6582` 76, `S7721` 61, `S7773` 60, `S4138` 60 — ~450 issues) and no run has triaged
  them. Verifying them means the pnpm/Nx build rather than Maven, so budget for that before starting.

## Where the classic allowlist stands

**Latest sweep (platform 35, commons 15, rendering 0).** The 56-rule mechanical allowlist queried in
ONE call per repo returns platform 60 / commons 18 / rendering 6 — and cross-checking those 84 keys
against `dropped-issues.md` left only **17 in platform and 0 in the siblings**. That cross-check is
the single cheapest step in the run: do it on the whole shortlist before opening any source file.
What paid instead was (a) `S1130` regenerating in the files the last PR touched, and (b) `S1117`,
which the index had wrongly written off. **The lesson generalises: when the allowlist reads zero,
the next pool is either a rule that regenerates or a rule a past run rejected too quickly** — re-read
the rejection's reasoning before trusting it.


The 39-rule classic allowlist is genuinely exhausted in all three repos (commons ~15 residue,
rendering 1, platform 18 and nearly all already dropped), and so is the test-code generation
(`S9016`/`S6068`/`S8714`). **A future run must open with the DISTRIBUTION FACET, not the allowlist.**

**After this sweep** (platform 77, commons 24, rendering 1):

- **Platform's long tail is spent** for S6068/S3024/S4719/S6353/S1659/S1905/S4201/S1596/S2133.
  Deliberately **deferred but still viable** there — a good next batch, not drops: `S3012` 5
  (manual array copy — needs a per-site whole-array-vs-suffix check), `S1450` 4 (field → local; the
  sites are `Thread` fields in `DefaultSolrIndexer` and `jodConverter` in `DefaultOfficeServer`, a
  real refactor), `S1121` 2 (`XWikiRepositoryModel` assignment-in-expression).
- **`java:S1130` in platform was that pool and it paid twice**: 155 in one 12-module PR, then the
  remaining **81 in one 49-module PR**, 0 drops either time. It is now spent (58 permanent drops left).
  The recipe that made it cheap: one `&rules=java:S1130&ps=500` query, split on `/src/test/`, then
  classify each site by the annotation above the flagged line **and by `private`** — no snippet reads
  at all, and the whole 85-site classification plus edit script fits in one turn.
- Commons and rendering are otherwise down to the rejected rules; expect a handful at best.
- `javabugs:S2259` has now been triaged and rejected (see above). The only untriaged generation left
  anywhere is platform's **`javascript:` rules** (~450).
- **Commons' last mechanical fodder was `S2629` + `S3358` in `extension-api`** (4 + 3, one module
  build, 3:13 / 223 tests). After that its whole 82-rule facet is denylisted, rejected or a recorded
  drop. **Rendering yielded literally zero** on the same pass — its 47-rule facet contains no rule
  that is not denylisted, rejected or already a dropped key. Do not budget a rendering PR at all
  until something regenerates; "check rendering" is now a one-facet-query answer.
- **Confirmed again: platform still regenerates a real pool, the siblings do not.** A later sweep
  found `S5778` at 53 in platform while commons and rendering yielded **literally zero** — every
  candidate key in both siblings was already in `dropped-issues.md`. The whole sibling check is now a
  two-call answer (one distribution facet + one `grep` of the candidate keys against the drop index);
  do not budget a commons or rendering PR up front, and do not spend snippet reads there.
- **Rendering is closed — RE-CONFIRMED, and the check is now two calls.** Pull its 49-rule facet and
  cross-check every key against `dropped-issues.md`: a later sweep produced exactly THREE keys not
  already denylisted/rejected/dropped, and all three were non-starters (S1845, S5411 ×2, S5843).
  Budget a rendering PR at 0 and do not spend a single snippet read there.
- **Rendering was already closed once before.** The last two mechanical issues (S1066 1 + S5778 1) shipped. Everything
  remaining is a refactor, a rename or a behaviour change: S127 (13), S135 (3), S1452 (3), S2176 (3),
  S8786 (3, regex backtracking), S6880 (2, if→switch), S5961 (11, assertion counts), S1134 (4, FIXME
  comments), S2925, S3457, S4274, S1182, S1700, S3252 (34), plus `javabugs:S2259` (94, never
  triaged). Budget a rendering PR at 0-2 issues and do not hunt for volume there.
- **Commons is nearly closed too.** After the S125/S5778/S1130 pass its whole facet is 87 rules and
  the mechanical part of it is spent; the only deferred-not-dropped item is S3358 (3). What is left is
  S6355/S1123 (154 — needs the deprecating version), S1135/S1134 (TODO/FIXME), S5993, S3776, S112,
  S1133, S1186, S1168, S2065 — all denylisted or design work, plus `javabugs:S2259` (16).

**After the S1117/S116 sweep (platform 107, commons 2, rendering 0):**

- **Platform's S1117 is spent**; its next naming pool is `S116` (17 fields). `S1130` has regenerated to
  **66** (it was written off at 58 permanent drops — re-query it, the cascade note applies). Still
  deferred-not-dropped and never triaged in platform: `S3824` 35, `S3457` 41, `S125` ~25, `S6126` 36,
  `S3358`, `S1871`, `S1185`, `S1118`.
- **Commons is down to singletons.** Its 84-rule facet cross-checked against `dropped-issues.md` left
  exactly FIVE keys: 3 × `S1214` (OKF-denylisted) and the 2 × `S116` that shipped. Budget commons at
  0-2 and do not spend snippet reads beyond the cross-check.
- **Rendering: closed, third confirmation.** Its 50-rule facet cross-checked against the drop index
  left 13 keys, all in already-rejected rules (`S4144` 5, `S1214` 4, `S2160`, `S1141`) plus one new
  drop (`S2198` ×2, now recorded). The whole rendering answer is two calls; do not budget a PR.

**XWiki's source level is Java 21** (`<xwiki.java.version>21</xwiki.java.version>` in the commons root
pom), so a rule is never disqualified for needing a recent JDK — `S6876`/`S6877` (`List#reversed()`,
SequencedCollection) converted cleanly. Past runs wrote off `S6885` (`Math.clamp`) and `S6916`
(pattern-match guards) as "Java 21"; that reason was wrong and those two deserve a re-read. Confirm the
level with one grep before rejecting a modernization rule.

**After the S125/S116 + long-tail sweep (platform 36, commons 0, rendering 0):**

- The whole run's find phase was **two calls per repo**: one `issues/search` with a ~78-rule mechanical
  allowlist (`&ps=500`) and one grep of every returned key against `dropped-issues.md`. That returned
  platform **69 fresh / 201**, commons **8 / 59**, rendering **0 / 22** — and of the 8 commons keys all
  were drops. Do exactly this before budgeting anything.
- **Platform's fresh mechanical pool is now `S125` 17 (all recorded drops) + `S116` 1 + a long tail of
  2-3-issue rules.** What is still un-triaged there: `S3457` 41, `S6126` 36 (all keys already dropped),
  `S2589`/`S1185` 14 each, `S1118` 7 — plus the `javascript:` generation (~450, needs the pnpm/Nx build).
- **Commons and rendering are CLOSED, fifth confirmation.** The only fresh commons keys were `S2447` (6),
  `S6876` (1) and `S1192` (1), all dropped for reasons now recorded. Budget both at zero.

**After the S3824 + long-tail sweep (platform 54, commons 0, rendering 0):**

- **Platform's remaining un-swept mechanical-looking pools**: `S3457` 41 (two shapes, neither free),
  `S125` 28 (test files are the clean subset), `S6126` 36, `S116` 17 (field renames, cross-module),
  `S2589` 14, `S1185` 14, `S1118` 7. `S3824` regenerates.
- **Commons and rendering are CLOSED, fourth confirmation, and the check is now ONE call each**: pull
  every open Java issue key (`ps=500`, 3 pages) and cross-check against `dropped-issues.md` — commons'
  1155 and rendering's 430 all sit in rules that are OKF-denylisted or recorded as rejected here.
  Budget both at zero and do not spend a single snippet read there.
- **`java:S9149`** "Rename this method; it hides X in Y" (commons 45, platform 8, rendering 1) —
  **rejected**: the commons pool is `org.xwiki.text.StringUtils` re-declaring commons-lang static
  methods, i.e. the same shape as the permanently-dropped `S3252` `StringUtils` sites. A rename of a
  public static method is an API break.
- **Platform rules classified from their MESSAGE this run and rejected** (one `ps=1` query each, no
  source read): `S6885` (`Math.clamp`, Java 21), `S6916` (pattern-match guard, Java 21), `S2676`,
  `S2583`, `S4838`, `S2097` (real-bug rules needing a per-site decision), `S1165` (make a field
  final), `S2440`, `S3599`, `S1860`, `S5998`, `S1171`, `S3400` beyond the 3 above.

Rules confirmed **not** worth attempting (S2065, S5993, S5411, S1168, S1172, S6355/S1123, S2143/S2160/
S1141, …) are in the OKF denylist — check `okf/sonarqube/index.md` before adding one here.
