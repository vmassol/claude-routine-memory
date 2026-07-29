# java:S8924 — use a static import for a Mockito method

> Load only when fixing S8924. Cross-cutting mechanics (assert-guarded script, line-length,
> orphaned imports, pool-shift) live in `learnings.md` → *General batch-fix techniques*.

Message: `Use a static import for "mock"` (also `when`, `verify`, `doReturn`, …). Test code only, so
**zero coverage risk** — the safest kind of batch after the comment-only rules, and a good platform
fallback when the classic allowlist there is 100% in `dropped-issues.md`. Datapoint: platform 27
sites / 13 files / 9 modules, **27 fixed / 0 drops**, one `-Plegacy,quality` reactor.

**Fully scriptable — no per-site reading.** Per flagged file:

1. Take the set of flagged method names from the issue `message` (regex `"([^"]+)"`), NOT from the
   line numbers — this is drift-proof. Group by file with a `Counter`.
2. Assert `content.count(f'Mockito.{name}(') == <issue count for that name in that file>` BEFORE
   writing anything. This caught every file exactly on a checkout ~1h ahead of the scan.
3. Replace `Mockito.<name>(` → `<name>(`. The trailing `(` matters: it stops `Mockito.mock(` from
   eating `Mockito.mockStatic(`.
4. Add `import static org.mockito.Mockito.<name>;` if absent. XWiki convention: static imports form
   ONE alphabetically-sorted block at the END of the import list, after a blank line — merge the new
   ones into the existing block and re-sort it; create the block after the last plain import when the
   file has none.
5. Drop `import org.mockito.Mockito;` **only if** `Mockito` no longer appears outside import lines
   (strip import lines first, then word-search) — a remaining `Mockito.verify` or a javadoc
   `{@link Mockito}` must keep it. In practice every file in the platform batch ended up dropping it.

Lines only get shorter, so the >120 check never fires. The rule pool is spread thin (1-8 per module),
so pick the modules by count and accept a ~9-module reactor; `oldcore` is often in it for a single
site, which is fine (~7 min).

**Not yet checked:** whether S8924 also fires for non-Mockito statics (e.g. `Assertions.assertEquals`).
The platform pool was 100% `org.mockito.Mockito`; re-read the `message`s before assuming.
