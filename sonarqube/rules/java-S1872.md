# java:S1872 — classes compared by name

Split verdict; read what the comparison is actually for.

- **Clean:** comparing a `Class` against a *fixed known* class, e.g.
  `"void".equals(method.getReturnType().getName())` → `void.class.equals(method.getReturnType())`.
  Exactly equivalent (only `void.class` is named `"void"`) and no import needed.
- **Drop:** `SomeEvent.class.getName().equals(event.getClass().getName())`. Sonar suggests
  `isAssignableFrom`/`instanceof`, which additionally matches **subclasses** — a behaviour change.
  Name comparison across what is likely a class-loader boundary (XWiki component/extension
  class-loaders) is a deliberate idiom, not a defect.
