# AGENTS.md — using the `Graph` library

> **There is no API yet.** This library is **designed, not implemented**:
> `graph.aql` exports an empty `Graph` namespace. If you are an agent
> trying to call it, stop — there is nothing to call.
>
> The design lives in **[DESIGN.md](DESIGN.md)**.

## What it will be

Shortest-path and heuristic search over a weighted directed graph:
`Graph.dijkstra`, `Graph.astar`, `Graph.floyd`, `Graph.bellman-ford`.

## Two things that are already fixed, and will not change

**Calling convention — forward args, receiver (the graph) LAST:**
`Graph.verb …args g`. The piping form `g Graph.verb …args end` binds
identically; only receiver-first-all-forward (`Graph.verb g …args`)
misbinds. This matches every sibling library in the ecosystem.

**Heuristics must be self-contained.** boru resolves a function value's
free words in the module that *runs* it, so a heuristic that calls your
own helper word will fail with `undefined_word` when `Graph.astar`
invokes it. Use parameters and builtins only. This is the library's
single sharpest edge — see DESIGN.md §7.

## Where to look next

- `DESIGN.md` — the roster, the representation, the constraints, the
  open questions.
- `docs/` — Diátaxis documentation, to be written against the real API.
