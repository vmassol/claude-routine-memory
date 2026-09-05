# The `css:` facet — `S4666` `S4656` `S4657` `S4670` `S4651` `S8759` `S125`

The OKF's sonarqube corpus is Java-only and the `javascript:` rules live in this directory, so the
CSS facet lives here too. It is small (platform ~19, commons and rendering **0**) and it sits
entirely in `xwiki-platform-web-war/src/main/webapp/resources/`, i.e. the same module as the WAR
JavaScript pool. First swept 2026-09: **13 fixed / 5 dropped / 1 false positive**.

Two facts make it cheap: a CSS issue is decidable from the ~10 lines around it plus a scan of what
sits *between* the two blocks it names, and the `web-war` module build (`closure-compiler:minify` ×3
+ `yuicompressor:compress`) parses and minifies every file, so there is real verification.

## `css:S4666` "duplicate selector" — merge, and the drop condition is a shared PROPERTY

The fix is to append the later block's declarations to the first block for that selector. It is
behaviour-preserving when **no rule between the two blocks sets any of the merged properties for
that same selector** — check that, not just the two blocks. Order inside the merged block must stay
the original order (append, never prepend), because a longhand written after a shorthand
(`background` then `background-position`) is doing real work.

**Drop when the two blocks set the SAME property**, which is the only shape where merging is not
free. `job.css` `.ui-progress-bar` sets `background-color: X !important` in the first block and
`background-color: Y` in the second: the merge would leave two `background-color` declarations in
one block, i.e. a fresh `css:S4656`, and clearing *that* means deleting the declaration the
`!important` already makes dead — a decision about which colour was intended, not a cleanup.

When the second block carries an explanatory comment, the comment moves with its declaration into
the merged block (3 of 8 sites had one).

## `css:S4656` "duplicate property" — free, because the later one already won

Within one block the last declaration wins, so removing the earlier one is a no-op. Two shapes,
both clean: byte-identical repeats (`border: none` twice) and an overridden value (`right: 0` then
`right: 5px`, `padding: 0px` then `padding: 8px`). Sonar flags the FIRST occurrence, so the line the
issue points at is the line to delete.

## `css:S4657` shorthand overriding a longhand — check the shorthand's value first

Free only when the shorthand re-sets the longhand to the same value it had
(`background-color: transparent` followed by `background: transparent 2px center no-repeat`).
Otherwise the longhand is dead code with a different intent and deleting it is a judgement call.

## `css:S4670` "unknown type selector" — expect a Velocity FALSE POSITIVE

A `.css` file under `resources/` may be a **Velocity template**. `columns.css` holds a
`#foreach` / `#set` / `#end` block, which desynchronises the CSS parser and makes it read the
following `@media (max-width: 768px)` as a type selector. There is no `@SuppressWarnings` for CSS,
so the resolution is a SonarCloud `falsepositive` transition with the reason — done for that key.
The other shape (`.xobject-content xdt`, an element that does not exist) is a real dead selector,
but both deleting it and correcting it to `dt` change rendering: drop.

## `css:S4651` / `css:S8759` vendor-prefixed legacy — drop, they are not cleanups

`-moz-repeating-linear-gradient(left top -30deg, …)` and `@-moz-keyframes` are Gecko-only
fallbacks. Crucially `@-moz-keyframes progress-animation` was the **only** definition of the
animation that the `-moz-animation-name` above it refers to, so removing the at-rule deletes the
animation rather than modernising it, and adding the unprefixed forms switches it on in browsers
where it is inactive today. Both are product decisions — say so and move on.

## `css:S125` commented-out code

Same taxonomy as `java:S125`: an unanchored leftover (`/*cursor:pointer;*/`) is a clean delete.
