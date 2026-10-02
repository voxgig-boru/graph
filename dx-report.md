# DX report — boru runtime gotchas

Project-specific boru gotchas hit while building **this** library.

**Nothing is implemented yet**, so the library itself has hit nothing.
The constraints already *known* to apply here (measured while designing
the sibling `sort` library's topological sort, and carried into this
design rather than rediscovered) are recorded in
[DESIGN.md §8](DESIGN.md#8-boru-implementation-constraints): quadratic
immutable-Map accumulation, the two-value `pop`/`shift` trap, the absent
priority queue, the `for`/`each` body-arity rules, non short-circuiting
`and`/`or`, hard integer overflow, and the reserved-name list (`node`
among them). Each was re-run on boru main; see the dated notes there.

Record here what *this* library hits that the design did not anticipate.

---

## Migration to boru main @ 64c5ab2 (2026-10-01)

The scaffolding was last run against boru `6185620` (2026-07-21), 1,587
upstream commits earlier. Since 2026-09-19 (`ba64e111c`) boru has **one
execution path**: `boru X` runs a static pre-flight check, then compiles
the program to bytecode and runs it on the VM, or fails with
`[boru/compile_failed] … this is a compiler defect`. There is no
interpreter fallback; `--compile` / `--force-compile` / `--no-compile`
are usage errors (`flag provided but not defined`, exit 1) and the
`BORU_COMPILE` / `BORU_FORCE_COMPILE` / `BORU_NO_COMPILE` env vars are
ignored. "A suite runs" now means "a suite fully compiles".

**Result:** all five suites compile, run green and `boru check` with
0 errors (one advisory `module_body_executed_in_check` info each);
`graph.aql` checks clean. `test/divergence/run.sh` passes.

### Breaking changes hit

**M1 — relative imports anchor on the importing file.** `import
"./graph.aql"` now resolves against the directory of the file that
contains the `import`, for `boru X` as well as `boru check X`. The suites
in `test/` imported `"./graph.aql"`, i.e. a non-existent
`test/graph.aql`, and every suite failed. Fix: `import "../graph.aql"`.
(CLI.md still says `boru run` resolves against the working directory;
that sentence is stale.) The failure did not name the missing file —
see **D1**.

**M2 — postfix `print` chains reorder.** The suites' summary
(`"---" print` / `"fail count: " print Test.fail-count end print` / …)
printed `fail count:`, `0`, `0`, `all green` on main — `print` collects
forward. Fix: one value per statement — `print ("---")`,
``print (`fail count: ${(Test.fail-count)}`)``,
`Assert.equal 0 (Test.fail-count)` (expected first), `print ("all green")`.
A failing `Test.test` still makes the suite exit 1 without `all green`.

**M3 — the free-word scope fix holds; `AGENTS.md` / `SKILL.md` /
`api.json` were stale.** They still said "heuristics must be
self-contained", although DESIGN.md §7 already recorded the upstream fix
(boru `7e98aeb`, 2026-08-15). Re-verified on main: module A's function
value calling A's private `secret`, passed as `A.h/v` into module B
(which has its own `secret`) and applied there — directly and through a
native `each` callback — runs A's `secret`; a caller-defined heuristic
calling the caller's own helper resolves inside B too. The docs now say
heuristics may call their own module's helpers, passed as `h/v`.

**M4 — `/r` is `/v`.** The value modifier was renamed on 2026-08-19
(ADR-011; `ref` became `valof`) and `/r` is now an `undefined_word`; a
bare name holding a function **calls** wherever it appears, so a
heuristic or comparator handed to a word must be written `h/v`.

**M5 — tooling names.** The CLI is `boru` (build `cmd/go` → `./boru`);
native modules are `boru:*`; `prep`/`pack` write `.boru/` (was `.aql/`,
now in `.gitignore`). The harness, hook, `.gitignore` and docs were
updated; the CI workflow needed no edit (it runs `test/divergence/run.sh`
and the suites directly).

### Workarounds applied

- **Library before `boru:test`.** Each assertion-bearing suite imports
  `../graph.aql` before `boru:test`, the order the sibling bloom-filter
  and stats suites need for the type-ID collision (**D2**). `Graph` will
  return plain Maps, not a class, so this library is unlikely to hit it
  at all; and the order is not a general fix (whether a class collides
  depends on the type ID it lands on).

### Open upstream defects (minimal repros)

None blocks a suite here; they matter to whoever implements the design.

- **D1 — a missing import is misreported as a compiler defect**
  (unrecorded in NUR.md). `import "./no-such-file.boru"` → `boru X`:
  `[boru/compile_failed]: … residual value of unknown provenance — this
  is a compiler defect`, `source position unknown`; `boru check`:
  `0 error(s)`. Same for a missing `boru:` module or bare module.
  Expected: an `import_error` naming the path.

- **D2 — `boru:test` type-ID collision.** `boru:test` builds its
  sub-registry without `Types.AdoptSeqFrom(parent.Types)`, so a class a
  program defines can share an ID with one of its record types:
  ```boru
  # lib.boru
  def Box class { v: 0 }
  def mk fn [ [n:Integer] [Box] [ make Box {v: n} ] ]
  export "L" { mk: mk/v }
  # main.boru
  import "./lib.boru"
  import "boru:test"
  print (L.mk 1)
  ```
  → `type_error: mk: return value 1: expected Box, got Box` (either
  import order; fine without `boru:test`).

- **D3 — keeping the element of a `flex` list's `pop`/`shift` fails to
  compile.** This is the heap's natural "take the root" shape, which
  DESIGN §6 already avoids (read by index, `pop q drop` to shrink — both
  compile):
  ```boru
  def take-last fn [[q:FlexList] [Any] [ pop q swap drop ]]
  print (take-last (flex [5 6 7]))
  ```
  → `fn take-last: body result is a fn-value lead a later dispatch
  collected past (NUR121)`; `pop q var [[a b] a]` in the body → `body
  leaves extra values (Stage 3 lowers in-order results)`; at top level
  `def x (pop q swap drop)` → `stack discipline: result operand of drop
  is not on top`, and `print (pop [5 6 7] var [[a b] a])` →
  `dynamic-scope def `a` of unknown provenance`. All pass `boru check`.

- **D4 — a cyclic `flex` structure crashes the process.**
  `def a (flex {v: 1})`, `def _1 (a set self a)`,
  `print (do [ a deq a ] error [ "caught" ])` → Go `fatal error: stack
  overflow`, exit 2, uncatchable (also `print`/`StructUtil.jsonify` of a
  cycle). Relevant if a graph is ever represented with node links rather
  than the adjacency map of DESIGN §5.

### Other notes from re-verifying DESIGN.md

- **The step budget bounds a search.** A run is capped at 10,000,000
  evaluation steps by default (`[boru/evaluation_limit]`; raise with
  `boru -options steps:N`). A Floyd–Warshall-shaped V³ relaxation over a
  flat flex matrix finishes V=50 and trips the budget by V=60.
- `while [cond] [body]` exists (since 2026-08-21) and compiles in a fn.
- `min` / `max` are no longer reserved binding names; `base` is, and
  `take` cannot be a fn name.
- `and` / `or` evaluate both operand expressions (they only *select* an
  operand); nest `if` to guard.
- Integer overflow raises `[boru/integer_overflow]` outside the signed
  64-bit range.
