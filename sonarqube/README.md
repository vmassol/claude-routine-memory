# SonarQube routine memory

Learnings for the "fix one (or a batch of) SonarQube issues in xwiki-platform" Claude Code routine.

## What lives here vs. in the xwiki plugin

**Rule-fix correctness moved to the OKF** in the `xwiki-dev-llm` plugin, under
[`xwiki/okf/sonarqube/`](https://github.com/xwiki/xwiki-dev-llm/tree/master/xwiki/okf/sonarqube).
That is where "what is the correct fix for `java:SXXXX`, and which sites must be dropped" now lives —
one file per rule family, loaded lazily by the `xwiki-fix-sonarqube-issue` skill. It ships to every
XWiki developer, so it holds only **durable, generic** knowledge.

**This repo keeps what the OKF must not hold:**

- **Volatile state** — how deep each rule's pool is, which modules it clusters in, observed drop
  rates, build-time datapoints. Stale within days.
- **Routine-environment facts** — this container's GitHub/`gh` restrictions, background-build and
  turn-cost discipline, branch and author overrides. True here, and *false* in a normal dev session —
  which is exactly why they must not ship to other developers.
- **Run history** — the index of issue keys already analyzed and rejected.

**Deciding where a new learning goes:** if it would still be true next year, in any session, on any
machine → it belongs in the OKF (offer it via the `xwiki-knowledge` EXTEND flow, which opens a
reviewed PR). Otherwise it belongs here.

## Layout

- **`learnings.md`** — the always-load core: read/write protocol, **Rule index**, find-phase strategy,
  batch composition, building, cost control, GitHub, process. Read this first, every run.
- **`rules/<language>-<rule>.md`** — one file per rule whose gotcha is NOT in the plugin OKF (a rule
  the OKF does not cover, or a shape it gets wrong). Read only the rows of the *Rule index* in
  `learnings.md` for the rules you commit to fixing; add a row when you add a file.
- **`pool-state.md`** — volatile per-rule state: where each rule's pool is deep or drained, module
  densities, observed drop rates. Consult it in the find phase; re-query before trusting it.
- **`dropped-issues.md`** — skip-index of SonarCloud issue keys already analyzed and rejected (with
  reason); consult it in the find phase to skip re-triaging, and add new rejected keys to it.
- **`token-cost-report.md`** — how to produce a token-cost report; loaded only when asked.

## How the routine uses it

- **Read:** load `learnings.md`, pick a rule in the find phase (using `pool-state.md` plus a fresh
  facet query), then read that rule's family file **from the OKF** — not from here.
- **Record learnings:** rule-correctness learning → OKF PR. Volatile pool observation →
  `pool-state.md`. Cross-cutting routine/build/GitHub/process fact → the matching section in
  `learnings.md`. Merge and trim in place (never append dated anecdotes), and re-synthesize only the
  file(s) you changed. Learnings are committed and pushed on `main`.

Keep everything here **generic and reusable** — techniques and observed state, not run history or PR
logs.
