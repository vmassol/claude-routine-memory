# java:S1872 — classes compared by name

Split verdict; read what the comparison is actually for.

- **Clean:** comparing a `Class` against a *fixed known* class, e.g.
  `"void".equals(method.getReturnType().getName())` → `method.getReturnType() != void.class`.
  Exactly equivalent (only `void.class` is named `"void"`) and no import needed.
  **Write it with `!=`/`==`, not `!equals(…)`** — that was asked for in review (tmortagne on #6187, which then merged) and
  it is the better form: `Class` instances are canonical, and `void.class` is VM-wide for a primitive,
  so the reference comparison is exact as well as clearer than a negated `equals`. It creates no new
  issue: `java:S1698` (`==`/`!=` with an overridden `equals`) is **not active** in XWiki's Java profile
  and could not fire anyway since `Class` does not override `equals`, and the active `java:S4973`
  covers only Strings and boxed types.
- **Drop:** `SomeEvent.class.getName().equals(event.getClass().getName())`. Sonar suggests
  `isAssignableFrom`/`instanceof`, which additionally matches **subclasses** — a behaviour change.
  Name comparison across what is likely a class-loader boundary (XWiki component/extension
  class-loaders) is a deliberate idiom, not a defect.
