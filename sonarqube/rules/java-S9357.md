# `java:S9357` — anonymous class on a functional interface should be a lambda

> *"Make this anonymous inner class a lambda"*. MAJOR, CODE_SMELL. The 2026 restatement of the old
> `S1604`, so **`dropped-issues.md`'s `S1604` entries apply to it verbatim** — check them before
> triaging (that is how the four `URIClassLoader` sites were recognised without a build).

## The pool

Mostly `src/test`: Mockito `Answer`/`ArgumentMatcher`/`VerificationMode`, `Runnable`, `Provider`,
`FilenameFilter`. The transform is mechanical and the **compiler is the whole verification** — a
lambda that does not match the target type simply does not compile.

## Drop conditions, all decidable before the build

1. **The target parameter is typed `Object`, not the interface.** The rule flags the anonymous class
   without looking at where it is passed. `TestComponentManager.registerComponent(Type roleType,
   String roleHint, **Object** instance)` is the XWiki case: the lambda has no target functional
   interface and does not compile. One grep of the callee's signature settles it. This is a genuine
   false positive — the site stays open.
2. **`AccessController.doPrivileged(new PrivilegedAction<T>(){…}, acc)`** — ambiguous with the
   `PrivilegedExceptionAction` overload. Permanent, and already recorded under `S1604`.
3. **The body's identity is observable.** `userReference.getClass().getSimpleName()` is `""` for an
   anonymous class and *not* empty for a lambda, so a test asserting the rendered message breaks.
   Seen in `NotificationPreferenceScriptServiceTest`, whose expected message is
   `"the given reference was a []"`. Grep the flagged type for `getClass()` /
   `getSimpleName()` / `getName()` uses on it before converting.
4. `this` inside the body referring to the anonymous instance (the rule's own caveat). Not seen in
   XWiki yet; `Outer.this.field` is *not* this case — it converts to `this.field`.

## Mechanics

* **Removing the anonymous class orphans imports** — `Answer`, `InvocationOnMock`,
  `ArgumentMatcher`, `VerificationMode`, `VerificationData`, `FilenameFilter`, and the type argument
  itself (`Block`, `DocumentModelBridge`). Commons' Checkstyle now checks unused imports in **test**
  sources too, so this is a build failure, not a nit. Run an orphan pass over every changed file
  after applying.
* **Two identical anonymous blocks in one file** (two `Runnable`s in
  `WrappedThreadEventListenerTest`) break an `assert count == 1`: extend each `old` upwards with the
  preceding unique line.
* **An anonymous class whose SAM returns another anonymous class converts too** — `new
  Iterable<T>() { iterator() { return new Iterator<T>(){…}; } }` becomes
  `() -> new Iterator<T>(){…}`, which is a ~30-line de-indent of the inner block. Worth doing; write
  it as one exact-string replacement.
* **Sonar under-reports the shape**: neighbouring anonymous `Answer`s that declare
  `throws Throwable` are *not* flagged even though they convert identically. Convert them in the
  same commit for intra-file consistency (they add no issue count) and say so in the PR body.
