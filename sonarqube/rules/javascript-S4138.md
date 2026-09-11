# `javascript:S4138` — "expected a `for-of` loop instead of a `for` loop with this simple iteration"

Recorded for a long time as one of the JS "traps" ("changes behaviour in at least one shape"). It is
not a trap; it has a **free classifier and a receiver check**, and it went 14/14 with 0 drops beyond
the vendored files in the first sweep that actually looked (platform #6321, WAR resources).

## The two checks, both from the ~8 lines the issue points at

1. **The index must be used only as `collection[i]`.** If the body reads `i` for anything else
   (an offset, a message, a sibling array) the loop is not a simple iteration — drop it. Nothing in
   the 2026-09 platform pool failed this.
2. **The receiver must implement `Symbol.iterator`.** Plain Arrays do (`Object.keys(...)`,
   `String#split` results, jsTree node arrays). So does `NodeList` (`mutation.addedNodes`). So does
   a **jQuery object** — but only from jQuery 3.0 on, so read the version out of the pom before
   converting a `$(…)` receiver (`xwiki-platform-web-war/pom.xml` pins `jquery` 4.0.0). An
   `arguments` object or a bare `{length: n}` array-like is the drop.

`files.item(i)` is a *different shape* and Sonar does not flag it, so leaving it alone is not an
intra-file inconsistency a reviewer will point at.

## Mechanics

* Declare the binding `const` — that is the rule's own compliant form, and it avoids re-introducing
  `javascript:S3504`/`S2814` on a fresh `var`. Leave the body's existing `var`s alone; the diff
  stays minimal and the file does not start speaking two dialects mid-function.
* The `var x = collection[i];` line on the first line of the body **is** the new loop binding —
  delete it and name the binding after it. When the body then wraps it (`var category =
  $(categories[i])`), name the binding `<thing>Element` and keep the wrap.
* Nested index loops convert together (`async.js`: `for i` over `mutations`, `for j` over
  `mutation.addedNodes`) and the inner body's indentation does not move, because the nesting depth
  is unchanged.
* Never name a binding `event` — it shadows the browser global. Use `eventName`.

## The expensive one: converting a file to ES6 CHANGES WHICH RULES SONARJS RUNS ON IT

Measured on `uicomponents/widgets/select/select.js`: **0 `javascript:S3504` issues on `master`, 19 in
the pull-request analysis.** Introducing a `const` loop binding is what flips SonarJS into treating
the file as ES6, and it then reports *every* `var` in it — including declarations the diff never
touched. Two consequences, and neither is visible to any pre-push check, because on `master` the
rule does not fire in that file at all:

1. **The three `var`s directly under a converted loop header are the PR's own**, because the
   conversion rewrites that line (`var category = $(categories[i]);` →
   `var category = $(categoryElement);`). `Quality / Analyze` fails on exactly those. Declare them
   `const` in the same edit — check first that each is assigned once and read only inside the loop.
2. **The remaining `var`s of the file become reported on `master` once the PR merges.** They are not
   attributed to the PR (right lines, wrong author) so the gate stays green, but the project's open
   count goes up. Say so in the PR body and offer the whole-file `var` → `let`/`const` conversion as
   a separate change rather than slipping it in — each one needs its own scope check.

Generalises beyond this rule: **any fix that introduces a modern construct into a legacy script can
switch on a whole rule family for that file.** Before converting, ask what the file currently does
*not* get flagged for that it "should" — one
`issues/search?componentKeys=<projectKey>:<path>&resolved=false` on master, compared with the same
query on `&pullRequest=N` after the first analysis, is the only way to see it.

## The ES6 flip is not just a gate risk — do the ARITHMETIC, it is usually a NET LOSS

The recorded hazard ("adding a `const` loop binding flips SonarJS into ES6 mode and it then reports
`javascript:S3504` on every `var` in the file, three of them on lines the conversion rewrote, so
`Quality / Analyze` failed") is the *symptom*. The decision rule is bigger: those new `S3504` issues
exist whether or not any lands on a written line, so the trade is **2 issues fixed against ~30
created** in a several-hundred-line Prototype-era script. The one-call pre-check is to read the
file's current `javascript:S3504` count — **zero means the file is not yet parsed as ES6**, i.e. you
are the one flipping it. Two such sites were dropped on that basis (`create.js:148`,
`locationPicker.js:436`), and the right fix for those files is to move them off `var` wholesale,
which is its own change. Note `for (var x of …)` does not save you: `for…of` is ES6 syntax
regardless of the binding keyword. Where the file **already** contains `let`/`const`/`?.` the flip
has happened and the conversion is free again.

## Where the pool is

Platform only (commons and rendering have no JavaScript), ~60 open, concentrated in
`xwiki-platform-web-war/src/main/webapp/resources/`. A large share sits in the **vendored** scripts
(`tablefilterNsort.js` alone holds 14, `ieemu.js` 3) — check the file header and interior for
third-party provenance first, per the recorded rule.

# `javascript:S1940` — "use the opposite operator" — expect a FALSE POSITIVE

Sibling found in the same sweep, and the one JS site of that rule in the WAR is wrong:
`if (!(delay >= 0))` is **not** `if (delay < 0)`. The negated form is also true for `undefined` and
`NaN`, which is exactly what a guard on an optional numeric argument is for. Resolved as
`falsepositive` in SonarCloud with that reason. Ask it of every `S1940` site whose operand can be
non-numeric: the rule assumes a total order the value may not have.
