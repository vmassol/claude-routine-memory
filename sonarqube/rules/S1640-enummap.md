# java:S1640 — "Convert this Map to an EnumMap"

Replace a `HashMap` (or other `Map`) whose **key type is an enum** with `EnumMap`: keep the declared
type (`Map<TheEnum, V>`) unchanged and change only the constructor `new HashMap<>()` →
`new EnumMap<>(TheEnum.class)`. Add `import java.util.EnumMap;`; drop `import java.util.HashMap;`
only if `HashMap` (word boundary) is absent afterwards. To find `TheEnum`, read the map's declared
generic key type at the flagged line or its declaration. Nested enums need qualification
(`Outer.ParametersKey.class`) or the outer class import.

A deep, mostly-safe pool (~35+ project-wide): concentrated in notification/ratings/security modules
and their **test** files (test-assertion maps are the safest sites). Good backbone rule when the tiny
pure-mechanical allowlist is drained.

## Gotchas — when it is NOT a safe conversion (drop these)

- **`EnumMap` rejects `null` keys** — `put(null, …)` throws NPE, whereas `HashMap` allowed it. If the
  map is ever populated with a `null` key it CANNOT become an `EnumMap`. This is a **runtime** break,
  invisible to the compiler; a `<clinit>`/init-path `put(null,…)` fails only when the class loads (an
  `ExceptionInInitializerError` cascading to `NoClassDefFoundError` in unrelated tests). Real case:
  `Right.java` (`security-authorization-api`) uses a `null` `EntityType` key as an "all entity types"
  wildcard via `enableFor(null,…)` — reverted. Before converting a **main-code** map, check every
  `.put(` / initializer for a possibly-null key (a literal `null`, or a key variable from a nullable
  source). A grep for a literal `.put(null` is necessary but not sufficient (keys can be null vars).
- **Iteration order changes**: `EnumMap` iterates in enum **ordinal** order, `HashMap` in hash order.
  Irrelevant for lookup-by-key and test-assertion maps (the common case), but drop any site whose
  output/behaviour depends on hash/insertion iteration order.
- Skip if the constructor already takes an initial-capacity or a source-map argument, or the key is
  not actually an enum.

## Verification

Purely mechanical but MUST run the module tests under `-Plegacy,quality` — the null-key break only
shows at runtime. `EnumMap` conversions do not affect JaCoCo/Revapi.
