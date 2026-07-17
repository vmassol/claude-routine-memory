# SonarQube routine memory

Learnings for the "fix one (or a batch of) SonarQube issues in xwiki-platform" Claude Code routine.
Structured for **lazy loading** so a run reads only what it needs instead of one large file.

## Layout

- **`learnings.md`** — the always-load core (~4k tokens): read/write protocol, the **Rule index**
  (rule → detail file), the denylist, and cross-cutting mechanics (find phase, batch mode, building,
  cost control, GitHub, process). Read this first, every run.
- **`rules/*.md`** — per-rule-family detail (fixes, gotchas, drop conditions). One file per family;
  the Rule index in `learnings.md` maps each rule key to its file.
- **`token-cost-report.md`** — how to produce a token-cost report; loaded only when asked.

## How the routine uses it

- **Read:** load `learnings.md`, pick a rule in the find phase from the Rule index, then read ONLY
  that rule's `rules/*.md` file (and any others you commit to fixing this run). Skip the rest — that
  is the whole point of the split.
- **Record learnings:** write each learning into the **smallest file that owns it** — a rule-specific
  gotcha → that rule's file under `rules/`; a cross-cutting technique or build/PR/process fact → the
  matching section in `learnings.md`. Merge and trim in place (never append dated anecdotes), and
  re-synthesize only the file(s) you changed. Adding a new rule's file? Add its row to the Rule index.
  Learnings are committed and pushed on `main`.

Keep everything **generic and reusable** — techniques and gotchas, not run history or PR logs.
