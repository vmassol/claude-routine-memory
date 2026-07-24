# java:S1643 — "Use a StringBuilder instead"

A `String` local is accumulated with `+=` inside a loop. Fix: declare a `StringBuilder` before the
loop (seed it with any pre-loop value, e.g. `new StringBuilder(initial)`), replace each `x += expr`
with `builder.append(expr)`, and assign `x = builder.toString()` after the loop (or use the builder
directly if `x` is only consumed afterwards). `StringBuilder` is in `java.lang` — no import. Small
pool (~9), mostly oldcore. More surgery than S1640/S1604, so verify each carefully.

## Gotchas — sites that are NOT safe (drop)

- **Prepend, not tail-append**: `x = expr + x` (e.g. zero-padding `s = "0" + s`, or building a
  reversed/hierarchical string `number = seg + "." + number`) is order-sensitive; a plain `.append`
  reverses the output. DROP. Real drops: `PasswordClass` (zero-pad prepend), `TOCGenerator` (segment
  prepend).
- **Loop condition/body reads the intermediate string**: if the loop tests `x.length()` (or otherwise
  reads `x`) between concatenations, the running `String` value is load-bearing — DROP.
- Only convert genuine tail-append accumulation where `x` is not read between appends.

## Gotcha — StringBuilder passed to a mock-verified call breaks the test

If the accumulated value flows into a method call that a **test verifies by argument equality**
(Mockito `verify(logger).warn(msg, value)`), passing the `StringBuilder` **directly** fails the test:
`StringBuilder` does not override `equals`, so it never equals the expected `String` — even though at
runtime SLF4J/`String.format` call `toString()` and the real output is identical. Fix: pass
`builder.toString()` at the use site (keep the builder for accumulation). Real case:
`WorkspacesMigration.restoreDeletedDocuments` → `logger.warn(…, documentsToRestoreAsString.toString())`.
General rule: when the built value is handed to any API a test asserts on, materialise it with
`.toString()`.

## Verification

Must run module tests — the mock-equality break above is invisible to the compiler. No JaCoCo/Revapi
impact expected, but a removed branch could shift coverage; `-Plegacy,quality` catches it.
