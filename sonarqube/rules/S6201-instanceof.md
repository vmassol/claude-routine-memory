# java:S6201 — instanceof pattern matching

> Load only when fixing S6201. Cross-cutting mechanics (assert-guarded script, subagent
> delegation + verification, line-length, orphaned imports, pool-shift) live in `learnings.md`
> → *General batch-fix techniques*.

The deepest clean pool; go-to when small pools are drained. By far the largest clean pool
(hundreds open, ~90-140 in oldcore alone) and it barely regenerates down — the reliable batch
source when the small mechanical rules are all simultaneously drained below 20 (the common
concurrent-session state). Java 16+ (xwiki-platform 18.x is Java 17 — compiles). Message:
"Replace this instanceof check and cast with 'instanceof Foo foo'". Fix = bind a pattern
variable and delete the redundant cast:
- Positive guard: `if (x instanceof Foo) { ((Foo) x).m(); }` → `if (x instanceof Foo foo) { foo.m(); }`
- Compound `&&` (pattern var scopes rightward + into block): `x instanceof Foo && ((Foo) x).m()` →
  `x instanceof Foo foo && foo.m()`
- Ternary/return: `return x instanceof Foo ? ((Foo) x).m() : y;` → `... instanceof Foo foo ? foo.m() : y;`
- Negated guard + early exit (flow scoping): `if (!(x instanceof Foo)) { return; } Foo foo = (Foo) x;`
  → `if (!(x instanceof Foo foo)) { return; }` (delete the redundant decl, use `foo`).
- Negated `||` short-circuit (VALID): `!(x instanceof Foo) || ((Foo) x).m()` → `!(x instanceof Foo
  foo) || foo.m()` — the `||` RHS runs only when the instanceof is TRUE, so `foo` is assigned there.
- Existing explicit local: `if (x instanceof Foo) { Foo foo = (Foo) x; ... }` → reuse that local's
  name in the pattern and delete its declaration line. `Object[]` patterns work (`x instanceof Object[] arr`).

**Module choice.** oldcore's ~90-140 make a single-module batch (`-pl xwiki-platform-oldcore
install`) — you can clear ALL of oldcore's S6201 in ONE PR: split the sites across ~6 PARALLEL
general-purpose subagents (~15 sites/agent) over DISJOINT files, one ~7.5-min build-with-tests, ~0
drops (oldcore S6201 is ~100% clean fodder). Don't cap at 50 — the whole module is feasible.
When oldcore is PR-claimed, the next-densest FEATURE module is a clean self-contained batch —
PREFER one concentrated in FEW submodules (cheaper reactor) over the same count spread wide, and
DROP Solr-based submodules (`-*-index`, `-solr-*` — slow) and feed-api (~5 min).
**Concurrent sessions routinely leave 9-10 S6201 feature-module PRs open at once** (incl. oldcore +
several wildcard "various modules" PRs) — often leaving only thin residue spread 1-2 per module across
26+ submodules; when that's all that's left, PIVOT (the S1118/S1144/S1185 family is the go-to).
**When even the big feature modules are ALL claimed, AGGREGATE untouched cheap leaf modules into one
reactor to reach 20-50.** Trust NO fixed list — the "obvious" aggregate sets get claimed too;
re-query by module each run. The `xwiki-platform-legacy-*` modules are a reliable untouched fallback
(cleanup waves skip them; add `-Plegacy`, they build fast). The aggregate-leaf pool is dominated by
near-100% 0-drop fodder: **event-listener `onEvent(Event event, ...)` guards**
(`if (event instanceof XEvent) { ...((XEvent)event)... }`), **exception-rethrow guards**
(`if (e instanceof XException) { throw (XException) e; }`), equals-style `if (!(o instanceof X))
return; X x=(X)o;`, servlet-filter `request/response instanceof HttpServlet*`, and internal
`*Reference`/`*Resolver`/`*Serializer` + AST-visitor converters (near-0-drop fodder).

**Mechanics & drops.** STRUCTURAL → delegate to PARALLEL general-purpose subagents (disjoint files,
~13 sites each), splitting BY module/submodule; verify full coverage after. A pattern var can be a
try-with-resources resource (effectively final). **Naming:** idiomatic camelCase (NOT Sonar's
all-lowercase); no collision with an in-scope name. **Replace EVERY cast within the pattern var's
scope**; a cast OUTSIDE scope (else branch, later statement, or to a DIFFERENT type) is a separate
site — leave it. **DROP** when: the cast can't reach the pattern var's flow scope (negated instanceof
with no early exit whose cast is under a separate positive instanceof; or a negated instanceof used as
a ternary/`&&` CONDITION whose cast is in the `:`/else branch — `x != null && !(x instanceof Y) ? ...
: ((Y) x)...` short-circuits via `x==null` too, so the var isn't definitely assigned); name collision;
unrelated expression. **Line length** is the #1 drop cause in dense feature modules — a long
pre-existing line (lambda-field initializer, tight decl) with no slack is an unavoidable drop.
Fix rate ~95-100% (oldcore ~98%, dense feature modules ~90-93%).
