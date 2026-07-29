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
  **When that decl line was followed by a blank line, deleting it leaves a stray leading blank right
  after the pattern-`if {`** — remove that blank too (grep the changed files for `{\n\n` / eyeball the diff).

**The pool is far deeper in the SIBLING repos than in platform** — when platform's S6201 is down to a
handful (2), xwiki-commons can still hold ~226 and xwiki-rendering ~52, with ZERO open agent PRs on
them. Commons densest: `extension-api` ~45 (**but that module is red on master — see
`dropped-issues.md`, don't pick it**), the whole `crypto` tree ~77, `filter-api` ~17,
`logging-api` ~12, `job-api` ~10. Rendering is NOT mostly `wikimodel` (1 site) — it is
`xwiki-rendering-api` 24 (the `Block` hierarchy's `equals()`), `transformation-macro` 10,
`xdomxml10` 7, rest 1-3 each. Datapoints: `xwiki-commons-xml` + `filter-xml` = 34 sites 34 fixed /
0 drops; the **commons crypto tree = 77 sites, 73 fixed / 4 dropped** (all 4 drops in one module,
for coverage — see below); **all 52 rendering sites fixed / 0 drops**. Commons/rendering S6201 is
the single best lever for a 30+-fix target.

**Coverage is the real drop cause on this rule, not correctness.** Removing a COVERED `CHECKCAST`
lowers the module's JaCoCo instruction ratio — `(c-k)/(t-k) < c/t` whenever `c<t` — so a module
pinned just above its ratio goes red under `-Pquality` even though nothing else changed. Live case:
`xwiki-commons-crypto-cipher`, 4 sites, ratio 0.70 → 0.69, `jacoco:check` fails. Don't lower the
pinned ratio; DROP that module from the batch (`git checkout -- <module>/`, remove it from `-pl`)
and ship the rest. Expect this on small modules with few sites; big modules absorb it fine.

**Equals()-heavy files are the cleanest fodder there is.** `if (obj instanceof XBlock && super.equals(obj))`
+ several `((XBlock) obj).getY()` in an `EqualsBuilder` chain = one edit resolving 1-3 issues, zero
dataflow. Name the pattern var `otherBlock`/`otherSource` (not the type name) to avoid colliding with
a field. A whole `Block`-style hierarchy converts in one script.

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

**Mechanics & drops.** STRUCTURAL, but it does NOT require subagents — a single assert-guarded
`(file, old, new, nb_issues)` script (see `learnings.md` → *General batch-fix techniques*) handles
120+ sites reliably and beats delegation on accuracy, since each `old` is a verbatim multi-line
block asserted to occur exactly once. Read the sites in ONE batched `±6 lines` snippet dump per
module group, then write the whole script. Track `nb_issues` per edit (an edit often resolves
2-7 issues) and assert the sum against the Sonar `total`. A pattern var can be a
try-with-resources resource (effectively final). **Naming:** idiomatic camelCase (NOT Sonar's
all-lowercase); no collision with an in-scope name. **Replace EVERY cast within the pattern var's
scope**; a cast OUTSIDE scope (else branch, later statement, or to a DIFFERENT type) is a separate
site — leave it. **DROP** when: the cast can't reach the pattern var's flow scope (negated instanceof
with no early exit whose cast is under a separate positive instanceof; or a negated instanceof used as
a ternary/`&&` CONDITION whose cast is in the `:`/else branch — `x != null && !(x instanceof Y) ? ...
: ((Y) x)...` short-circuits via `x==null` too, so the var isn't definitely assigned); name collision;
unrelated expression. **Line length** is the #1 drop cause in dense feature modules — a long
pre-existing line (lambda-field initializer, tight decl) with no slack is an unavoidable drop; it is
usually RECOVERABLE by re-wrapping the condition, and when you do, XWiki style puts the `{` **on its
own line** for a multi-line `if` condition (copy the shape already in the file). Fix rate ~95-100%
(oldcore ~98%, dense feature modules ~90-93%, commons/rendering ~95-100%).

**Sonar reports one issue PER CAST, all keyed to the `instanceof` line** — so a single line can carry
2, 4, even 7 issues (`if (password instanceof KeyWithIVParameters)` with 4 casts in its block = 4
keys on that line). Don't read a repeated line number as a duplicate; count the casts in the block to
reconcile. Casts to a DIFFERENT type inside the same block are NOT flagged (they're a separate check),
and `(Map<?, ?>) x` style **generic** casts are never flagged — leave both alone.

**Flow-scoping shapes that DO work** (Sonar flags them, and they compile):
- `if (!(x instanceof T t)) { … } else { /* t is in scope here */ }` — no early exit needed; the
  `else` branch is exactly the "true" case.
- `if (obj instanceof T t) { return t; }` followed by a later `T2 t2 = …` — when the then-block
  completes ABRUPTLY the pattern var leaks into the enclosing scope, so a later declaration with the
  SAME name is a compile error. Give the two different names (this is a real trap in the
  `getInstance(Object obj)` ASN.1 converters, which chain 2-3 such ifs).
- `if (a instanceof T t) { … } else if (a instanceof U u) { … }` — reusing the same name in both
  branches is legal (the first is not in scope in the else), but distinct names read better.
