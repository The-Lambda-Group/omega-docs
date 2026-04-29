[< Home](../README.md) | [Explanation](README.md)

# Code-step pagination pattern

A **code step** is a workflow agent step with no LLM call — it reads input data deterministically, calls an adapter or performs a computation, and writes results. When the input list is too large to process in one Coordinator firing, the step must paginate: fire once per batch, pass a cursor forward, and stop when the list is exhausted.

This doc describes the canonical pattern for code-step pagination across Coordinator firings, including the revision-continuity contract that makes multi-batch drains behave as one logical snapshot. The pattern generalises to any future code step that paginates over a list (Allocate, Load Subjects, etc.).

**Canonical first occurrence:** Execute (`mab/impl/a24d83d6-workflows/83f5d9b7-design/6e122408-mab-execute/dfe47018-implementation.oql`, commit `1ac8a50`) introduced `ResolveLimit`, `BatchTakeN`, and `ComputeExecHasMore`. Sync Results (`mab/impl/a24d83d6-workflows/c6ca77cd-allocation/45a2f6d4-mab-sync-results/8826b1a4-implementation.oql`, commits `1a58c50` + `e4fac11`) cargo-culted those three helpers, added `ResolveSkip` and `ResolveOrComputeRevision`, and established the deterministic-sort requirement.

## The five load-bearing pieces

### 1. Skip / Limit / Revision input args

OnEvent reads three optional args from `Exec.args[0]`:

- `skip` (default `"0"`) — positional cursor into the sorted input list.
- `limit` (default `"1"`) — how many items to process this firing. Caller controls batch size; the implementation never hard-codes it.
- `revision` (optional) — if present, this firing is continuing an in-progress drain at the named revision; if absent, compute a fresh revision.

All three are numeric strings (OQL convention). Parse with `json-stringify` before arithmetic.

```oql
(:- (ResolveSkip Exec SkipStr)
    (get Exec "args" Args)
    (get Args 0 Input)
    (with-table-if (get Input "skip" CallerSkip)
      [Input CallerSkip SkipStr]
      (= SkipStr CallerSkip)
      (= SkipStr "0")))

(:- (ResolveLimit Exec LimitStr)
    (get Exec "args" Args)
    (get Args 0 Input)
    (with-table-if (get Input "limit" CallerLimit)
      [Input CallerLimit LimitStr]
      (= LimitStr CallerLimit)
      (= LimitStr "1")))
```

`ResolveLimit` is cargo-culted from Execute verbatim. `ResolveSkip` is the Sync-Results addition — Execute does not use positional skip because its filter (`ReadExecFilter`) is self-updating: already-written items are excluded on each firing, so the cursor advances implicitly. Positional skip is needed when the input list is fixed and the cursor must advance explicitly.

### 2. Deterministic-ordered input read

**This is the load-bearing correctness requirement.** `prop-vals-by-folder` returns rows in unstable order across separate `qo run` invocations. Without sorting, position `N` in firing 1 is not the same item as position `N` in firing 2. The Coordinator fires the step once per batch, and if the list order shifts between firings, `skip` points at the wrong item — some items are processed twice, others are missed entirely.

The failure mode observed in Sync Results V1 (pre-`e4fac11`): 32-34 unique items written out of 40 expected per drain, with the exact miss count varying by run. After applying the deterministic sort: 40/40, stable across runs.

The fix is `with-order-by` applied to the fold that builds the list, using a stable key:

```oql
(:- (ReadActiveExperiments Ctx ActiveExperiments ActiveCount {"capture" [ReadActiveTreatmentIds]})
    (call ReadActiveTreatmentIds Ctx ActiveSet)
    (get Ctx "experiments-table-id" ExperimentsTableId)
    (Qo.Public.OqlApi.Db.Prop/prop-vals-by-folder ExperimentsTableId Pid Pv)
    (get-in Pv ["prop-vals" "treatment_id"] Tid)
    (get ActiveSet Tid _)
    (get-in Pv ["prop-vals" "platform_experiment_id"] PlatExpId)
    (= Experiment {"platform_experiment_id" PlatExpId "treatment_id" Tid})
    (with-order-by [PlatExpId]           ;; ← load-bearing: stable sort before fold
      (fold [] Experiment append ActiveExperiments))
    (length ActiveExperiments ActiveCount))
```

Rules:
- The sort key must be unique per item and stable (e.g. a natural ID, not a derived or computed value).
- `with-order-by` wraps the entire `fold` — not just one term inside it.
- The sorted list is what `BatchTakeN` slices; the sort must happen before the slice, not after.

### 3. BatchTakeN helper

Slices `Limit` items from the sorted list starting at `Skip`. Handles the tail-of-list case where fewer than `Limit` items remain.

```oql
(:- (BatchTakeN List ActiveCount SkipStr LimitStr BatchedKeys)
    (json-stringify Skip SkipStr)
    (json-stringify Limit LimitStr)
    (+ Skip Limit End)
    (with-table-if (>= ActiveCount End)
      [List ActiveCount Skip Limit End BatchedKeys]
      (and (range Skip End I)
           (get List I B)
           (fold [] B append BatchedKeys))
      (and (range Skip ActiveCount I)
           (get List I B)
           (fold [] B append BatchedKeys))))
```

Then-branch: `[Skip, End)` — a full batch fits within the list. Else-branch: `[Skip, ActiveCount)` — tail slice, fewer than `Limit` items remain.

Cargo-culted from Execute's `BatchTakeN` (commit `1ac8a50`), extended with the `SkipStr` arg for positional pagination. Execute's original takes `(UnwrittenKeys UnwrittenCount LimitStr BatchedKeys)` with `range 0 Limit` (always starts at zero, because the filter produces a fresh list each firing). Sync Results adds `SkipStr` and replaces `range 0 Limit` with `range Skip End`.

### 4. Revision continuity across batches

A "drain" is a complete pass over the entire input list across one or more Coordinator firings. All batches in one drain must share one revision, so that the Platform Metrics table records one consistent snapshot at a single revision number.

The contract:
- **First firing** (no `revision` in Exec args): compute `max existing revision + 1`. If the table is empty, use `"1"`.
- **Subsequent firings** (has `revision` in Exec args): inherit the revision from the `next-args` the previous firing returned.

```oql
(:- (ResolveOrComputeRevision Ctx Exec NextRev {"capture" [ComputeNextRevision]})
    (get Ctx "platform-metrics-table-id" PlatformMetricsTableId)
    (get Exec "args" Args)
    (get Args 0 Input)
    (with-table-if (get Input "revision" InheritedRev)
      [Input InheritedRev PlatformMetricsTableId NextRev ComputeNextRevision]
      (= NextRev InheritedRev)
      (call ComputeNextRevision PlatformMetricsTableId NextRev)))
```

`ComputeNextRevision` uses `with-table-if` to handle the cold-start (empty table) case: if any rows exist, reads all revisions, sorts them, takes the max, and increments; else binds `NextRev` to `"1"`.

This helper is **specific to Sync Results** (or any step that owns a versioned output table). Execute does not need it because Execute's output table (Experiments) does not version by revision — each row is unique by `(treatment_id, draft)` and is only written once.

### 5. Result envelope with next-args

When `has-more=true`, the result must include `next-args` so the Coordinator knows how to re-fire the step:

```oql
(:- (BuildResultEnvelope HasMore NextRev ActiveCount BatchedCount WrittenCount SkipStr LimitStr Result)
    (with-table-if (= HasMore "true")
      [HasMore NextRev ActiveCount BatchedCount WrittenCount SkipStr LimitStr Result]
      (and (json-stringify Skip SkipStr)
           (+ Skip BatchedCount NextSkip)
           (string-format "%d" NextSkip NextSkipStr)
           (= NextArgs [{"step" "Sync Results" "skip" NextSkipStr "limit" LimitStr "revision" NextRev}])
           (= Result {"has-more" HasMore
                      "revision" NextRev
                      "active-count" ActiveCount
                      "batched-count" BatchedCount
                      "written-count" WrittenCount
                      "next-args" NextArgs}))
      (= Result {"has-more" HasMore
                 "revision" NextRev
                 "active-count" ActiveCount
                 "batched-count" BatchedCount
                 "written-count" WrittenCount})))
```

The `next-args` entry:
- `step` — the step name as the Coordinator knows it. Must match exactly.
- `skip` — `current skip + batch count`. Computed inline in `BuildResultEnvelope` via `(+ Skip BatchedCount NextSkip)`.
- `limit` — unchanged from this firing. The caller controls pacing.
- `revision` — the revision computed or inherited this firing. Passed through so subsequent firings inherit it.

When `has-more=false`, `next-args` is omitted entirely — the Coordinator interprets its absence as "drain complete."

## How the pieces connect: ProcessSyncResults

The orchestrating helper calls the five helpers in order:

```oql
(:- (ProcessSyncResults Ctx Exec ActiveExperiments ActiveCount Result
                        {"capture" [ResolveSkip ResolveLimit ResolveOrComputeRevision
                                    BatchTakeN CallPullResults BuildPlatformMetricsRow
                                    WriteAllPlatformMetrics ComputeSyncHasMore BuildResultEnvelope]})
    (call ResolveSkip Exec SkipStr)
    (call ResolveLimit Exec LimitStr)
    (call ResolveOrComputeRevision Ctx Exec NextRev)    ;; step 1 args + step 4 revision
    (call BatchTakeN ActiveExperiments ActiveCount SkipStr LimitStr Batch)  ;; step 3 slice
    (length Batch BatchedCount)
    ;; ... per-item work ...
    (call ComputeSyncHasMore SkipStr BatchedCount ActiveCount HasMore)
    (call BuildResultEnvelope HasMore NextRev ActiveCount BatchedCount WrittenCount SkipStr LimitStr Result))
```

OnEvent gates the entire path on `(> ActiveCount 0)`:

```oql
(:- (OnEvent Exec Result {"capture" [ResolveSyncResultsCtx ReadActiveExperiments ProcessSyncResults]})
    (call ResolveSyncResultsCtx Exec Ctx)
    (call ReadActiveExperiments Ctx ActiveExperiments ActiveCount)  ;; step 2 sorted list
    (if (> ActiveCount 0)
      (call ProcessSyncResults Ctx Exec ActiveExperiments ActiveCount Result)
      (= Result {"has-more" "false"
                 "skipped" "no-active-experiments"
                 "active-count" 0
                 "written-count" 0})))
```

The sorted list (`ReadActiveExperiments`) is built in OnEvent, before `ProcessSyncResults` is invoked. This is the correct placement: the sort runs once per firing, the slice runs inside `ProcessSyncResults`.

## Execute vs Sync Results: two flavours of the pattern

| Feature | Execute | Sync Results |
|---|---|---|
| Cursor mechanism | Self-updating filter (already-written rows excluded) | Positional `skip` |
| `ResolveSkip` | No — cursor implicit in filter | Yes — explicit positional cursor |
| `ResolveLimit` | Yes | Yes (cargo-culted) |
| `BatchTakeN` | Yes (range 0..Limit) | Yes (range Skip..Skip+Limit) |
| `ComputeHasMore` | `BatchedCount < UnwrittenCount` | `Skip + BatchedCount < ActiveCount` |
| Revision continuity | Not applicable — no versioned output table | Yes — `ResolveOrComputeRevision` |
| Deterministic sort | Not needed — filter rebuilds list each firing | Required — skip into a fixed list |

Execute's filter-based cursor is the right shape for append-only output tables where "unwritten" is derivable from the table state. Sync Results' positional-skip cursor is the right shape when the input list is stable within a drain and the cursor must be passed explicitly.

## Failure mode the deterministic sort prevents

Without `with-order-by` in `ReadActiveExperiments`, `prop-vals-by-folder` returns rows in an order that is consistent within one OQL run but differs across separate runs. The Coordinator fires the step as separate `qo run` invocations. Firing 1 builds a list with item X at position 7; firing 2 builds a list with a different item at position 7. `skip=7` in firing 2 skips the wrong item.

Observed effect (pre-`e4fac11`): 40-experiment drain produced 32-34 unique Platform Metrics rows instead of 40. The miss rate (~15-20%) corresponds to the fraction of positions that shifted between consecutive invocations.

The fix is trivially applied at the list-build site — one `with-order-by [StableKey]` wrapper. The diagnostic signal is "drain ran to completion but fewer rows than expected were written." If a pagination drain produces fewer unique rows than `active-count`, check for a missing sort first.

## Generalising to future code steps

Any future code step that paginates over a fixed input list must apply all five pieces. Checklist:

- [ ] `ResolveSkip` and `ResolveLimit` read from `Exec.args[0]` with defaults.
- [ ] Input list is sorted by a stable key before `BatchTakeN` is called (`with-order-by` in the read helper).
- [ ] `BatchTakeN` uses `range Skip End` for the full-batch case and `range Skip ActiveCount` for the tail case.
- [ ] If the step writes to a versioned output table: `ResolveOrComputeRevision` reads inherited revision from Exec args or computes max+1. Revision is passed through in `next-args`.
- [ ] `next-args` in the result envelope contains `{step, skip: next_skip, limit, revision}` (or omit `revision` if no versioned output).
- [ ] `has-more=false` result omits `next-args` entirely.

Steps with self-updating filters (Execute style) omit `ResolveSkip` but keep `ResolveLimit`, `BatchTakeN` (with `range 0 Limit`), and `ComputeHasMore`.

## Related

- [agent-step-plan-shape.md](agent-step-plan-shape.md) — flat-sequential OnEvent shape this pattern plugs into. Every helper here is one `(call ...)` line in OnEvent or `ProcessXBatch`.
- [agent-step-write-shapes.md](agent-step-write-shapes.md) — the three write-half patterns (uniqueness validation, column vs HTML-block write, Memory cold-start gate). Pagination is the read/dispatch half; write shapes are the write half.
- [reference/reducers.md § fold over zero solutions does not bind its accumulator](../reference/reducers.md#fold-over-zero-solutions-does-not-bind-its-accumulator) — why `with-table-if` is used in `BatchTakeN`'s tail branch rather than an `if` with a direct fold.
- [reference/control-flow.md § with-order-by](../reference/control-flow.md) — `with-order-by` semantics. The sort wraps the fold, not individual terms inside the fold body.
- [how-to/develop-oql.md](../how-to/develop-oql.md) — push-run-verify discipline. Every helper here should be verified individually before wiring into the orchestrating helper.
