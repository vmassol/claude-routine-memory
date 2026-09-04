# `javascript:S3504` — "Unexpected var, use let or const instead"

## The one drop, and it is not a style question

`var XWiki = (function (XWiki) { … })(window.XWiki || {});` at the top of a WAR script **must stay
`var`**. These files are classic scripts, not modules: a top-level `var` creates the property on
`window`, `let`/`const` does not, and the whole "XWiki augmentation" pattern depends on the next
file seeing `window.XWiki`. Two of the 18 platform sites were exactly this (`uicomponents/lock/lock.js`,
`uicomponents/widgets/notification.js`). Everything *inside* the IIFE is function-scoped and free.

Say so in the PR body — leaving 2 issues open in files where you cleared every other `var` reads as
an oversight unless you name it.

## The safety argument for the rest, per site

`const`/`let` differ from `var` only in block scope and the TDZ, so a site is provably equivalent
when **the declaration sits directly in the body of the function (or of the block) that reads it,
and nothing reads the name before that line**. Check the reads, not the declaration: a
`const f = function() {…}` that is only called further down is fine, a self-recursive one
(`setTimeout(maybeScheduleRefresh, …)` inside its own body) is fine too. Use `let` for the one
variable that is reassigned, `const` for the rest — `const images = []` followed by `images.push(…)`
is correct, only rebinding is forbidden.

## Two co-located rules turn the drop risk into extra yield

The pool overlaps with rules that fire on the same lines, and the pre-push gate check
(`learnings.md`) will report them as collisions:

* **`javascript:S2814`** ("X is already defined") on a `var` you convert is a **hard error** if you
  ignore it — `let`/`const` cannot be redeclared, so the file stops parsing. The real fix is the same
  edit either way: declare the variable once at the top of the function and assign it in each branch,
  which is what the function-scoped `var` already meant. In `usersandgroups.js` that cleared 6
  `S2814` **plus** the 2 `javascript:S2392` ("referenced outside current binding context") on the
  same two variables.
* **`javascript:S4138`** ("expected a `for-of` loop") on a `for (var i …)` you were about to turn
  into `let i`: go straight to `for (const x of xs)` and both issues go. Safe when the body uses the
  index only as `xs[i]` and the receiver really is an Array — Prototype's `element.select(…)`
  returns one.

## Where it fires

Only in the *modern-looking* WAR scripts (`uicomponents/**` — `gallery.js`, `lock.js`,
`clean.js`, `sortPicker.js`, `notification.js`, `confirmedAjaxRequest.js`), never in the big
Prototype-era files that are full of `var` (`xwiki.js`, `livetable.js`, `suggest.js`). So a file
this rule flags is usually one where *every* `var` can go, which is what makes the intra-file
consistency story cheap: after the batch, `grep -nE '\bvar '` each changed file and expect only the
`var XWiki` line to remain.
