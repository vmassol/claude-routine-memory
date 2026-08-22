# `java:S1172` — "Remove this unused method parameter"

**OKF-denylisted as "a real behaviour or signature change". That is right for a non-`private` method
and wrong for a `private` one** — and the private subset is a real, deep, three-repo pool
(platform 132 → 39 private, commons 25 → 5, rendering 5 → 1). Fourth rule rescued from a
prior-shaped rejection after `S1117`, `S3252` and `S3415`.

## Why the private subset is provably safe

* Sonar's own exceptions already remove the hard cases: it never fires on an **override or interface
  implementation**, on an annotated method, on a non-private method with an empty body or that only
  throws, or on an overridable method whose parameter is documented in Javadoc. So the flagged set is
  already narrowed, and the only remaining question is **visibility**.
* A `private` method cannot be called from outside its compilation unit, so **the compiler is the
  complete verification**: a missed call site, or a shortened signature that collides with an
  overload, cannot compile.
* Non-`private` is a permanent drop (Revapi / cross-module callers). The split is free — read the
  modifier at the start of the flagged declaration line. Platform's 132 split 39 private / 56 public /
  18 protected / 18 package-private.

## The classifier and the drop shapes

Decidable from the flagged declaration plus the call sites in the same file:

1. **The shortened signature collides with an existing overload.** `SheetDocumentDisplayer`'s private
   `display(DocumentModelBridge, DocumentModelBridge, DocumentDisplayerParameters)` minus its first
   parameter *is* the public `display(DocumentModelBridge, DocumentDisplayerParameters)` it delegates
   from — "method is already defined". Guard **before** applying: look for another declaration of the
   same name whose arity is `nparams - 1`, not just `nparams`. (Checking only `nparams` is what
   correctly refused the `EntityReferenceConverter#convertToType(Type, Object)` overload pair but
   missed this one.)
2. **A `TODO`/commented-out block in the body explains why the parameter is there** — the universal
   "a comment on the flagged code is authoritative" condition, and it was 2 of 42 sites
   (`sendRUNNINGResponse(status)` with a `// TODO: Send back a REST version of the job status`,
   `getMappedCategoriesForURL(categories)` with a BLOG-198 TODO and the using code commented out).
3. **The parameters mirror a sibling method the callers pair it with.** `MessageStreamTest`'s
   `setupForLimitQueries(expectedLimit, expectedOffset)` is always called next to
   `verifyLimitQueries(expectedLimit, expectedOffset)`; removing one of the two breaks the pairing a
   reader relies on. (Sonar flags only one of the two parameters, which makes the asymmetry worse.)

Everything else converted: **41 of 45 private sites, 0 build failures beyond the two below.**

**Outcome: all three PRs merged, none with a single review comment** — platform #6214 (35 sites,
mostly oldcore `src/main`), commons #1918, rendering #408. So the private subset is not merely safe,
it is uncontroversial: `src/main` is not a reason to hold a site back, and the visibility split is
the whole judgement this rule needs.

## Applying it

* Rewrite the declaration **and every call site**, from the last file offset backwards. Locate call
  sites by `(?<![\w$.])name\s*\(` and keep only those whose top-level argument count equals the
  declaration's parameter count — that is what separates an overload's calls from this method's.
  Assert exactly **one** declaration matches (a declaration is `)` + optional `throws …` + `{`).
* Splitting a parameter/argument list needs a scanner that tracks parens, brackets, **angle brackets**
  (`Map<String, Serializable>` is one parameter) and string literals.
* **When the removed parameter is index 0, strip the whitespace the next argument inherited**, or
  every site comes out as `f( b, c)`.
* **Removing an argument can orphan the local that produced it.** In commons it left an `instanceof`
  pattern binding unused → Checkstyle `UnusedLocalVariable`, which fails the build *after* the tests.
  Cheap post-apply scan: for every bare identifier removed from a call line, count remaining
  word-boundary occurrences in the file; `<= 1` means only its declaration is left. Fix by dropping
  the binding (`x instanceof T name` → `x instanceof T`).
* **Delete the orphaned `@param` tag** in the same edit (about a third of the sites had one).
* Check each removed argument expression for side effects. In practice they are all bare identifiers,
  literals or pure accessors (`list.get(i)`, `doc.getDocumentReferenceWithLocale()`,
  `(String) data[2]`), so nothing observable is lost — but say so in the PR body, it is the first
  question a reviewer asks.

## Where the pool sits

Thin-spread with one dense file per repo. Platform: oldcore 21 (`XWikiHibernateStore` 3 `bTransaction`
leftovers from before `executeRead(…)`, the three `*EventGeneratorListener#onDocumentCreatedEvent`
copies, `XWiki` 2), `DocumentLocaleReader` 7, then singletons over 7 more modules. Commons 5 in 5
different modules; rendering 1.
