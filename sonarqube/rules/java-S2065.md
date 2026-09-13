# `java:S2065` — "Remove the `transient` modifier from this field"

**OKF-denylisted as "load-bearing in XWiki: job statuses and requests are serialized with XStream,
which honours `transient`". That reason is correct, and it is therefore not a drop — it is the
finished argument for an FP suppression.** Same escape axis as `S2176`/`S9149`: the drop entry
already contains the sentence the `//` comment has to say, so the rule is a pool, not a dead end.

The rule fires on a `transient` field of a class that does **not** implement `java.io.Serializable`,
on the premise that `java.io` serialization is the only reader of the modifier. In XWiki it is not.

## The proof, and it is three greps in `xwiki-commons`

Do not assert "XStream honours transient" — show it, it is the strongest paragraph in the PR body:

1. `org.xwiki.job.internal.JobStatusSerializer#write` serializes the job status with
   `org.xwiki.xstream.internal.SafeXStream`.
2. `SafeXStream extends XStream` and installs a `SafeReflectionProvider`, whose
   `visitSerializableFields` delegates to the JVM reflection provider — which skips `transient` and
   `static` fields.
3. Nothing on that path reads `java.io.Serializable`.

## The free classifier: what is NOT transient one level up

The whole triage is *"is this object reachable from something the job status store writes out?"*, and
the answer is a one-line grep on the **owner's** field, not on the flagged class:

| Group | The non-transient field that makes it reachable |
|---|---|
| the job status itself | it *is* what `JobStatusSerializer` writes |
| job progress | `AbstractJobStatus#progress` is not transient |
| distribution step | `DistributionJobStatus#stepList` is not transient |
| extension repository | `AbstractExtension#repository` is not transient, and extensions travel in job statuses |
| derived-value cache (`idStringCache`, `hashCode`, `urlCache`, `serialized`) | the value object travels inside a serialized request/status |

**The drop condition is the mirror image: when nothing one level up is serialized, there is no true
sentence to write.** Two platform shapes failed it and stay open (keys in `dropped-issues.md`):
`AbstractJob`'s *own* `@Inject`ed fields are **not** `transient`, so the codebase does not treat a
`Job` as serialized — which kills `IndexerJob`'s three sites — and nothing serializes a
`DefaultQuery`, which kills `DefaultQuery#executer`. A comment asserting an intent the code does not
support is worse than the open issue; that is the same truthfulness gate as `S1186`.

## Mechanics

* **Insert-only, and `@SuppressWarnings` has `SOURCE` retention — the compiled bytecode is
  byte-for-byte identical.** Say that in the body; it answers the whole "is this safe" question and
  it also means JaCoCo and Revapi cannot be affected.
* **Scope: field-level for a file with 1-2 flagged fields, class-level for 3+.** `code-style.md` asks
  for the narrowest scope, but repeating a four-line comment nine times in `XWikiExtensionRepository`
  is worse for the reader than one statement about the class. State the splitting rule in the body.
* **Two classes already carry `@SuppressWarnings("checkstyle:ClassFanOutComplexity")`**
  (`AbstractJobStatus`, `XWikiExtensionRepository`) and Java forbids a second `@SuppressWarnings` on
  one element — **merge the key** (`{ "checkstyle:…", "java:S2065" }`), never add a second
  annotation. Any batch of this shape needs that branch.
* **Derive the insert indentation from the flagged line, not a fixed four spaces.** A flagged field
  can sit in a nested static class (`PDFExportJobStatus.DocumentRenderingResult#xdom`, 8 spaces) and
  the misindent is a Checkstyle failure *after* the tests.

## Where the pool sits

Thin-spread over `xwiki-commons-extension-*` and `xwiki-commons-job-api`, platform's
`xwiki-platform-extension-distribution`, and three `xwiki-rendering-api` value objects — i.e. **all
three repos**, which is what a multi-repo run needs on a day the mechanical catalogue is swept.

## The sibling `java:S1948` is NOT the same argument — it is untriaged

`S1948` ("make this field `transient` or serializable") is the OKF's "exact inverse", but its fix
*adds* `transient`, so the XStream argument does not transfer unchanged: for a class that really is
serialized, adding the modifier changes what is persisted (a drop), while for a class Serializable
only by inheritance (`XWikiContext extends Hashtable`) the argument is "nothing ever serializes
this", which is weaker and has to be made per class. ~59 open, unclaimed, never triaged.
