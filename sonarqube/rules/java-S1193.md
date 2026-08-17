# java:S1193 — `catch (Exception e) { if (e instanceof X) … }` → an explicit `catch (X e)`

Behaviour-preserving and compiles as long as the `try` body really can throw `X` — check the called
method's `throws` clause first (a `catch` for an unthrowable checked exception does not compile).

**The trap is `java:S1192`, not the compiler.** The `throw` at the end of the old single `catch` has
to be repeated in both new branches, so a file with two such sites goes from 2 to 4 copies of the
same message literal and the fix *creates* a duplicated-literal issue (threshold 3). Extract the
message into a `private static final String` constant in the same edit.

Drop when splitting would leave an **empty** catch block (the old body only acted on one of the
multi-catch types) — that trades the issue for a fresh `java:S108`.
