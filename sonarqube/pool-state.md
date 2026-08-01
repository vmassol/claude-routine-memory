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
- **The next generation of pools is the TEST-CODE rules and the small "new" mechanical rules.**
  Platform holds `S9016` 101 (extract nested mock creation), `S6068` 102→35 (useless Mockito `eq()`),
  `S8714` 49 (assertThrows). Rendering has no volume at all — its only viable fodder is a handful of
  one-off mechanical rules (S2209/S3012/S3024/S1264 gave 9 in one PR).
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
| **S6068** Mockito `eq()` | Platform 102 → 35 after a 67-site sweep of the dense modules (security-authentication-default 15, wiki-creationjob 10, wiki-template-default 10, wiki-default 9, wiki-script 8, wiki-user-default 8, security-authorization-bridge 7). Zero in commons/rendering. The remainder is thin-spread across ~20 modules. | **0 drops (67/67, 120 `eq()` wrappers).** Fully scriptable and test-only, so the module tests are the whole verification. |
| **S9016** nested mock creation | Platform **101**, unswept: oldcore 15, notifications-notifiers-api 7, rest-server 7, user-default 6, mail-send-default 5, refactoring-default 5. Test-only. | Untried. |
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
| **S1118/S1144/S1185** dead code | Deep MAJOR pools, near-always zero open PRs. oldcore is the densest source and is fair game even with an open oldcore PR for a *different* rule. | **Frequently 100% drops.** The S1185 pool is dominated by `com.xpn.xwiki.plugin.*` classes; S1118 by instantiated factories, abstract bases and public nested holders. Triage the class kind before committing a build. |
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

- **`S8714`** try/catch/fail → `assertThrows` and **`S6068`** drop Mockito `eq()` — test-only and
  structural; safe-ish but more effort, own PR if attempted.
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
open with the *distribution facet*, not the allowlist, and pivot to the test-code pools
(`S9016` 101, `S6068` 35, `S8714` 49 — all platform) or classify a not-yet-swept rule.

Rules confirmed **not** worth attempting (S2065, S5993, S5411, S1168, S1172, S6355/S1123, S2143/S2160/
S1141, …) are in the OKF denylist — check `okf/sonarqube/index.md` before adding one here.
