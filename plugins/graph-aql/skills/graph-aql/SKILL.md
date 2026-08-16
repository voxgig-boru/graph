# Graph (boru) — status

**This library is DESIGNED, NOT IMPLEMENTED.** `graph.aql` exports an
empty `Graph` namespace. There is no word to call, and no idiom to copy.

If you are trying to use a shortest-path or A\* API from boru: it does
not exist yet. Do not invent one, and do not write code against the
names below expecting it to run.

## The design

`DESIGN.md` at the repository root is the substance: the planned roster
(`Graph.dijkstra`, `Graph.astar`, `Graph.floyd`, `Graph.bellman-ford`),
the weighted-adjacency representation, the priority-queue primitive that
must be built first because boru has none, and the boru runtime
constraints that force each shape.

## Two rules already fixed

**Calling convention** — forward args, receiver (the graph) LAST:
`Graph.verb …args g`. Piping (`g Graph.verb …args end`) binds
identically; only receiver-first-all-forward misbinds.

**Heuristics must be self-contained** — parameters and builtins only.
boru resolves a function value's free words in the module that *runs* it,
so a heuristic calling your own helper raises `undefined_word` when
`Graph.astar` invokes it. This is the library's sharpest edge.
