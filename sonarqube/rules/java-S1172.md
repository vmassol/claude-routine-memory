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
  overload, cannot compile. **With one exception that broke `master`, see below.**
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
   **Its strongest form is a `switch`-driven dispatch family**: platform's
   `DefaultXWikiDocumentMerger#OVERWRITE/SKIP/SKIP_ALLWAYS` are three CONSTANT_CASE private methods
   called from three arms of one `switch` with deliberately uniform argument lists (3 issues in one
   file). Recognise it from the flagged declarations alone — same file, same shape, same arity — and
   drop the whole family; the same reasoning took
   `ExpressionNodeToEventQueryConverter#parseUnaryOperator`, which mirrors `parseBinaryOperator` and
   `parseBlock` on the `ingroup` flag.
4. **The parameter is threaded down a private chain from a PUBLIC entry point.** Removing it at the
   frontier only pushes the same issue one level up, and the chain terminates at API you cannot
   change — so the rule regenerates in the file you just fixed and you gain nothing.
   `DocumentLocaleReader#readXMLElement(xmlReader, filter, proxyFilter)` is fed by the private
   `read(xmlReader, filter, proxyFilter)`, which is fed by the **public**
   `read(Object filter, XARInputFilter)`. Cheap test: grep the callers; if the argument at the call
   site is itself a parameter of the calling method, walk one more level before committing.

Everything else converted: **41 of 45 private sites, 0 build failures beyond the two below.**

**The rule REGENERATES from ordinary refactoring — re-bucket it every run, it is ~2 calls.** Eleven
runs after the pool was declared "the non-`private` residue", the same query found **13 fresh
`private` platform sites and 2 commons** (platform's totals had moved 132 → 97 open, i.e. the public
residue shrank while new private sites appeared). Of those 15, 5 shipped and 10 were drops — a much
higher drop rate than the original sweep's 9%, because the easy shapes were taken first and what
regenerates is disproportionately dispatch families and FIXME-annotated parameters. Budget it at
~⅓ conversion now, not 90%.

**Outcome: all three PRs merged, none with a single review comment** — platform #6214 (35 sites,
mostly oldcore `src/main`), commons #1918, rendering #408. So the private subset is not merely safe,
it is uncontroversial: `src/main` is not a reason to hold a site back, and the visibility split is
the whole judgement this rule needs.

## The one thing "private ⇒ the compiler is the whole verification" misses: AspectJ

**Platform #6214 broke `master`.** It shortened the private
`BooleanClass#getDisplayValue(XWikiContext, int)` to `getDisplayValue(int)`; oldcore compiled and its
1149 tests passed, but `xwiki-platform-legacy-oldcore` then failed its **`ajc`** compile —
`BooleanClassCompatibiityAspect.aj` still called `getDisplayValue(context, 0)`. An **inter-type
declaration (`public String BooleanClass.displaySelectSearch(…)`) is compiled *as* a member of its
target class, so it can call that class's `private` methods** — from a different module, in a source
file `javac` never sees and Sonar never scans. The breakage surfaced only a run later, in a reactor
that happened to include the legacy module; fixed by `xwiki/xwiki-platform#6218`.

Two guards, both nearly free, for any rule that changes a `private` (or any) member of an oldcore
class:

* Grep the aspect sources for the member name before applying:
  `grep -rn "\bname(" --include=*.aj .` — the compatibility aspects live in
  `xwiki-platform-*-legacy-*/src/main/aspect/**` (and `xwiki-commons-legacy-*`,
  `xwiki-rendering-legacy-*`). A hit means the "private" argument does not hold for that site.
* **Put the matching `*-legacy-*` module in the `-pl` list** whenever the batch touches a class that
  has a `*CompatibilityAspect.aj` / `*CompatibiityAspect.aj` (both spellings exist in-tree). A
  reactor that builds `xwiki-platform-oldcore` but not `xwiki-platform-legacy-oldcore` cannot see
  this class of failure at all.

Generalises past this rule: **"the compiler is the whole verification" is only true when every
compiler that sees the member is inside your reactor** — and in XWiki one of them is `ajc`, in
another module, on sources no other tool reads.

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
