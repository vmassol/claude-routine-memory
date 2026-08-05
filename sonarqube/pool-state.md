# Pool state (VOLATILE — re-query before trusting any number here)

Where each rule's issues were last observed, how they cluster, and the drop rate actually seen. This
is the part of the old `rules/*.md` files that must NOT live in the OKF, because it is stale within
days. **The fix itself, and the drop conditions, are in the OKF** —
[`xwiki/okf/sonarqube/`](https://github.com/xwiki/xwiki-dev-llm/tree/master/xwiki/okf/sonarqube).

Every count below is a *last-seen* observation, not a fact. Confirm with
`issues/search?...&rules=java:SXXXX&ps=1` → `total` before planning a batch.

## The standing shape of the pools

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
| **S6068** Mockito `eq()` | Platform 102 → 35 → **17** after a second 18-site pass; the remainder is thin-spread one-or-two per module across ~11 modules (wiki-workspaces-migrator 3, legacy-events-hibernate-api 3, notifications-preferences-api 2, icon-default 2, …). Commons had 4, all in `job-default`/`DefaultJobExecutorTest` — now done. Zero in rendering. | **0 drops (89/89).** Fully scriptable and test-only, so the module tests are the whole verification. |
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
| **S3415** swap operands | — | **Default DROP.** |

## Candidates not yet swept

- **Commons rules triaged and REJECTED in one pass** (read the message, not the code): `S1319`
  return-an-interface, `S2326` unused type parameter, `S1150` implement Iterator not Enumeration,
  `S1210` override equals for compareTo, `S4968`, `S4929`, `S2157` — all public-API shape changes;
  `S3077`, `S2274`, `S2445`, `S2442` — concurrency semantics; `S2112` URL→URI, `S8688` `now()` time
  zone, `S4517` signed byte, `S2885` static→instance `SimpleDateFormat`, `S1163` throw-in-finally,
  `S6876` manual reverse iteration — behaviour changes; `S1075` hardcoded URI, `S3398` move method
  into inner class, `S5413`, `S119`, `S131` add a default case — design/refactor calls; `S3011`
  `setAccessible` (that IS the test framework's job).

- **`S1117`** "rename this local variable which hides the field declared at line N" (commons 15, mostly
  test files; platform 107) — **rejected**: renaming a shadowing local is the one rename where a *missed*
  occurrence still compiles, because it silently re-binds to the field of the same name. Only worth it
  with a guard that asserts zero bare occurrences of the old name between the declaration and the end of
  the method, and even then the payoff is one issue per rename.
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

## Where the classic allowlist stands

A 39-rule allowlist query now reads **commons 125 → ~15 residue, rendering 1 (a dropped key),
platform 18 (nearly all already dropped)**. The allowlist is genuinely exhausted: a future run must
open with the *distribution facet*, not the allowlist, and pivot to the test-code pools. After the
S9016/S6068/S8714 sweep the deepest remaining ones are **platform `S8714` 49 and `S9016` ~49**, both
still worth a full PR each; commons and rendering test-code pools are drained.

**After this sweep**: platform's `S8714` is gone, commons' long tail is down to the rejected rules
above, and rendering is at zero mechanical issues. The next run should open with the distribution
facet on **platform** (its long tail — `S4719` 15, `S3024` 16, `S6068` 17, `S6126` 39 — is the only
place left with volume) and expect commons/rendering to yield a handful at best.

**Rendering is closed.** Its last mechanical singletons (S1450, S1121, two S2589) are done; what
remains is S127 (assign to loop counter), S135 (multiple break/continue), S1452, S2176, S8786, S2925,
S3457, S4274, S1182, S1700 — all refactors, renames or behaviour changes, plus the standing denylist.
Expect a rendering PR to be 4 issues, not 50, and budget accordingly rather than hunting for volume
there. The one place volume may return is `javabugs:S2259` (94 open, never triaged).

Rules confirmed **not** worth attempting (S2065, S5993, S5411, S1168, S1172, S6355/S1123, S2143/S2160/
S1141, …) are in the OKF denylist — check `okf/sonarqube/index.md` before adding one here.
