# `java:S1186` — "Methods should not be empty"

**CRITICAL, 192 open across the three repos (platform 112, commons 50, rendering 30), never attempted
by this routine until now — 172 shipped in one sweep, 0 drops for correctness.** The rule's own
remediation is *"add a nested comment explaining why this method is empty, throw an
`UnsupportedOperationException`, or complete the implementation"*, and in XWiki the first option is
almost always the right one: the pool is overwhelmingly **deliberate** no-ops.

```java
     @Override
     public void sendError(int sc) throws IOException
     {
+        // Nothing to do, this stub does not send any response.
     }
```

Comment-only: no signature, no behaviour, no API, nothing for Revapi or Checkstyle to say. The whole
risk of this rule is **whether the comment is true**, not whether the edit is safe.

## Why it is a real pool and not noise

XWiki already uses this idiom — ~150 `// Nothing to do` / `// Do nothing` comments across the three
repos before this sweep — so the fix follows house style rather than inventing one. Grep it before
arguing the point:

```
grep -rniE "^\s*//\s*(nothing to do|do nothing|no-op|intentionally (empty|left blank))" --include=*.java .
```

## The free classifier: the pool is CLUSTERED by class, not scattered

**Group the sites by FILE first — the comment is a property of the class, not of the method.** 192
issues sat in only 78 files, and the top 6 files held 96 of them (`PrintTextListener` 26,
`TestFilterImplementation` 25, `HttpServletResponseStub` 21, `Sax2Dom` 10, `VoidMailListener` 8,
`HttpServletRequestStub` 6). One sentence per class, applied to every empty body in it, is both the
cheapest triage and the most reviewable diff — and it means the whole batch needs ~20 class-level
judgements, not 192 method-level ones.

Six recurring shapes cover essentially the whole pool, and the **class name or its Javadoc names the
shape**, so no method-body reading is needed:

| Shape | Tell | Comment |
|---|---|---|
| Stub | `*Stub`, `*RequestStub` | *this stub does not implement this part of the API* |
| Void / null object | `Void*`, `Null*`, Javadoc "doing nothing" | *this listener voluntarily ignores all events* |
| Default hook | `protected`, non-`@Override`, subclasses override it | *nothing to do by default, meant to be overridden by subclasses* |
| Deprecated no-op | `@Deprecated` above the flagged line | *this deprecated setter is kept for backward compatibility only* |
| Component-manager constructor | Javadoc **already says** "Empty constructor needed for component manager" | *this empty constructor is only needed by the component manager* |
| Velocity helper factory | class Javadoc "Helper factory class for creating X objects in velocity" | *only needed to instantiate the factory from Velocity* |

Two of these shapes are free in the strongest sense: the **Javadoc above the flagged line already
states the reason** (the component-manager constructors, and the deprecated methods whose
`@deprecated` tag says "this method does nothing"). That is the same lever as `S6355` — *the missing
input is already written down on the element* — applied to a different rule.

## The drop condition: "I would have to invent the reason"

Not a safety condition — every edit is comment-only — but a **truthfulness** one. Drop a site when the
emptiness might be a genuine gap rather than a decision, because a comment asserting intent that the
code does not support is worse than the open issue. 6 of 192 on this sweep:
`BaseClass#merge`, `ComputedFieldClass#displayHidden`, `UserInstanceOutputFilterStream#endUser`,
`ResourceLoader.JarFileHandle#close`, `ExtensionMojoHelper()`, `InternalWikiScannerContext#endListItem`.
These want a maintainer's answer; say so in the PR body rather than guessing.

The other 14 not shipped were dropped purely on **build ROI** — one singleton alone in its own module
would add a whole module to the reactor per issue. They are not drops; a later run picking up any of
those modules for another reason should ride them along.

## Locating and applying

Sonar flags the **method declaration line** (not the `{`, which XWiki's brace style puts on the next
line). The apply script is drift-proof without any pattern matching:

1. From the flagged line walk forward ≤4 lines to the line ending in `{`.
2. Brace-match forward to the matching `}`.
3. **Assert the interior is entirely blank** — that assertion *is* the verification that the site is
   what Sonar says it is. It held for 192 of 192.
4. Replace the interior with one comment line indented +4 from the `{`; assert ≤120 columns.

Apply bottom-up per file (descending brace index) so earlier edits do not shift later ones. Note the
interior is sometimes a blank line rather than nothing (`{\n\n}`) — replacing the whole interior
handles both and incidentally removes the stray blank line, which is why the diff shows deletions.

## Verification

Comment-only, so the build is a formality — but run it, because the batch touches oldcore. Datapoint
(cold `~/.m2`, 172 sites / 58 files / 20 modules): commons 9 modules **5:19** (470 tests) + rendering
1 module **1:02** (138) + platform 10 modules incl. oldcore and legacy-oldcore **8:52** (1412, oldcore
1149) = **~15 min for 2020 tests**, all green, `checkstyle:check` and `revapi:check` included.

**Accept pass: 172/172 on the first pass**, zero stragglers, and the remainder confirms the triage
exactly — platform 95 accepted / 17 open, commons 48 / 2, rendering 29 / 1, i.e. the OPEN count
equals the recorded drop list (6 real drops + 14 ROI-deferred singletons) in every repo.

**Review outcome: commons #1922 and rendering #411 both MERGED within ~6 h of opening**, the only
comment on the whole sweep being Vincent's LGTM — *"Going to be hard to verify every single comment. I
propose that we adopt them (a quick check shows that they look ok to me)."* — and he merged both about
a minute after I posted the distinct-sentence tables. Platform #6220 (the 95-site half) was still open,
green and unreviewed at that point. Two things this settles:

* **The "is a comment-only sweep welcome?" question is answered YES** — this was the run's one real
  bet (172 comments is a large ask, and the failure mode was a whole-batch style rejection, not a
  per-site one). It merged uncommented. Do not hedge the next comment-only rule.
* What the review actually wants is a way to **bound the check**, not fewer fixes. Answer it with the
  distinct-sentence count and the per-class table (see the learnings entry), and state the uncommented
  sites explicitly so an "adopt them all" does not sweep up the ones that need a maintainer.
