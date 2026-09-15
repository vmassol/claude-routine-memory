# `java:S1181` — "Catch Exception instead of Throwable"

**Recorded twice as a whole-rule drop ("narrowing what is caught is a behaviour change") and both
times that is an objection to the REMEDIATION THE MESSAGE NAMES, not to the finding.** Same escape
axis as `S2176`/`S9149`/`S2065`/`S1214`: the sentence that closed the rule is the sentence the
`@SuppressWarnings` comment has to say. 48 of 54 shipped as suppressions across all three repos.

## Why the suppression is the resolution, not a dodge

The rule's premise is *"an application should not attempt to recover from an `OutOfMemoryError`"*.
Both remediations it offers are wrong on this pool:

* narrowing to `Exception` lets an `Error` raised by the third-party code behind the boundary escape
  and break exactly the thing the boundary protects — a behaviour change, and the one the code was
  written to prevent;
* letting it propagate is the same thing.

So there is no compliant form of the code, which is the definition of a false positive.

## The free classifier — three keep shapes, all decidable from the flagged `catch` block

1. **The reason is already a comment inside the block.** Grep the block for it before anything else;
   XWiki states it far more often than expected — *"protect from bad listeners"*, *"Monitoring must
   never break the request being monitored"*, *"We catch Throwable because most of the time we end up
   with a LinkageError"*, *"no reason to crash the whole cache because of some badly implemented
   dispose() we don't control"*, *"we want to never break the whole rendering"*, *"Solr create Runtime
   exceptions in case of client issues"*. The team decided and wrote the sentence; only the Sonar key
   was missing. This is the `S1214` story with a different linter.
2. **Nothing is recovered from** — the block logs and rethrows, releases a resource and rethrows with
   the failure `addSuppressed`, or wraps the Throwable into the exception the API declares. The rule's
   premise does not hold at all, which is the least arguable sentence available.
3. **The method's own contract is "never throws"** — `InternalTemplateManager#getXDOMNoException` /
   `#executeNoException` are named after it; `SafeArrayConverter` / `SafeTreeMarshaller` /
   `SafeTreeUnmarshaller` exist to make XStream (de)serialization total.

## The drop condition is truthfulness, and one shape of it is new

The usual "no statable reason" drop applies (an unlogged `catch (Throwable t) { return null; }` in
crypto code — `BcStoreX509CertificateProvider`; an enum `valueOf` with no boundary to protect —
`DefaultURLConfiguration`). The shape worth naming: **a `// TODO:` inside the block that objects to
the clause itself**. `FeedPlugin#getSyndEntrySource` carries *"catching Throwable here also hides a
failure of the constructor that was found"* — the code calls its own clause a problem, so suppressing
it asserts the opposite of what the file says. Read the block's comment for *which way it points*, not
merely for whether one exists: the same comment that is the strongest KEEP evidence in shape 1 is the
strongest DROP evidence when it reads as an apology.

Corollary on scope: when two flagged sites sit in one method and only one is truthful, the method-level
annotation covers both — so the honest verdict drops the whole method (both `FeedPlugin` keys went out
together for that reason).

## Mechanics

* **`@SuppressWarnings` cannot go on a `catch`**, so the target is the enclosing METHOD. Several sites
  in one method collapse to one annotation (`XWikiAction#execute` 4 → 1, `MacroTransformation#transform`
  2 → 1, `AttachInputSocket#channel` 2 → 1): 48 issues were 43 annotations.
* **Method-level, not class-level**, per `okf/conventions/code-style.md` — even for `MonitorPlugin`,
  where six methods share one sentence. (That contradicts the `S2065` file's "class-level for 3+",
  which was about a four-line comment; a one-line reason is cheap to repeat.)
* The `//` reason goes directly above the annotation, and the annotation last before the declaration
  (after `@Override`), so the inserted block lands *between* `@Override` and the signature.
* Locating the enclosing method by "walk up to the nearest line at 4-space indent containing `(`"
  works everywhere except a catch inside an **inner class** (`NotificationEventExecutor.CallableEntry`
  sits at 8 spaces and the walk-up finds a field). Assert the found line starts with the signature
  prefix you expect and hand-fix the misses — one of 43 here.
* Insert-only, `SOURCE` retention, bytecode byte-for-byte identical: say it in the PR body, it answers
  the whole safety question and rules out Revapi and JaCoCo.
