# Graph (boru) — status

**This library is DESIGNED, NOT IMPLEMENTED.** `graph.aql` exports an
empty `Graph` namespace. There is no word to call, and no idiom to copy.

If you are trying to use a shortest-path or A\* API from boru: it does
not exist yet. Do not invent one, and do not write code against the
names below expecting it to run.

Re-verified against boru main @ `64c5ab2` (2026-10-01).

## The design

`DESIGN.md` at the repository root is the substance: the planned roster
(`Graph.dijkstra`, `Graph.astar`, `Graph.floyd`, `Graph.bellman-ford`),
the weighted-adjacency representation, the priority-queue primitive that
must be built first because boru has none, and the boru runtime
constraints that force each shape.

## Rules already fixed

**Calling convention** — forward args, receiver (the graph) LAST:
`Graph.verb …args g`. Piping (`g Graph.verb …args`) binds identically;
only receiver-first-all-forward misbinds. Import with
`import "./graph.aql"`, resolved against the importing file's own
directory (`"../graph.aql"` from `test/`).

**Heuristics may call their own module's helpers** — boru resolves a
function value's free words in the module that *defined* it (fixed
upstream in boru `7e98aeb`, re-verified on main). Pass a heuristic as a
value, `h/v`: a bare name holding a function calls wherever it appears
(`/r` was renamed `/v`).
