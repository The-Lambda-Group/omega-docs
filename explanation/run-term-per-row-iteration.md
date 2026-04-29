[< Home](../README.md) | [Explanation](README.md)

# `run-term` Per-Row in Multi-Solution Iteration

## The Rule

**When a captured helper is invoked per-row inside a multi-solution iteration at scale (K ≥ ~30 rows), use `run-term`, not `call`.**

This is a carve-out from the default "use `call` for in-memory captured helpers" rule documented in [run-term does not work on in-memory clauses](../gotchas/run-term-in-memory-clause.md). The general rule still holds for one-solution-in / one-solution-out helpers, list-mapping helpers, and validation helpers. The exception applies specifically to the **per-row iteration context**: a helper that is called once per solution row in a multi-row solution table, where the solution table has on the order of 30+ rows.

## What "per-row iteration" means

A per-row iteration is any pattern where the solution table has K rows (one per arm, one per treatment, one per subject, etc.) and a helper is invoked once per row, with per-row input bindings:

```oql
;; K rows in scope (one per arm)
(get Arms _ ArmData)
(get ArmData "arm_id" ArmId)
(get ArmData "budget" Budget)

;; Helper invoked once per row — THIS is the per-row iteration
(run-term BuildAllocationsRow Ctx ArmId Budget Row)   ;; CORRECT at K=40

;; BAD at scale — hangs at K~40
;; (call BuildAllocationsRow Ctx ArmId Budget Row)
```

The distinguishing feature: the helper invocation sits inside the row-level solution (after terms that produce K rows from an array or datastore scan), and the result of the helper contributes per-row output.

## Empirical evidence

Two independent cases from MAB Allocate V1 (commits at `~/Development/wttc/mab/impl/a24d83d6-workflows/c6ca77cd-allocation/05a97b4e-mab-allocate/992aad8a-implementation.oql`, 2026-04-30):

### Case 1 — `BuildAllocationsRow` (per-arm allocation row construction)

`ProcessWithBudget` iterates over K arms and calls `BuildAllocationsRow` once per arm to construct an allocations row envelope.

- Original: `(call BuildAllocationsRow Ctx ArmId Budget Row)`
- At K=40: 3 socket hangs in one debug session, no useful error — engine stalled
- Fix: `(run-term BuildAllocationsRow Ctx ArmId Budget Row)`
- After fix: ~3–5s wall-clock at K=40

### Case 2 — `CountSaForArm` (per-arm subject-assignment count)

`CheckLoadCompleteForRev` iterates over K arms and calls `CountSaForArm` once per arm to count subject-assignment rows for that arm.

- Original: `(call CountSaForArm Ctx ArmId Count)`
- At K=40: engine hung, required a database restart to clear state
- Fix: `(run-term CountSaForArm Ctx ArmId Count)`
- After fix: <5s wall-clock at K=40

### Scaling baseline

- Strategist's `WriteStrategy` uses `run-term` per-row at K=27 (commit `b779ca8`) — proven working. That implementation already used `run-term`, so the call-vs-run-term question was never surfaced there.
- Sync Results Phase 1 uses `if (call ...)` for a single-call gated helper (not per-row iteration over K rows) — works fine at K=40. Shape distinction: a single conditional `call` is not an iteration.

**Empirical threshold:** `call` in per-row iteration hangs somewhere between K=27 and K=40. The exact threshold is unknown. At K=40 it reliably hangs; prior to K=27 it has not been tested in the iteration-specific shape.

## Hypothesis on root cause (not empirically verified)

`(call CapturedHelper ...)` per-row in a multi-solution iteration may produce solution-table residue that scales non-linearly. Each `call` invocation shares solution scope with the enclosing iteration — the K-row solution table is visible during each call. As K grows, the accumulated per-call scope overhead may compound.

`(run-term CapturedHelper ...)` runs the helper as a separate execution against a fresh solution. This isolates per-row state from the enclosing K-row iteration scope. The engine handles the partitioned executions efficiently, each seeing only its own row's bindings.

This is the same mechanism that makes `run-term` fast in the partition-before-fold pattern (see [Reducers — Performance](../reference/reducers.md#performance-partitions-and-run-term)) — the key insight is that "fresh solution per invocation" is beneficial in both the partition-fold case and the per-row-iteration case.

## The exception and the default rule

This carve-out does **not** change the default rule. The [run-term does not work on in-memory clauses](../gotchas/run-term-in-memory-clause.md) gotcha doc's core guidance still applies:

| Context | Use |
|---------|-----|
| One-solution-in / one-solution-out helper (standard `call` shape) | `call` |
| List-mapping, validation, transformation helper (not per-row iteration) | `call` |
| Partition-before-fold scaling (replacing `with-group-by`) | `run-term` |
| **Per-row iteration at scale (K ≥ ~30 rows)** | **`run-term`** |

The rule is: if your helper is invoked once per row in a multi-row solution table AND K might reach ~30+, use `run-term`.

If you are not in the per-row-iteration shape and not replacing a `with-group-by`, use `call`.

## What to do when you hit a hang

If a multi-row iteration hangs (socket hang, `ECONNRESET`, no-return with engine stall):

1. Identify the per-row helper invocation(s) in the hanging clause.
2. Switch `(call Helper ...)` → `(run-term Helper ...)` for each per-row invocation.
3. Verify that the helper's logic is correct for stateless per-invocation execution (no side-effects that depend on cross-row shared state — each `run-term` call is independent).
4. If the engine is stalled after a hang: restart the omega-db pod. See the [502 restart note](../../omega-knowledge-base/README.md) in the KB root.

## Cargo-cult sources

Both canonical implementations are in the MAB Allocate step:

- `ProcessWithBudget` — `run-term BuildAllocationsRow` per arm
- `CheckLoadCompleteForRev` — `run-term CountSaForArm` per arm

Location: `~/Development/wttc/mab/impl/a24d83d6-workflows/c6ca77cd-allocation/05a97b4e-mab-allocate/992aad8a-implementation.oql` (commit `5341753`).

## Related

- [run-term does not work on in-memory clauses](../gotchas/run-term-in-memory-clause.md) — the base rule. This doc is a carve-out from that rule's "use `call` by default" guidance.
- [Reducers — Performance: partitions and `run-term`](../reference/reducers.md#performance-partitions-and-run-term) — the partition-before-fold pattern (the other valid `run-term` shape for captured helpers). The per-row isolation mechanism is the same.
- [`call` vs `run-term` solution scoping](https://github.com/The-Lambda-Group/query-omega-oql/blob/master/docs/explanation/call-vs-run-term-on-captured-helpers.md) — deeper explanation of when each term is structurally required (per-row freshness, partition-before-fold).
- [Agent-step write shapes](agent-step-write-shapes.md) — Pattern 2 covers `run-term` per-row for HTML block writes (`add-or-get-by-name` / `set-content`). Same `run-term` per-row shape, different motivation (Block-UUID isolation rather than scale).
- MAB session handoff: `~/Development/omega/omega-knowledge-base/projects/mab/reference/session-handoff.md` — Allocate V1 section documents both cases in context.
