# AGENTS.md — using the `Graph` library

> **There is no API yet.** This library is **designed, not implemented**:
> `graph.aql` exports an empty `Graph` namespace. If you are an agent
> trying to call it, stop — there is nothing to call.
>
> The design lives in **[DESIGN.md](DESIGN.md)**. Everything below was
> re-verified against boru main @ `64c5ab2` (2026-10-01).

## What it will be

Shortest-path and heuristic search over a weighted directed graph:
`Graph.dijkstra`, `Graph.astar`, `Graph.floyd`, `Graph.bellman-ford`.

## Already fixed, and will not change

**Calling convention — forward args, receiver (the graph) LAST:**
`Graph.verb …args g`. The piping form `g Graph.verb …args` binds
identically; only receiver-first-all-forward (`Graph.verb g …args`)
misbinds. On boru main `boru check` rejects that order when the swapped
arguments have different types (`uncalled_function: … matched no
signature`, which also blocks `boru X`); two arguments of the same type
still bind silently in signature order. This matches every sibling
library in the ecosystem.

**Import it relative to the importing file.** `import "./graph.aql"`
resolves against the directory of the file that contains the `import`
(for `boru X` and `boru check X` alike), not the working directory — a
suite in `test/` writes `import "../graph.aql"`.

**Heuristics may call their own module's helpers.** boru resolves a
function value's free words in the module that *defined* it (boru
`7e98aeb`, 2026-08-15; re-verified on main, including through a native
`each` callback), so a heuristic that calls your own helper word keeps
working when `Graph.astar` invokes it. Pass it as a **value**: `h/v` — a
bare name holding a function *calls* wherever it appears, and the old
`/r` spelling is now an `undefined_word`. Close over the goal through a
parameter (`make-h goal` returning `(n:Integer => [goal sub n])`) rather
than reading a top-level name, which resolves late (DESIGN.md §7, §12 Q3).

## Where to look next

- `DESIGN.md` — the roster, the representation, the constraints, the
  open questions (dated notes record what changed on boru main).
- `dx-report.md` — boru gotchas and the migration to boru main.
- `docs/` — Diátaxis documentation, to be written against the real API.
