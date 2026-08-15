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
  `.claude/hooks/session-start.sh`) builds `boru` in remote sessions so a
  fresh session can run the suites.
- The whole library is one file, `graph.aql`, exporting the single
  `Graph` namespace. Keep it that way: boru resolves a function value's
  free words in the module that *runs* it, so a heuristic or comparator
  that reaches a private helper must share the module with the algorithm
  that invokes it (DESIGN.md §7).
- Tests live in `test/`, named `graph_<unit|prop>_<test|spec>.aql` plus a
  `graph_smoke_test.aql`: `_test` = imperative (`Test.test` /
  `Test.check-prop`), `_spec` = declarative spec; `unit` = example-based,
  `prop` = property-based. Each assertion-bearing suite ends by asserting
  `Test.fail-count` is `0` and prints `all green`.
- The planned keystone property is the degeneracy check: **A\* with
  `h ≡ 0` must agree with Dijkstra on cost.** Note it must be asserted on
  *cost*, not path — equal-cost paths are not unique (DESIGN.md §9).
- This repo was instantiated from the `bloom-filter` template; the
  scaffolding is renamed but no bloom logic or documentation was carried
  across.
