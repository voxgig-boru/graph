# CLAUDE.md

This repository is the `Graph` shortest-path and heuristic-search
library, written in boru.

## Status — designed, not implemented

`graph.aql` exports an **empty** `Graph` namespace. The five test suites
exist to hold the naming convention and are green placeholders. There is
no API to call yet.

**@DESIGN.md is the substance of this repository right now.** It records
the algorithm roster (Dijkstra, A\*, Floyd–Warshall, Bellman–Ford), the
representation ruling, the priority-queue primitive that must be built
first because boru has none, the heuristic contract, and the boru
runtime constraints that force each shape. Read it before adding
anything to `graph.aql`.

## Using the library

See @AGENTS.md — currently a status stub, to be written against the real
API as words land.

## Working on this repository

- A SessionStart hook (`.claude/settings.json` →
  `.claude/hooks/session-start.sh`) builds `boru` from boru-lang/boru
  **main** HEAD in remote sessions so a fresh session can run the suites.
  Locally, build it from source
  (`cd <boru>/cmd/go && go build -o ~/.local/bin/boru ./boru`).
- **One execution path.** `boru X` runs a static pre-flight check, then
  compiles to bytecode and runs on the VM, or fails with
  `[boru/compile_failed] … compiler defect`; there is no interpreter
  fallback, and `--compile` / `--force-compile` / `--no-compile` are
  retired (usage errors). Never use `-no-check` to get green.
- The whole library is one file, `graph.aql`, exporting the single
  `Graph` namespace. That is now a packaging choice, not a language
  constraint: boru resolves a function value's free words in the module
  that *defined* it (fixed upstream in boru `7e98aeb`, re-verified on
  main), so a heuristic may call its own module's helpers (DESIGN.md §7).
  Pass function values with `/v` (`h/v`) — a bare name holding a function
  calls.
- Relative imports resolve against the **importing file's own
  directory**: the suites in `test/` import `"../graph.aql"` (and import
  it before `boru:test` — see `dx-report.md`).
- Tests live in `test/`, named `graph_<unit|prop>_<test|spec>.aql` plus a
  `graph_smoke_test.aql`: `_test` = imperative (`Test.test` /
  `Test.check-prop`), `_spec` = declarative spec; `unit` = example-based,
  `prop` = property-based. Each assertion-bearing suite ends with
  `Assert.equal 0 (Test.fail-count)` and prints `all green`, one
  `print (…)` per value (postfix print chains reorder).
- `test/divergence/run.sh` is the gate (CI calls it): every suite exits 0
  under `boru X` and prints `all green` where it asserts, and `boru check`
  reports 0 errors on every suite and on `graph.aql`. `BORU=/path/to/boru`
  reuses a binary. Verified against boru main @ `64c5ab2` (2026-10-01);
  boru gotchas and the migration notes are in `dx-report.md`.
- The planned keystone property is the degeneracy check: **A\* with
  `h ≡ 0` must agree with Dijkstra on cost.** Note it must be asserted on
  *cost*, not path — equal-cost paths are not unique (DESIGN.md §9).
- This repo was instantiated from the `bloom-filter` template; the
  scaffolding is renamed but no bloom logic or documentation was carried
  across.
