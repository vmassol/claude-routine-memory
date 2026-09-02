# `java:S2198` — "Remove this comparison; it will always return false/true"

One rendering site, and it is the cautionary tale for the *general* technique it prompted.

## The technique it validated

When a dead-code drop is justified by **"but the code documents intent"**, the fix is usually to move
the intent into a comment rather than to leave the issue open. That reasoning is right and is recorded
in `learnings.md`; it re-opened this pair after a previous run had written it off.

## Why it did not pay HERE

`WikiPageUtil.isValidXmlNameStartChar(char ch, boolean)` ends its range table with
`(ch >= 0x10000 && ch <= 0xEFFFF)`, unreachable because a `char` stops at `0xFFFF`. Deleting it plus a
note saying the range needs an `int` shipped as rendering #425, green and uncommented.

**But the dead line was a symptom of a real defect, and the real fix RESTORES it.** Vincent took the
PR's own open question to the forum
([t/18812](https://forum.xwiki.org/t/change-wikipageutil-isvalidxmlnamestartchar-isvalidxmlnamechar-to-take-an-int-code-point/18812/),
accepted answer: go through deprecation + legacy) and the agreed change is to take an `int` code
point — at which point that exact comparison becomes the row that implements the supplementary half
of the XML `NameStartChar` production, and `S2198` stops being raised at all. So the cleanup deletes
the line the fix needs back and adds a comment the fix obsoletes: **net churn, and the Sonar issue
closes either way.**

## The generic lesson

Before "documenting the intent" of a provably-dead comparison, ask **why** it is dead. Two cases:

* **The intent is unreachable by design** (a range a narrower type cannot hold *and* nothing wants it
  to) → the comment is the right fix.
* **The intent is unreachable because of a bug in the signature/type** → the dead line is *evidence of
  the defect*, the fix is the signature, and a cleanup that deletes the evidence is worse than the
  open issue. Symptoms to check: does a neighbouring method already take the wider type
  (`isValidXmlChar(int ch)` did), and does deleting the line lose a row of a spec the code
  transcribes?

The cheap test is one sentence: *"if someone fixed this properly, would my deleted line come back?"*
If yes, report the defect and leave the issue open.

## Outcome

#425 was **closed unmerged** and superseded by **xwiki-rendering#426 (XRENDERING-814,
"Support supplementary characters when validating XML names")**, which does the whole thing properly:
`int` code-point overloads, `isValidXmlName` walking the name by code point, and the `char` signatures
moved to a new `xwiki-rendering-legacy-wikimodel` module. So the routine's cleanup was not merely
churn — it *surfaced* a 20-year-old defect (`isValidXmlName("\uD840\uDC00foo")` returned `false` for a
valid XML name) and the maintainer turned it into a real fix within hours. **That is the best possible
outcome for a site of this shape, and it is worth more than the two issues would have been**: report
the defect, do not paper over it.

Note the sequence that produced it: the PR body stated the open question ("the real fix is an `int`
code point, which is an API change"), the maintainer took *that* to the forum, the community chose
deprecation + legacy over a compat break, and the fix followed. Stating the open question in the body
is what did the work.

## Keys

`AZ_01MtBbzuOmnNi3w77`, `AZ_01MtBbzuOmnNi3w78` — both on `WikiPageUtil` L310, same line. Expect them
to close as FIXED off the `int` change, not off a cleanup.
