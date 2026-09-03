# `java:S5993` — "Constructors of an `abstract` class should not be declared `public`"

OKF-denylisted as *"reduce a constructor to `protected` — **reduces visibility** → Revapi
`java.method.visibilityReduced`"*. **The reason is false for the WHOLE pool** — `revapi:check`
accepts the change on published, non-`internal` API too. The rule is a plain mechanical pool with no
visibility split at all, and the two runs that found this out are worth reading in order because the
second one corrects the first.

- **Run A** split the pool on `/internal/` in the path (the packages `revapi.json` excludes) and
  shipped **81 internal sites**, leaving the rest recorded here as a "permanent drop".
- **Run B** ran the gate instead of reasoning about it: four non-`internal` sites in
  `xwiki-commons-cache-api`, then `mvn package revapi:check -Pquality -DskipTests -pl <that module>`
  — **2:01**, `Comparing [18.7.0] against [18.8.0-SNAPSHOT] … API checks completed without
  failures`. That re-opened the whole residue and shipped **162 sites** across the three repos.

## Why Revapi does not care

`revapi.json` (in `xwiki-commons-tool-verification-resources`) ends with an explicit reclassification:

> *"We reclassify all source compatibility checks to be of EQUIVALENT severity so that the build
> doesn't fail on source incompatibilities since we're only interested in binary and (some) semantic
> incompatibilities."*

Reducing an **abstract** class's constructor to `protected` is a source-only change — see the JLS
argument below — so it lands in exactly that bucket. The `internal`/`test` package excludes are a
*second*, independent reason the internal subset is safe; they were never the only one.

## Why it breaks nothing at all (not just nothing Revapi checks)

For an **abstract** class there are exactly two ways to reach a constructor, and
[JLS §6.6.2.2](https://docs.oracle.com/javase/specs/jls/se21/html/jls-6.html#jls-6.6.2.2) permits
both on a `protected` constructor **from any package**: a `super(...)` call in a subclass, and an
anonymous class instance creation `new AbstractX(...){…}`. The one form `protected` would forbid,
`new AbstractX(...)`, is already illegal — the class is abstract. So no *compilable* caller can exist
that the change breaks, in any package, in any module. That is a proof, not an argument; it plus the
`revapi:check` result is the whole PR body.

## Mechanics

- The flagged line is uniformly `    public <ClassName>(`; the fix is `public ` → `protected `, one
  token, nothing else on the line.
- **The line grows by 3 characters**, so re-check 120 columns — 2 of 81 and 2 of 162 sites needed the
  parameter list re-split onto a continuation line (the list itself unchanged).
- Assert the class really is `abstract` (`abstract\s+class\s+<Name>`) per file before writing.
- No import, no body, no signature and no `@since` — the constructor's descriptor is unchanged, so
  there is nothing to document.
- `.**.` in the revapi exclude pattern matches ZERO segments, so
  `com.xpn.xwiki.internal.event.AbstractXObjectEvent` is excluded too. Irrelevant now that the whole
  pool is fair game, but it is why run A's internal bucket was bigger than it looked.

## Drop condition

Only the universal one that bites this rule: **a flagged declaration line that already carries an
open method-metric issue** (`S3776`, `S107`, `S1541`). Rewriting the line hands that pre-existing
finding to your own `Quality / Analyze` gate. One site in 163 hit it
(rendering `test/integration/AbstractInternalRenderingTest.java:89`, `S107` with 8 parameters).
Compute the collision set from the project's open issues before applying — no source reads needed.

## Outcome

- Run A: **81 sites merged** in all three repos within hours, not one review comment
  (`xwiki/xwiki-platform#6226` 59, `xwiki/xwiki-commons#1926` 20, `xwiki/xwiki-rendering#414` 2).
- Run B: **162 sites** (`xwiki/xwiki-platform#6293` 83, **`xwiki/xwiki-commons#1945` 60 and
  `xwiki/xwiki-rendering#427` 19 — both MERGED by Vincent within a minute of each other, ~4 h after
  opening, with zero review comments on either**) over 33
  modules, 4120 tests green, `revapi:check` passing in every one of them. So the visibility question
  did not come up on the *non*-`internal` half either: the two proofs in the body (the `revapi:check`
  output and JLS §6.6.2.2) are what a reviewer needs, and a rule rescued by running its gate merges as
  quietly as one rescued by re-reading its reason.
