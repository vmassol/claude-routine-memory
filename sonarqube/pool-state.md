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
| **S6201** instanceof | The deepest pool by far, and it barely regenerates down. Commons has held ~226 (crypto tree ~77, `extension-api` ~45, `filter-api` ~17, `logging-api` ~12, `job-api` ~10); rendering ~52 (`rendering-api` 24 — the `Block` hierarchy's `equals()`, `transformation-macro` 10, `xdomxml10` 7); platform oldcore ~90-140. Concurrent sessions routinely leave 9-10 feature-module PRs open at once. | ~95-100%. oldcore ~98%, dense feature modules ~90-93%. Drops are coverage, not correctness. |
| **S6204/S6211** `.toList()` | ~100 open, rarely PR-touched, thin-spread; the 3 livedata modules once held 25. Go-to pivot when the small families are all drained. | Near-0 in tests; real drops on escape analysis in oldcore/feature code. |
| **S7158** `isEmpty()` | **Drained in all three repos** by one multi-repo run (platform 37, commons 37, rendering 40 = 114). Regenerates. When it comes back: commons `xml`/`extension-api`/`repository-api`, rendering `wikimodel` whitespace filters + `syntax-xwiki20` printers, platform oldcore. | **0 drops in 114 sites.** Nothing about this rule disqualifies a site. |
| **S7476** `///` banner | 34 platform / 27 commons on first sweep. Comment-only, the safest rule there is. | 0 drops in 48 applied. |
| **S3706** `.stream().forEach()` | Part of the proven second-generation pivot trio with S7476 and S2130. | 0 drops (19/19 platform+commons). |
| **S2130** `valueOf`→`parseX` | Same trio. One sweep of the three cleared 48 platform + 28 commons. | 0 drops (11/11). |
| **S8924** Mockito static import | Test-only, fully scriptable. Platform 27 sites / 13 files / 9 modules. Best platform fallback when the classic allowlist there is 100% dropped. | **0 drops (27/27).** |
| **S1124** modifier order | Usually ZERO open PRs (cleanup waves skip it). Dense in oldcore + many leaf modules. Unswept in commons/rendering while drained in platform; rendering ~24. Clusters many-per-file (a constants block gave 13 in one file). | 0 drops (24/24 in rendering, one ~20-line script). |
| **S1640** EnumMap | ~35+ project-wide, concentrated in notification / ratings / security modules and their **test** files (test-assertion maps are the safest sites). | Low; main-code maps need the null-key check. |
| **S1604** anon class→lambda | ~20 project-wide, one-or-two per file across many modules. | Low. |
| **S1643** `+=`→StringBuilder | Small pool (~9), mostly oldcore. More surgery than its siblings. | Moderate — prepend and read-between-appends shapes. |
| **S1066** nested `if` | Density NOT reliably oldcore-concentrated (sometimes ~70 in oldcore = one batch; other runs thin-spread across ~16 modules). Structural, so cleanup waves never touch it, and it regenerates. | ~25%. |
| **S1118/S1144/S1185** dead code | Deep MAJOR pools, near-always zero open PRs. oldcore is the densest source and is fair game even with an open oldcore PR for a *different* rule. | **Frequently 100% drops.** The S1185 pool is dominated by `com.xpn.xwiki.plugin.*` classes; S1118 by instantiated factories, abstract bases and public nested holders. Triage the class kind before committing a build. |
| **S1068/S1481/S1854** unused | ~100+ open; sometimes ~45 in oldcore alone, else thin-spread. | ~10-15% residue. |
| **S1192** duplicated literal | Fluctuates, sometimes under 20 project-wide — then a 20-50 batch is impossible; mix simplification rules instead. oldcore often holds 20+ alone. | Moderate (stale / substring / coincidental-semantics). |
| **S2093** try-with-resources | Low-yield in XWiki. | **Near-100% drops** — the `finally` is a state restore, not a close. |
| **S6126** text block | ~90 open, rarely PR-touched. **Dominated by TEST files** (expected-output strings in rendering/macro `*Test.java`); dense test modules seen: `rendering-macro-include` ~15, `display-macro` ~8, `rendering-wikimacro-store` ~8. | **~30-40%** — a drained-pool run netted 25/41. |
| **S3878** varargs array | Spread across oldcore + legacy; commons `logging-*` holds ~30. | Commons `logging-*` is **100% drops** (deliberate disambiguation). |
| **S5785** assertEquals | Rendering has held ~30. | High in `equals()` contract tests — suppress those instead. |
| **S5786** JUnit5 visibility | Deep once syntax/simplification/unused are drained. Open PRs usually sit on sibling modules. | Low. |
| **S3415** swap operands | — | **Default DROP.** |

## Candidates not yet swept

- **`S6397`** redundant regex character class — ~9 platform / 8 rendering. Worth triaging.
- **`S6485`** `new HashMap<>(n)` → `HashMap.newHashMap(n)` — *safe and compiles*, but its ~29 platform
  sites are spread over 14 modules including solr-api → poor build ROI. Take it only as a rider on a
  reactor you are already building (oldcore holds 9).
- **`S8714`** try/catch/fail → `assertThrows` and **`S6068`** drop Mockito `eq()` — test-only and
  structural; safe-ish but more effort, own PR if attempted.

Rules confirmed **not** worth attempting (S2065, S5993, S5411, S1168, S1172, S6355/S1123, S2143/S2160/
S1141, …) are in the OKF denylist — check `okf/sonarqube/index.md` before adding one here.
