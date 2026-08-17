# javascript:S6353 — concise character class (`[^0-9]` → `\D`, `[^a-zA-Z0-9_]` → `\W`)

Safe, but **prove the equivalence rather than assuming it** — it depends on the regex flags. Without
the `u`/`v` flag JS defines `\D` as exactly `[^0-9]` and `\W` as exactly `[^A-Za-z0-9_]`. A 10-second
`node` loop over `U+0000`–`U+1117F` comparing `c.replace(/[^0-9]/g,'')` with `c.replace(/\D/g,'')`
settles it and is worth quoting in the PR body.

Same safety class as `java:S6397`/`S6353`: a one-token edit inside a regex literal, no dataflow.
