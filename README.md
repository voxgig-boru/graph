# graph

Shortest-path and heuristic-search algorithms — Dijkstra, A\*, and the
rest of the family — for [boru](https://github.com/boru-lang/boru).

> **Status: designed, not implemented.** `graph.aql` exports an empty
> `Graph` namespace and the test suites are green placeholders. The
> argued design — the roster, the representation, the priority-queue
> primitive the family rests on, and the boru constraints that force each
> shape — is in **[DESIGN.md](DESIGN.md)**. Read that first; there is no
> working API yet.

## What it will be

One priority-driven search engine, with the priority function pluggable:

| Word | Priority | Gives you |
|------|----------|-----------|
| `Graph.dijkstra` | `g(n)` — cost from the start | optimal paths, non-negative weights |
| `Graph.astar` | `g(n) + h(n)` — cost so far **+ estimate to goal** | optimal point-to-point, far fewer expansions |
| `Graph.floyd` | O(V³) dynamic programming | all pairs; also the test oracle |
| `Graph.bellman-ford` | relax all edges V−1 times | negative weights; detects negative cycles |

A\* is Dijkstra plus a heuristic: set `h ≡ 0` and it *is* Dijkstra, which
is both the design and the keystone test.

## Project layout

```
graph.aql                 the library (the Graph namespace) — currently a stub
DESIGN.md                 the argued plan for what goes in it
AGENTS.md                 agent guide: how to call this library correctly
test/graph_*.aql          the five suites (naming convention held, bodies empty)
test/divergence/          three-surface guard (interpreter · check · byte compiler)
docs/                     Diátaxis documentation
```

## Running it

Build the `boru` interpreter, then run any suite — see
[How-to → Install and run](docs/how-to.md#install-and-run-aql):

```bash
boru test/graph_smoke_test.aql
```

## License

See [LICENSE](LICENSE).
