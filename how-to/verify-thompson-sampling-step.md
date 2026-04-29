[< How-to](README.md)

# Verify a Thompson-sampling allocation step

> **Confidentiality: PUBLIC.**

A cargo-cult-able verification pattern for any OQL code step that runs Thompson sampling to distribute a budget across arms. Covers four invariants: variance is real (sampling is not deterministic), no arm starvation, budget tracking is correct, and selection pressure flows from prior strength.

Extracted from MAB Allocate V1 Phase 2 verification (2026-04-30, KB commit `39e2df4`). Apply to any step whose implementation calls something like `sample-beta` per arm and then allocates based on sampled values.

---

## The pattern: N-firing drop-between distribution sanity

Run the step multiple times against a clean slate, record per-arm counts, then check the four invariants below.

### Procedure (5 firings)

1. **Capture initial state.** Verify the expected starting row count in the output table (e.g. `qo describe "...Allocations"` or query for row count). Record the revision and arm set.

2. **For each firing 1–5:**
   a. Drop the existing output rows for the test revision:
      ```
      qo drop ".../<output-table>"
      ```
   b. Fire the step (Coordinator `OnEvent` equivalent, e.g. via `qo run` with the step's coordinator path and empty args).
   c. Read the new output rows; record the per-arm count (the allocation count column).

3. **Build a table** with columns `[arm_id, alpha, beta, R1, R2, R3, R4, R5, mean, stdev]`.
   - `alpha` / `beta` come from your platform metrics or prior state at test time.
   - `Rn` is the count assigned to that arm in firing n.
   - `mean` = arithmetic mean across the 5 runs.
   - `stdev` = sample standard deviation across the 5 runs.

4. **Apply the four checks** (see below).

---

## Four verification dimensions

### 1. No arm starvation (required)

**Check:** No arm has `count = 0` in ANY run.

**Pass criterion:** Min count observed across all `(K arms × N runs)` pairs is ≥ 1. For Budget=400 and K=40 you should not see any zero; a minimum of ~5 is healthy.

**What it catches:** A degenerate sampler that never selects certain arms, or a budget exhaustion bug that leaves trailing arms with nothing.

**Concrete reference (V1, K=40, Budget=400, N=5):** Min count = 5. Zero starvation across all 200 arm-run pairs.

---

### 2. Variance is real (required)

**Check:** Per-arm stdev across the N runs is > 0 for each arm.

**Pass criterion:** Mean per-arm stdev > 0 across all arms. In practice, expect stdev ≥ 1 for a healthy Thompson sampler with a reasonable budget. A stdev of 0 for every arm means sampling is deterministic (a bug).

**What it catches:** A frozen sampler (e.g. the random draw is re-seeded identically each run, or the sample is being cached instead of redrawn).

**Concrete reference (V1, Beta(1,1) uniform priors):** Mean per-arm stdev = 1.94. Max per-arm stdev = 3.19. 17 of 40 arms had stdev ≥ 2.0. Individual arms swung 6–10 count points across runs.

---

### 3. Budget tracking (required)

**Check:** The sum of all arm counts per run (Σ count) is within tolerance of the configured Budget.

**Pass criterion:** `|Σ count - Budget| ≤ Budget × 0.10` (10% tolerance). A tighter bound (±5%) is better if your implementation uses exact allocation.

**What it catches:** Under-allocation (budget not fully used) or over-allocation (budget exceeded), both indicating arithmetic errors in the budget-tracking path.

**Concrete reference (V1, Budget=400):** Per-run sums: R1=398, R2=406, R3=380, R4=393, R5=404. All within ±20 of budget (spec allowed ±40). Observed tolerance ≤ 5%.

---

### 4. Selection pressure from prior strength (conditional)

**Check:** Arms with higher α (stronger positive prior) receive higher mean count than arms with lower α, measured via Spearman rank correlation between mean count and α.

**Pass criterion:** A positive Spearman rank correlation between per-arm mean count and per-arm α value, significantly above 0 (not necessarily perfect — Thompson sampling has noise, but direction should hold across enough arms).

**When to apply:** ONLY when α/β values differ across arms. If all arms share the same prior (e.g. Beta(1,1) cold-start), this check is N/A — all arms are identically distributed and Thompson sampling is expected to produce roughly uniform counts with per-run noise, which is correct behavior.

**What it catches:** A uniform-lock bug where all arms receive equal counts regardless of posterior strength, indicating the sampler is ignoring Beta parameters.

**Related gap:** `mab-allocate-bandit-verification-data-gating` — tracking the gating requirement for this check. The check cannot be run until real reward data differentiates the Beta posteriors (i.e., after subjects have completed reply cycles and Platform Metrics rows have been updated with real reward/max_reward values).

---

## Example output table

From MAB Allocate V1 Phase 2 (all arms Beta(1,1), Budget=400):

| exp_id  | alpha | beta | R1 | R2 | R3 | R4 | R5 | mean | stdev |
|---------|-------|------|----|----|----|----|----|------|-------|
| 3250508 |   1   |   1  |  5 | 13 |  8 |  7 | 11 |  8.8 |  3.19 |
| 3250511 |   1   |   1  | 10 | 13 |  7 |  8 |  8 |  9.2 |  2.39 |
| 3250512 |   1   |   1  |  9 |  9 | 10 |  7 |  9 |  8.8 |  1.10 |
| 3250513 |   1   |   1  | 12 |  7 | 11 |  9 | 12 | 10.2 |  2.17 |
| 3250514 |   1   |   1  | 12 |  9 |  9 | 12 |  9 | 10.2 |  1.64 |
| 3250515 |   1   |   1  |  9 | 11 |  8 | 13 | 14 | 11.0 |  2.55 |
| ...     |  ...  |  ... | .. | .. | .. | .. | .. |  ... |   ... |

Per-run sums: R1=398, R2=406, R3=380, R4=393, R5=404.

Checks applied:
- Starvation: min count = 5. PASS.
- Variance: mean stdev = 1.94. PASS.
- Budget: all runs within ±20 of 400. PASS.
- Selection pressure: N/A (all priors uniform).

---

## When to run this

Run after implementing a new Thompson-sampling code step, before shipping to production, whenever you have:
- A `sample-beta` (or equivalent) per-arm draw in the implementation.
- A budget-allocation loop that converts sampled values to integer counts.
- A write step that records per-arm allocations.

The 5-firing drop-between procedure takes roughly 10–15 minutes to run manually and produces a data table you can commit to the session handoff for traceability.

---

## Cargo-cult sources

- MAB session-handoff (KB `projects/mab/reference/session-handoff.md`) § "Allocate Phase 2 — Distribution Sanity" at commit `39e2df4` — the original data table and analysis.
- MAB Allocate V1 implementation at `~/Development/wttc/mab/impl/a24d83d6-workflows/c6ca77cd-allocation/05a97b4e-mab-allocate/992aad8a-implementation.oql` (commit `5341753`) — the step under test.
- MAB Allocate design spec at `~/Development/wttc/mab/docs/superpowers/specs/2026-04-29-allocate-design.md` § Verification case 6 — the spec definition that motivated this pattern.
