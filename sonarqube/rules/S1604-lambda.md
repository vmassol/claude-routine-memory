# java:S1604 — "Make this anonymous inner class a lambda"

Replace an anonymous class implementing a **functional interface** (single abstract method) with a
lambda: `new Comparator<X>(){ public int compare(a,b){ return E; } }` → `(a, b) -> E` (expression body
if the method just returns; block body `(args) -> { … }` otherwise). Common targets in XWiki:
`Comparator`, `Runnable`, `Visitor`, `BlockFilter`, `ElementSelector`, and even `Iterable`
(single abstract `iterator()`). Compiler-verified and behaviour-neutral — a solid mid-size safe pool
(~20 project-wide) spread one-or-two-per-file across many modules.

## Gotchas

- **`this` semantics change**: in an anonymous class `this` = the anon instance; in a lambda `this` =
  the enclosing instance. If the body references `this` (or its own instance/added fields/methods),
  DROP — it is not a plain conversion. Enclosing-field/method refs and effectively-final locals are
  fine (identical in both forms).
- **Lambda params share the enclosing method's scope** — a lambda param may not shadow a variable in
  scope. If the natural param name collides with an enclosing method param/local (e.g. a `BlockFilter`
  lambda inside `createLink(Block block, …)`), RENAME the lambda param (`block` → `child`) or it is a
  compile error. (Anonymous-class method params had their own scope and did not collide.)
- **Orphaned imports/Javadoc**: remove the now-unused interface import only with a word-boundary check
  that the simple name is truly gone (it often survives as the method's return/field type — keep it
  then). Delete any dangling `@Override`/method Javadoc left behind.
- **Line length**: a one-line lambda can breach 120; wrap to a 2-line/block body instead of dropping.

## Verification

Compiler catches type-inference/shadowing failures at `install`; run the module tests as usual. No
JaCoCo/Revapi impact.
