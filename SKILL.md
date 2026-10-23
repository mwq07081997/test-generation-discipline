---
name: test-generation-discipline
description: Generation-side discipline for writing tests during implementation — the defect-sentence gate every test must pass before it is written, risk-tiered entitlements (glue code gets zero dedicated tests by default), the behavior-manifest-before-test-code procedure, placement rules (cheapest sufficient level, one home per behavior), authoring rules that keep counts low and value high, per-task test budgets, and the CI runtime ratchet. Use whenever tests are being written or planned in any implementation, sprint task, or PR — "write tests for X", "add coverage", "implement X with tests", TDD, deciding how many or which tests a feature needs, or reviewing the tests in a diff. Trigger even when the task never questions test strategy — the manifest and budget come before the first test file. For pruning an already-bloated suite use test-suite-audit; for which suite types must exist and what gates a release use saas-qa-strategy.
---

# Test Generation Discipline

Tests are derived from risk, never from the file tree — and every test must justify its existence *before* it is written. This skill is the budget office underneath `saas-qa-strategy`: that skill says which suites exist and when they gate; this one governs how many tests get generated, which behaviors they encode, where they live, and what earns each one its place.

## Why this discipline exists

Agent-generated suites bloat for predictable reasons, and each rule below inverts one of them. Test count is the cheapest *visible* proxy for thoroughness, and the agent writing tests never pays the maintenance, CI-time, or comprehension bill — those land on future sprints. Generation walks the file tree (one `.spec` per source file, one test per function) instead of the risk surface. Task specs say "write tests" with no definition of sufficient. Coverage percentages get chased, which manufactures assertion-free tests. And nothing in the loop ever prompts deletion, so suites are append-only by default. The end state is **comprehension debt**: a suite that has outlived anyone's ability to audit it can neither be trusted nor pruned — the worst of both worlds. Preventing that state is the whole point.

## The cost model (hold this while generating)

- **A test is a purchased instrument.** Four costs: authoring (once), execution (every CI run, forever), maintenance (every refactor it touches), and comprehension (every future reader deciding "is this failure real?" / "can this be deleted?"). Comprehension is the cost agents ignore and the one that compounds worst.
- **A test's value is the expected cost of the defects *only it* catches.** A test whose every failure is accompanied by another test's failure adds ~zero detection while charging full cost. Redundancy is duplicate premium on the same policy, not safety.
- **Prefer instruments that encode invariants over instruments that encode examples.** Strength hierarchy: property/invariant > contract > table-driven example set > single example > interaction assertion. A property fails for whole *classes* of bugs; an example fails for one. Maximize information per unit of maintenance.
- **Deletion is a first-class operation** with the same dignity as authoring. (Boundary: *flaky* tests are de-flaked, never deleted for flakiness — deletion is exclusively for redundancy and purposelessness.)

## The five-question gate (run before writing any test)

1. **Can the defect sentence be completed — all three blanks?**
   > *"This test fails when ___ breaks. That matters because ___. No other instrument catches it because ___."*
   Any blank unfillable → don't write it. This one gate kills most bloat: framework tests fail blank 1, trivial tests fail blank 2, pyramid duplicates fail blank 3.
2. **Is this already statically guaranteed** (strict types, closed boundary schema, lint)? Yes → don't write it.
3. **Is this behavior already asserted at a lower level?** Yes → don't write it (unless declared T1 defense-in-depth, below).
4. **Does this surface's tier entitle this instrument?** (T4 glue asking for a dedicated unit test → no.)
5. **Can it extend an existing table, contract case, isolation case, or journey instead of creating a file?** Almost always yes → extend. New test *files* are the exception needing the strongest justification.

Only the path 1-yes, 2-no, 3-no, 4-yes writes a test — and question 5 decides its shape.

## Static guarantees are test substitutes

In a strict-typed codebase with closed schemas at boundaries (`saas-engineering-standards`: the schema *is* the egress allowlist), anything the compiler or schema already enforces is not re-tested dynamically. This deletes whole categories agents love — "returns a string", "rejects missing field", "handles null" on internally-typed functions. What remains testable at a boundary is exactly one thing: *that the schema is applied there* (one contract test) — never N tests re-proving the schema's contents.

## Risk tiers (classify before inventorying; ambiguity rounds up)

| Tier | What lives here | Test entitlement |
|---|---|---|
| **T1 — Catastrophic / irreversible** | Authn/authz, tenant isolation, money & billing math, metering, destructive data ops, migrations, crypto, privacy egress/scrubbing | Dense and *deliberately* redundant: unit + property + integration + the dedicated invariant suites (isolation, privacy-egress fuzzing). The only tier where multi-level re-assertion is permitted — declared, never accidental. |
| **T2 — Core domain logic** | The algorithms that ARE the product: decision engines, routing, ranking, pricing, state machines | Table-driven unit tests + 1 property test wherever an algebraic invariant exists. No integration-level duplication of the arithmetic. |
| **T3 — Contracts & boundaries** | Wire formats, endpoint schemas, events, error envelopes, third-party adapters | Exactly one contract test per contract (generated from the shared schemas where possible). Adapters: one integration test against a fake/sandbox. |
| **T4 — Wiring & glue** | Routing, DI, orchestrating handlers, 1:1 typed mappers, config plumbing | **Zero dedicated tests by default** — verified transitively by the thin E2E journey set plus the type system. 60–70% of agent bloat lives here. |
| **T5 — Cosmetic / inert** | Copy, logging, styling, constants | Zero tests. Lint and type-check are the instruments. |

A tier is an entitlement *ceiling*, not a quota to fill. And note the twin failure: bloat and gaps coexist — the typical bloated suite duplicates T4 trivialities while missing the ten T1 tests that matter. Tiering exposes both.

## Behavior inventory → test manifest (the central artifact)

Never write tests directly from code. Produce the manifest first — getting the *list* right is cheap; fixing a generated swarm is expensive.

**1. Enumerate behaviors, not functions.** Sources in priority order: acceptance criteria / CUJs from the task or PRD · contracts the change touches · invariants (conservation, idempotency, roundtrip, monotonicity, ordering — from the architecture doc, threat model, or the algebra itself) · failure modes (timeout, malformed input, budget exhaustion, replay, concurrent mutation — error paths are behaviors; happy-path-only is bloat *and* gap at once) · state-machine transitions, including illegal ones. A behavior is an **externally observable claim** stated as a sentence: not "`calculateOverage` works" but "usage beyond included volume is billed at the tier's overage rate, and the crossover unit is billed exactly once."

**2. Compress before counting.** Equivalence partitioning (one representative per input class) → boundary values (min, max, empty, one, many, crossover) → pairwise instead of Cartesian for multi-dimensional config → subtract everything statically guaranteed.

*Worked micro-example:* a discount function accepting `percent: 0–100`. Reflexive generation: 12 tests (0, 1, 5, 10, 25, 50, 99, 100, −1, 101, `null`, `"50"`). Compressed: `null`/`"50"` die at the boundary schema (0 tests); interior values are one class (1 row); boundaries 0 and 100 (2 rows); invalid −1 and 101 (2 rows). **One table-driven test, five rows** — same protection, one maintainable unit.

**3. Emit the manifest, then gate it on budget** before any test code exists:

```
| # | Behavior (defect sentence, blank 1)                    | Tier | Level    | Instrument              | New / Extends |
|---|--------------------------------------------------------|------|----------|-------------------------|---------------|
| 1 | Overage billed at tier rate past included volume       | T1   | unit     | pricing table (+3 rows) | Extends       |
| 2 | Invoice conservation: Σ(line items) === total, ∀ inputs | T1   | property | new property            | New           |
| 3 | POST /invoices round-trips documented schema            | T3   | contract | invoices contract       | Extends       |
| 4 | Handler wires pricing→invoice→persist                   | T4   | —        | swept by billing journey| 0 tests       |
```

The **New / Extends** column is where discipline bites: default answer is *Extends*. If the manifest exceeds the task budget (below), compress again or write one sentence justifying tier escalation. Review the manifest, not the test code, first.

## Placement: cheapest sufficient level, one home per behavior

| The claim is about… | Level |
|---|---|
| Pure logic / a computation | Unit |
| An invariant over an input space | Property |
| A boundary's documented shape | Contract |
| Composition, persistence, transactionality, auth flow | Integration (real infra emulation — never mocked-away infrastructure) |
| A user's end-to-end outcome | E2E journey |

Each behavior is asserted at exactly one level. Upper levels assert only what lower levels cannot see: integration asserts wiring, transactions, side effects — never re-asserts arithmetic already unit-tested; E2E asserts the journey outcome — never re-asserts validation messages. Sole sanctioned exception: declared T1 defense-in-depth. Detection heuristic: the same assertion text at two levels outside T1 → delete the upper one.

**E2E cardinality — journeys, not features.** An app has a handful of critical journeys (sign-up→first-value, the core workflow, a billing change, a permission denial). New features extend an existing journey; they almost never create one. An E2E suite growing linearly with feature count is the pyramid inverting — single-digit journeys per app is the healthy steady state.

## Authoring rules

1. **Table-driven by default** for any behavior family ≥ 3 cases: one test function, N rows, one maintainable unit that reads as a specification.
2. **Property tests wherever the invariant is algebraic:** roundtrip (serialize∘deserialize = id), idempotency (f∘f = f), conservation, monotonicity, oracle comparison (optimized ≡ reference). One property retires dozens of examples; privacy-egress properties ("no planted secret ever leaves") stay permanent zero-violation gates.
3. **Public seams only.** The urge to test a private helper is a design note ("this wants to be its own module"), never a test.
4. **Mock only at boundaries you don't own** — network, clock, randomness, third-party APIs. Never mock the module's own in-process collaborators out of isolation dogma. If mock setup exceeds assertion lines, the test is testing its mocks: delete it and assert one level up. Mock clocks are mandatory for time-window logic — sleeps are flakes waiting to happen.
5. **Assert outcomes, not interactions.** `toHaveBeenCalledWith(…)` is legitimate only when the call *is* the contract ("sends exactly one email", "emits exactly one billing event").
6. **Names are the audit interface:** `overage_crossover_unit_billed_exactly_once`, `rejects_expired_token_with_401`. Standing test: a stranger reading only test names must be able to state what the suite protects. Enforce this at generation and the "1000 unknown tests" state never recurs.
7. **Determinism is non-negotiable** at unit/contract level: no real time, no real network, no inter-test ordering. Flake budget: zero.
8. **One behavior per test; one test per behavior.** Five behaviors packed into one test → a failure tells you nothing; one behavior smeared across five tests → the count lies.
9. **Bug fixes always add exactly one test** — the regression test reproducing the bug first. The one context where a new test is *mandatory*: the defect sentence just wrote itself in production.

## Anti-pattern catalog (match these signatures during generation and review)

| Anti-pattern | Generation reflex | Detection signature | Replacement |
|---|---|---|---|
| Framework test | "Test everything" | Asserts library behavior: the ORM saves, the validator rejects, the router routes | Delete; trust the dependency, contract-test *your configuration of it* once |
| Change detector | Asserting implementation | Body restates the impl's steps; breaks on refactor with behavior unchanged | Assert the observable outcome only |
| Mock theater | Isolation dogma | > 50% of lines are mock setup; assertions target mocks | Real collaborators, or move up a level |
| Permutation storm | Exhaustiveness reflex | N×M generated cases with no distinct failure mode per case | Equivalence classes + boundaries + pairwise |
| Snapshot landfill | Cheap assertions | Large snapshots historically updated with `-u`, unreviewed | Targeted assertions on the fields carrying the contract |
| Pyramid duplicate | Undeclared belt-and-suspenders | Same assertion text at ≥ 2 levels outside T1 | Keep the lowest level |
| Test-per-function reflex | File-tree generation | `.spec` mirrors source 1:1 including trivial exports | Behavior manifest as the generator |
| Assertion-free / trivial | Coverage chasing | No assert; or asserts constants, types, constructor-sets-field | Delete — that coverage was a lie |
| Private-method test | White-box reflex | Rewires visibility / reaches into internals | Public seam, or extract the module |
| Happy-path-only | Demo bias | Zero failure-mode tests on a T1–T3 surface | Failure modes are inventoried behaviors — the *gap* twin of bloat |

## Budgets, ratchet, and suite metrics

- **Suite-level CI budget as a hard-fail gate** (house standard): e.g., unit ≤ 2 min, full per-PR ≤ 10 min — numbers are policy knobs; the gate's existence is not. **The ratchet:** adding tests that would breach the budget requires merging or deleting equivalent runtime in the same PR. Suites stop being append-only the day this lands.
- **Metrics that matter:** suite runtime trend · flake rate (zero at unit/contract) · % of tests whose name yields a defect sentence · sampled **mutation score on T1/T2 directories** (quarterly) · defect-escape correlation (did production bugs land in tested or untested behaviors?).
- **Banned as targets:** raw test count · raw line coverage. Coverage survives only as a **floor alarm** on core packages (~80%, per `saas-qa-strategy`) — a drop is a signal to investigate; a rise is never a goal.
- The asymmetry to remember: 300 tests at 90% mutation score on T1/T2 is strictly stronger than 1000 at 60%. Count measures cost; mutation score measures protection.

## Wiring into sprint tasks and review

**Every sprint task carries a test-budget line** in manifest vocabulary, e.g.:

> `Tests: extends pricing.spec table (+3 rows: zero-boundary, included-volume max, overage crossover); +1 property (invoice conservation); isolation suite +1 case (new query path — same PR); 0 new files.`

Default budgets by task class (deviating needs one written sentence):

| Task class | Default budget |
|---|---|
| Pure glue / wiring (T4) | 0 new tests — swept transitively |
| Domain logic new/changed (T2) | 1 table-driven unit (+rows); +1 property iff an algebraic invariant exists |
| Wire contract new/changed (T3) | 1 contract test per contract |
| New store or query path | + isolation cases in the same PR |
| New tenant-facing endpoint | Contract + isolation + extend the relevant journey |
| Bug fix | Exactly 1 regression test (mandatory) |
| Genuinely new user journey | 1 E2E journey (rare) |
| Behavior-preserving refactor | 0 new tests; any test that *breaks* is flagged as a change-detector for quarantine |

TDD is fully compatible — the manifest *is* the red list; TDD without a manifest is exactly how permutation storms happen.

**Two symmetric adversarial review questions** for every PR's tests (this is `execution-loop` Phase 7 aimed at the tests themselves):

1. *"Which of these tests would you delete first — and what evidence lets it survive anyway?"* No defensible answer → delete before merge.
2. *"Which behavior in this diff has NO test — and why is that fine?"* The answer must cite a tier, a static guarantee, or an existing home. This catches the gap twin of bloat.

**Cadence:** a 15-minute per-sprint suite health check — runtime delta, flake list, and test-count delta vs. behavior delta (count growing faster than behaviors is the early bloat signal).

## Ecosystem boundaries

`saas-qa-strategy` is the constitution (which suites, what gates); nothing here overrides its matrix — isolation stays a release-blocker forever, the privacy property suite still gates data-capture changes. `test-suite-audit` is the remediation twin for suites that already bloated; it consumes this skill's tiers, gate, and anti-pattern signatures as classification criteria. `saas-engineering-standards` supplies the enforcement idiom reused here (hard-fail CI budgets, schema-as-allowlist, same-PR coupling). In sprint plans (`sprint-planning`, `saas-pack-execution`), the test-budget line is a required task field and Final Output Validation inherits "no test without a defect sentence" alongside "tests pass".
