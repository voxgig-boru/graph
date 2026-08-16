# DESIGN — shortest paths and heuristic search

> **Status: design only. Nothing is implemented.** `graph.aql` exports an
> empty `Graph` namespace; the test suites are green placeholders. This
> document is the argued plan: what the library is, what it must not
> become, and — for the parts where boru's runtime decides the answer —
> what shape the code will have to take. Sketches below are *shapes*, not
> implementations.

This repository was instantiated from the `bloom-filter` template. The
scaffolding (CI, hook, skill, plugin, docs skeleton, test naming) has
been renamed for `graph`; none of the bloom filter's logic, tests or
documentation was carried across, because a textual rename of those
would have produced a library that describes a probabilistic set under
graph names.

---

## 0. The decisions, at a glance

| # | Decision | Ruling | Why (one line) |
|---|----------|--------|----------------|
| 1 | Scope | **Shortest paths and heuristic search** — not a general graph library | One family, one engine; matches the one-library-per-family ecosystem |
| 2 | Engine | **One priority-driven search**, with the priority function pluggable | Dijkstra, A\* and greedy best-first differ *only* in the priority; three words, one loop |
| 3 | Keystone property | **A\* with `h ≡ 0` must equal Dijkstra** | The family's own degeneracy check — the analogue of `sort`'s cross-agreement |
| 4 | Enabling primitive | **A binary heap must be built first** — boru has no priority queue | Verified absent from `REFERENCE.md`; the whole family rests on it |
| 5 | Node identity | **Strings** | boru map keys are String/Atom only — a runtime constraint, not a choice |
| 6 | Direction | `{a: {b: 3}}` means **an edge a→b of weight 3** | Same ruling, same reason as the sibling `sort` library's topological sort |
| 7 | Unreachable | **Absent from the result**, no infinity sentinel | boru integer overflow is a hard error, so sentinel arithmetic is a trap |
| 8 | Heuristics | **May call their own module's helpers** (since boru 7e98aeb) | A function value resolves its free words in the module that *defined* it |
| 9 | Dependency direction | **graph → sort**, never the reverse | Kruskal consumes a sort; nothing in `sort` needs a graph |
| 10 | Out of scope | flow, matching, colouring, layout, incremental replanning | Each is a different family with a different engine |

---

## 1. The family

Every algorithm here is the same loop — *take the most promising frontier
node, relax its edges, repeat* — differing only in what "most promising"
means:

| Algorithm | Priority | Gives you |
|---|---|---|
| **BFS** | insertion order (unit weights) | fewest hops |
| **Dijkstra** (uniform-cost) | `g(n)` — cost from the start | optimal paths, non-negative weights |
| **A\*** | `g(n) + h(n)` — cost so far **+ estimate to goal** | optimal point-to-point, far fewer expansions |
| **Greedy best-first** | `h(n)` alone | fast, **not** optimal |
| **Bellman–Ford** | relax all edges V−1 times | negative weights; detects negative cycles |
| **Floyd–Warshall** | O(V³) dynamic programming | all pairs, dense graphs, tiny implementation |
| **DAG shortest path** | relax in topological order | linear time when the graph is acyclic |

**A\* is the centre of the family, and it is Dijkstra plus a heuristic.**
`h(n)` estimates the remaining cost to the goal; adding it to the cost
already paid steers the search toward the target instead of expanding
uniformly in all directions. Set `h ≡ 0` and A\* *is* Dijkstra; set
`g ≡ 0` and it is greedy best-first. That is not a curiosity — it is the
design (§3) and the keystone test (§9).

The wider orbit, none of it proposed for v1: **IDA\*** and **beam search**
bound memory (beam sacrifices optimality), **bidirectional search** meets
in the middle, **Jump Point Search** exploits uniform grids, **D\* Lite**
and **LPA\*** re-plan incrementally when edge costs change under a moving
agent, and **contraction hierarchies** / **ALT landmarks** preprocess
road-network-scale graphs for repeated queries.

## 2. Why this is a separate library

The sibling `sort` library's design doc draws the line this repository
sits on the other side of: *does the algorithm return an ordering of the
whole input, or a route through part of it?*

Topological sort returns a permutation of every node, and on an edgeless
graph degenerates exactly to `Sort.merge` — which is why it lives in
`sort`. A shortest-path search returns a **route**, usually after
visiting a fraction of the graph, and has no ordering degeneracy at all.
It also needs two concepts a comparator library does not have — **edge
weights** and a **heuristic** — plus a priority queue and a graph
representation. That is a graph library, and it belongs in its own repo,
which is the established ecosystem pattern (`bloom-filter`, `decision`,
`sort`, `stats`, `template`, `trie`).

**The seam with `sort` runs one way: graph → sort, never the reverse.**
Kruskal's minimum spanning tree is the clean example — sort the edges by
weight, then union-find — a legitimate consumer of `Sort`. Where this
library eventually wants a topological order (for DAG shortest path), it
should **duplicate the ~40-line Kahn loop rather than take a cross-module
dependency**: these libraries are vendored by copy, and a cross-module
function-value dependency is exactly what §7's free-word rule punishes.

## 3. The proposed surface

Three words share one engine. All follow the ecosystem convention —
**arguments forward, receiver (the graph) LAST**.

| Call | Returns | Notes |
|------|---------|-------|
| `Graph.dijkstra from g` | `Map` | Single-source: `{dist: {node: cost}, prev: {node: node}}`. Unreachable nodes are **absent**, not infinite. |
| `Graph.astar h to from g` | `Map` | `{cost, path}` — point-to-point with heuristic `h`. `path` is a `List` of nodes, start first. |
| `Graph.path-to node result` | `List` | Reconstruct a route from a `dijkstra` result's `prev` map. |
| `Graph.floyd g` | `Map` | All-pairs. Independent implementation, so it doubles as the test oracle (§9). |
| `Graph.bellman-ford from g` | `Map` | Negative weights; raises `negative_cycle` with the offending cycle. |

`Graph.dijkstra` and `Graph.astar` are the same loop with a different
priority. Whether they are two exported words over one private engine or
one word with an optional heuristic is §12's first open question; two
words is the current leaning, because boru's multi-signature dispatch is
a known sharp edge and two names document themselves.

```boru
def g {a: {b: 4, c: 2}, b: {d: 5}, c: {b: 1, d: 8}, d: {}}

Graph.dijkstra "a" g
# => {dist: {a: 0, c: 2, b: 3, d: 8}, prev: {c: "a", b: "c", d: "b"}}
```

## 4. Roster, and what ships when

| Phase | Words | Why here |
|---|---|---|
| **0** | the private binary heap (§6) | Nothing in the family works without it |
| **1** | `dijkstra`, `path-to` | The base case; A\* is this loop plus one term |
| **2** | `astar` | The headline; needs only the heuristic contract on top of phase 1 |
| **3** | `floyd` | Independent algorithm ⇒ the cross-check oracle the property suite needs |
| **4** | `bellman-ford` | Negative weights and negative-cycle reporting |
| **later** | `bfs` (unit-weight shortcut), DAG shortest path, MST | Each is a small specialisation once the engine exists |

## 5. Representation

**Weighted adjacency map: `{node: {neighbour: weight}}`.** Nested maps
rather than an edge list, for three reasons: it is what the relaxation
loop consumes; isolated nodes are representable (`{a: {}}`), which an
edge list silently drops; and weight lookup is a direct `get` rather
than a scan.

**Direction: `{a: {b: 3}}` is an edge a→b.** This is the same ruling the
sibling `sort` design makes for topological sort, and for the same
reason — the reverse reading produces a plausible, silently wrong answer
rather than an error. Undirected graphs are expressed by declaring both
directions; a `Graph.undirected` helper that symmetrises a map is a
candidate for phase 1.

**Nodes are Strings.** boru map keys must be `String` or `Atom` (they are
the same slot), so a node cannot be an arbitrary value. Callers with
richer nodes key by id and re-project afterwards. This is the same
constraint the `sort` toposort design records, and it is a hard runtime
property, not a preference.

**Weights are Integers or Floats, and must be non-negative for
`dijkstra`/`astar`.** A negative edge breaks Dijkstra's central
invariant (that a node's distance is final when it is dequeued);
`bellman-ford` is the word that handles them. Reject negatives loudly
with `bad_input` rather than returning a quietly wrong answer.

**Unreachable means absent.** Do not synthesise an infinity sentinel:
boru raises a hard error on 63-bit integer overflow rather than wrapping,
so a large sentinel that gets added to an edge weight is a crash waiting
for a big graph. An absent key in the `dist` map is unambiguous, and
`has` is total.

## 6. The enabling primitive: boru has no priority queue

Verified: there is no heap, no priority queue, and no pathfinding code
anywhere in boru (`REFERENCE.md`, `design/`, `kg/`). A\* is *defined* by
its priority queue, so **phase 0 is building one**, and the constraints
below decide its shape:

- **A binary heap over a `flex` list**, with explicit index arithmetic
  (parent `(i sub 1) div 2`, children `2i+1` / `2i+2`) and hand-written
  sift-up / sift-down.
- **Do not use `pop` or `shift`.** Both return **two** values — the
  container *and* the element — which leaks a residual the bytecode
  compiler refuses (`residual shape beyond Stage 1`). `shift` is also
  O(n), measured quadratic in aggregate. Read the root by index, move
  the last element into slot 0, and shrink.
- **Lazy deletion over decrease-key.** A decrease-key operation needs a
  node→heap-index map maintained through every sift. The standard
  alternative is to push a duplicate entry at the better priority and
  skip any entry already finalised when it surfaces. It costs O(E log E)
  instead of O(E log V) and removes an entire class of bug; every
  practical Dijkstra implementation does this.

The `sort` library's `DESIGN.md` proposes a private heap of its own for
`Sort.least`/`greatest`. **These cannot be shared** — see §7: a heap
parameterised by a comparator must live in the module that drives it.
Each library carries its own; that is the boru-imposed cost of the
single-module rule, and it is cheaper than the alternative.

## 7. The heuristic contract — and the sharp edge

**Admissible** — `h(n)` never *overestimates* the true remaining cost.
This is what makes A\* optimal. `h ≡ 0` is trivially admissible (and
gives Dijkstra); straight-line distance is admissible for travel on a
map because no route is shorter than the direct line.

**Consistent** (monotone) — `h(n) ≤ weight(n, n') + h(n')` for every
edge. Stronger than admissible, and it is what makes the *closed set*
safe: with a consistent heuristic a node's cost is final the first time
it is expanded. With a merely-admissible heuristic, a node can be reached
later by a cheaper route and must be re-opened.

**Ruling: require consistency, document the difference, and do not
implement re-opening in v1.** Nearly every real heuristic is consistent,
re-opening doubles the state machine, and silently returning a
sub-optimal path is the worst possible failure. If the docs say
"consistent" and a caller supplies something weaker, the result may be
sub-optimal — say so plainly rather than paying for the general case.

### The sharp edge — RESOLVED UPSTREAM, 2026-08-15

> This section previously called the free-word scope rule "the single
> biggest API risk in the library" and required heuristics to be
> self-contained. **That is no longer true, and the constraint it imposed
> on this library's API is lifted.** The original text is kept below the
> line because it explains why the surrounding design looks the way it
> does.

boru now resolves a function value's free words in the module that
**defined** it, on both engines, whether the value is applied, bound to a
name, or passed through a native callback
(`design/FUNCTION-VALUE-SCOPE.0.md` §11 rule 1, merged as boru
`7e98aeb`). A user-supplied heuristic may therefore call the user's own
helper words freely — `Graph.astar` invoking it does not change where
those words resolve.

Two consequences for this design:

- **Heuristics need not be self-contained.** Open question 3 below leaned
  toward `h(n, goal)` over a closure "for exactly that reason"; that
  reason is gone, so the question reopens on its merits alone.
- **The one-file constraint is lifted too.** This library is no longer
  barred from sharing `sort`'s planned heap by the scope rule. Whether to
  share it is now an ordinary dependency-direction question (row 9),
  not a language limitation.

What remains true, and is a *different* fault, is the combinator failure
in `sort`: a bare namespace comparator fed to a combinator raises
`uncalled_function`. Under boru's ADR-016 a bare name is a call and `/r`
is how you ask for the value, so the fix there is `Sort.by-number/r`, not
a scope change.

---

*Original text, superseded:*

**A function value's free words resolve in the module that *runs* it,
not the module that defines it.** This is the rule that forces the
sibling `sort` library to be a single file… Applied here: a
user-supplied heuristic that calls the user's *own* helper word will fail
with `undefined_word` when `Graph.astar` invokes it. **Heuristics must be
self-contained** — parameters and builtins only.

## 8. boru implementation constraints

Carried from measurements taken while designing the sibling library's
topological sort; all apply verbatim here.

- **Accumulators must be `flex`.** Immutable `Map` accumulation is
  quadratic — 211 ms at n=1,000 rising to 11,403 ms at n=8,000, against
  26→52 ms for the mutable equivalent (**219× at n=8,000**). The `dist`
  map, `prev` map, closed set and the heap are all `flex`; convert back
  with `node` at the boundary so the returned value is plain and
  immutable.
- **`Store` is not an option** — `make Store` is a coded refusal.
  There is no Set type either; a Map to `true` is the idiom, and `has`
  is total.
- **No `while`.** The idiom is `for N [body]` with an early `break`.
  Inside a `fn`, a `for` body must net **zero** values while an `each`
  body must yield **exactly one** — opposite rules, both hard errors.
- **Recursion will not carry a traversal.** Tail-call optimisation
  requires that nothing pends below the call and that the callee re-binds
  every name the caller's frame holds; a search carrying a visited map
  through body-local `def`s fails both. The engine is a loop.
- **`and` / `or` do not short-circuit.** Guard with nested `if`.
- **Reserved names — `node` is one of them**, which stings in a graph
  library. Also reserved: `keys`, `vals`, `has`, `depth`, `stack`,
  `walk`, `find`, `list`, `min`, `max`, `range`. Verified free and
  idiomatic here: `nd`, `adj`, `edges`, `nodes`, `frontier`, `visited`,
  `seen`, `cur`, `out`, `dist`, `prev`, `cost`. Capitalised names bind
  *types*, so all state is lowercase.
- **Integer overflow is a hard error at 63 bits**, not a wrap — the
  reason §5 rejects an infinity sentinel.

## 9. Testing

**The keystone is the degeneracy check: `Graph.astar` with `h ≡ 0` must
return the same cost as `Graph.dijkstra`.** That is this family's
analogue of the `sort` library's "every algorithm agrees with
`Sort.merge`", and it is the single most valuable property here.

Note the same non-uniqueness lesson the toposort design ran into:
**equal-cost paths are not unique**, so cross-algorithm agreement must be
asserted on **cost**, not on the path. Assert path *validity* separately.

| Property | Catches |
|---|---|
| `astar h≡0` cost == `dijkstra` cost | the whole engine (the keystone) |
| Every algorithm agrees with `floyd` on all-pairs costs (small graphs) | independent-implementation cross-check |
| Path validity: consecutive pairs are real edges, and the weights sum to the reported cost | reconstruction bugs in `prev`/`path-to` |
| An admissible heuristic never changes the optimal *cost* (only the number of expansions) | a broken heuristic contract |
| A\* expands no more nodes than Dijkstra on the same query | the heuristic doing nothing (silent perf regression) |
| Unreachable target ⇒ absent, not an error, not a wrong number | §5's absence convention |
| Negative edge to `dijkstra` ⇒ raises `bad_input` | §5's rejection rule |
| Negative cycle ⇒ `bellman-ford` raises `negative_cycle` with a witness | detection, not just non-termination |
| Single node, self-loop, disconnected components, zero-weight edges | the usual edges |

**Oracle:** generate a random weighted graph, run `floyd` for ground
truth, and check every other word against it. For heuristics, a random
point-set with straight-line distance is admissible by construction.

## 10. What landing any of it requires

The scaffolding is in place and CI is live, so a first implemented word
must also update: `api.json` (`word_specs`), `AGENTS.md` (call shapes and
"Common mistakes"), **both byte-identical copies** of
`.claude/skills/graph-aql/SKILL.md` and
`plugins/graph-aql/skills/graph-aql/SKILL.md` (a CI job diffs them),
`docs/reference.md` and the other three Diátaxis pages, the suite bodies
in `test/`, `test/divergence/run.sh`'s `SUITES` list, and `README.md`.

## 11. Deferred and declined

**Deferred** — natural once the engine exists: `bfs` as a unit-weight
shortcut; DAG shortest path (relax in topological order — duplicate the
Kahn loop per §2); minimum spanning tree (Prim, and Kruskal as the
`Sort` consumer); bidirectional search; `Graph.undirected`.

**Declined**, with the reason:

- **Incremental replanning** (D\*, D\* Lite, LPA\*) — needs a mutable
  graph and a persistent search state handle across calls. A different
  shape of library; revisit only with a real robotics consumer.
- **Contraction hierarchies / ALT landmarks** — preprocessing schemes
  that pay off at road-network scale, which is far outside what a boru
  library will be handed.
- **Jump Point Search** — a grid specialisation; it assumes uniform-cost
  8-connected grids, not the general weighted graph this library takes.
- **Max-flow, matching, colouring, layout** — different families,
  different engines. Not pathfinding.
- **Beam search** — non-optimal by construction; belongs with heuristic
  optimisation, not shortest paths.

## 12. Open questions

1. **One word or two for Dijkstra/A\*?** `Graph.astar h to from g` with
   `h ≡ 0` *is* Dijkstra, so a single word could serve both. (Leaning:
   two words — boru's multi-signature dispatch is a known sharp edge, and
   `Graph.dijkstra` documents itself.)
2. **Should `dijkstra` take an optional target** and early-exit when it
   is finalised? It is a large win on big graphs and a trivial change.
   (Leaning: yes, as a separate word rather than an option — the
   early-exit result has a *different shape*, since the other distances
   are then only provisional.)
3. **Does the heuristic take one node or two?** `h(n)` closing over the
   goal is the textbook form. This previously leaned toward `h(n, goal)`
   because §7's free-word rule made closures fragile — **that rule is
   fixed (boru `7e98aeb`), so the leaning is withdrawn** and the question
   is open on its merits: the textbook single-argument form against the
   explicitness of passing the goal. Note closures capture by *snapshot*,
   not by cell, so a captured goal is fixed at construction — which is
   what you want here.
4. **Is `Graph.floyd` public or test-only?** It is the natural oracle,
   but O(V³) invites misuse on graphs where it will never finish.
   (Leaning: public, documented with a size warning.)
