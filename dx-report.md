# DX report — boru runtime gotchas

Project-specific boru gotchas hit while building **this** library.

**Empty so far** — nothing is implemented yet, so nothing has been hit.

The constraints already *known* to apply here (measured while designing
the sibling `sort` library's topological sort, and carried into this
design rather than rediscovered) are recorded in
[DESIGN.md §8](DESIGN.md#8-boru-implementation-constraints): quadratic
immutable-Map accumulation, the two-value `pop`/`shift` trap, the absent
priority queue, the opposite `for`/`each` body-arity rules, non
short-circuiting `and`/`or`, hard integer overflow, and the reserved-name
list (`node` among them).

Record here what *this* library hits that the design did not anticipate.
