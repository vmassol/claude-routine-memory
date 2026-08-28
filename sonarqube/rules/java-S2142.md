# `java:S2142` — `"InterruptedException"` and `"ThreadDeath"` should not be ignored

BUG/MAJOR, and **the pool splits for free on the catch clause's TYPE** — no source read beyond the
`catch` line and the first statement of the handler. No OKF entry.

## The transform

```java
} catch (InterruptedException e) {
    Thread.currentThread().interrupt();
    …unchanged…
}
```

One inserted line as the **first statement** of the handler, indented one level deeper than the
`catch`. Nothing else changes — no log message, no control flow — so the whole batch is a
`(file, catchLine + firstBodyLine)` anchor table and the insertion point is unambiguous.

It applies to two shapes that look different but lose the same thing:

- the handler **logs or ignores** and continues/returns (~half the pool);
- the handler **wraps into an unrelated exception type** and throws (`XWikiException`,
  `XWikiRestException`, `MojoExecutionException`, …). Sonar flags this too, correctly: the wrapper
  carries the cause but the *interrupt status* is gone, so nothing upstream can see that cancellation
  was requested.

## Drop condition: a BROAD catch clause

`catch (Exception e)` / `catch (Throwable e)` is flagged only because `InterruptedException` is a
subtype. An unconditional `Thread.currentThread().interrupt()` there would set the flag for **any**
exception, so the real fix needs an added `catch (InterruptedException e)` clause or an `instanceof`
test — a per-site design decision, not a cleanup. **Drop those** (9 of 30 platform sites: 7 broad
catches + 2 multi-catch, see below).

A **multi-catch** (`catch (ComponentLookupException | InterruptedException e)`) is the same problem
one step down. Fix it only when the handler *already* has an `if (e instanceof InterruptedException)`
branch — then the line goes inside that branch, which is free. Introducing the test just for this is
a readability judgement: drop.

## Sites that need a second look before shipping

A site whose handler feeds a **loop condition** deserves one read of the enclosing loop. The
platform pool had one (`DefaultSolrIndexer#startIndexerThread`, `while (!Thread.interrupted())`):
restoring the flag there does not change the outcome, because the handler already queues a STOP entry
that breaks the loop on the next statement, and if it ever did not, the loop condition would now stop
the thread — which is the intent. Note `Thread.interrupted()` *clears* the flag, so a loop written
that way consumes it exactly once.

## Verification

The compiler proves nothing here (an inserted statement always compiles), so the module's own tests
are the whole verification — and they are sufficient in practice, since no test asserts on a thread's
interrupt status. What the fix *can* change at runtime: a later blocking call on the same thread now
throws `InterruptedException` immediately instead of blocking. That is the point of the rule, but it
makes the batch a **judgement PR**, not a mechanical one — ship it separately from the run's safe
batch so a maintainer can drop it without holding up the rest.
