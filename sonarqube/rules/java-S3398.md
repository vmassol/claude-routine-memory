# `java:S3398` — "`private` methods called only by inner classes should be moved to those classes"

Pool: platform 6, commons 4, rendering 0. Mechanical *in principle* — the method keeps its
signature, its visibility and its body, and the compiler is the whole verification — but this is the
one rule whose remediation is **metric-exposed by construction**, and it is the first rule found
where a Checkstyle metric fires on a change that adds no statement and lengthens no expression.

## The drop condition is Checkstyle `ClassFanOutComplexity`, and it fires on the MOVE itself

Moving a method into a class makes that class reference every type the method mentions. So a rule
whose whole fix is "move code into class X" raises X's fan-out by construction, and
`checkstyle:check` (which runs **after** the tests) rejects it:

```
ResourceLoader.java:[399,5] (metrics) ClassFanOutComplexity: Class Fan-Out Complexity is 22
                                      (max allowed is 20)
```

This extends the recorded metric list (`BooleanExpressionComplexity`, `ExecutableStatementCount`,
`CyclomaticComplexity`) with a fourth cap and a *new trigger shape*: not "the fix adds control flow"
but "the fix relocates code". Pre-count in the apply script where you can — but the honest cheap
move is to run the module build, because the fan-out cap counts distinct referenced types and
guessing the baseline is unreliable.

## The recovery is a PARTIAL application, not a whole-rule drop

The recorded reflex when a metric fires is "drop the site". Here that would have cost the whole
rule; instead **rank the flagged methods by how many new types each drags in and move the cheap
ones**. `ResourceLoader`'s three helpers:

| method | types it adds to `JarInfo` |
|---|---|
| `parseJarIndex` | `JarEntry`, `InputStream`, `BufferedReader`, `InputStreamReader`, `StandardCharsets`, `LinkedHashMap`, `Collections` |
| `parseClassPath` | `Manifest`, `Attributes`, `StringTokenizer`, `URI`, `URISyntaxException`, `MalformedURLException` |
| `package2url` | `HashMap`, `ArrayList`, `Map.Entry` |

Dropping the heaviest one (`parseJarIndex`) took 22 → under the cap and still shipped 2 of 3, with
0 Checkstyle violations. One extra 23-second module build settled it.

**Outcome: commons #1953 merged uncommented**, partial application and the `toPackageIndex` rename
included — so neither the "you only did two of the three" gap nor the rename drew a question, given
that the PR body stated the Checkstyle cap and the `S1845` collision as the reasons. Platform still
holds 6 untouched sites.

## Two more drop conditions, both free to check

* **The method reads OUTER instance state.** `DefaultExtensionJobHistory#save(...)` uses three
  fields of the outer instance and calls another outer private method; the move replaces plain
  calls with a chain of `Outer.this.…` accesses. Churn, not a cleanup — drop. The clean shape is a
  `private static` helper that touches nothing but its arguments.
* **A name collision with a field of the target class.** `JarInfo` already had a
  `private Map<String, URL[]> package2url` field, so moving the same-named method in is legal Java
  and an instant `java:S1845` ("methods and field names should not be the same"). Rename as part of
  the move (`package2url` → `toPackageIndex`) and update the single call site, or you trade one
  issue for another. Grep the target class's field names before moving anything.
