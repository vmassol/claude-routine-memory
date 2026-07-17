# java:S1068 / S1481 / S1854 — unused-code removal

> Load only when fixing one of these. Cross-cutting mechanics live in `learnings.md` → *General
> batch-fix techniques*. Deep MAJOR pool.

~100+ open; sometimes ~45 in oldcore alone, else thin-spread (then a wide leaf-module reactor, and you
can EXCLUDE oldcore when all its hits are DROPs). Bundle `S1161` (@Override) for a clean multi-type PR.
Needs light dataflow judgement → delegate reading to ONE Explore subagent returning a precise action
per issue KEY (DELETE range / STRIP_PREFIX / INSERT_OVERRIDE / DROP+reason); apply by line number in
one assert-guarded script. Expect ~40 of ~45 after drops.
- **S1481 + S1854 fire as a PAIR** on the same `Type x = expr;` line — one edit clears both.
- Pure RHS → delete the whole decl line. Side-effecting RHS → KEEP the call as a bare statement, drop
  only the `Type name = ` prefix (`indent + line.split(' = ',1)[1]`). Must-keep side effects:
  `doc.addAttachment`/`newXObject` (mutates the doc the asserts check), `registerMockComponent`, a
  lazily-initializing getter, `velocityManager.getVelocityContext()`.
- `x = null` dead store in an `else` whose `if` returns → delete the whole `} else { x = null; }`. A
  trailing dead `timer++` → drop just the `++`, keep the read.
- **Removing a private LOGGER/field usually ORPHANS its import** → delete it too (orphaned-import rule,
  General techniques).
- Clean up what removal leaves: a COMMENT that solely described the removed line (or a field's javadoc);
  STRAY BLANK LINES (double blanks, trailing before `}`, leading after `{`). ASSERT each deleted blank
  is truly blank (`line.strip()==''`) — a hint can mis-point at the next member's `/**` opener.
- **Removal CASCADES:** deleting `T x = other.getFoo()` can newly-orphan `other` or a sibling local/block
  that only fed it. Sonar flags only the outermost; trace each removed RHS's inputs and delete the whole
  dead chain (pure getters only; keep side-effecting calls).
- A write-only field/var assigned in ONE place IS fixable (delete decl + that assignment; `@Override`
  setter's now-unused param is not re-flagged — S1172 skips overrides). **DROP:** assigned in SEVERAL
  places; exposed via a public setter (API); a dead store whose call must MOVE. ~10-15% residue.
