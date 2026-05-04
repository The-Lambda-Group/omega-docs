[< Gotchas](README.md)

# sec-index and row-index do not reliably filter rows — use prop-vals-by-folder scan + filter instead

> **Confidentiality: PUBLIC.** All content in this file is public platform documentation.

`Qo.Public.OqlApi.Db.Prop/sec-index` and `Qo.Public.OqlApi.Db.Prop/row-index` return **all rows** in the table rather than the single matching row, regardless of whether the table has a composite PK or a single-column PK. Empirically observed on:

- **Copy** table — composite PK `[treatment_id, draft]` (MAB Critic, MAB Copywriter)
- **Critiques** table — composite PK `[treatment_id, draft]` (MAB Critic)
- **Campaign Analytics** table — single-column PK `id` (MAB SmartLead adapter, commit `8507532`)

The symptom is a silent full-scan: instead of binding `Pid` to the single matching row-id, the term binds it to every row-id in the folder, and downstream reads or writes fan out across all rows.

## Distinct failure mode: inline key literals with symbols

Not every `row-index` fan-out means the table has duplicate rows or that the index lookup itself is broken. A separate OQL rule applies when the key argument is written inline as a list literal containing symbols:

```oql
;; WRONG — Tid does not resolve inside the inline literal
(Qo.Public.OqlApi.Db.Prop/row-index StrategiesDbId [Tid] RowPageId)
```

In that form, OQL reads `[Tid]` as a literal list before row bindings are applied, so `Tid` stays symbolic instead of becoming the actual key value. The practical symptom can look like the broader `row-index` gotcha: the call fans out or otherwise fails to target the single intended row.

The workaround is to bind the key vector first, then pass that bound symbol to `row-index`:

```oql
(= RowKey [Tid])
(Qo.Public.OqlApi.Db.Prop/row-index StrategiesDbId RowKey RowPageId)
```

This same bind-first rule applies to composite keys:

```oql
(= RowKey [TreatmentId Draft])
(Qo.Public.OqlApi.Db.Prop/row-index CopyDbId RowKey RowPageId)
```

During the 2026-05-04 MAB Critic full-page audit, this exact pattern resolved a misdiagnosed Strategy lookup. `qo describe Strategies` showed the table PK was `["treatment_id"]`, and direct verification showed 27 unique Strategy treatment IDs with no duplicates. The issue was the inline `[Tid]` literal, not duplicate Strategy rows.

## Why this matters

Code that looks correct — passing exactly the right key vector — silently returns the wrong result set. There is no error or Mango throw; the query succeeds with extra rows. Downstream logic (writes, aggregations, existence checks) operates on all rows rather than the target row.

## Known scope

The failure has been observed on at least three tables covering both composite and single-column PKs. The root cause (whether this is a `defterm`-presence concern, an engine index-registration issue, or a more fundamental `sec-index`/`row-index` limitation) is unresolved as of 2026-04-29. Treat both `sec-index` and `row-index` as unreliable for single-row lookup until the engine team confirms otherwise.

## Canonical workaround: prop-vals-by-folder scan + filter

Replace any `sec-index` or `row-index` single-row lookup with a `prop-vals-by-folder` full scan filtered on the PK column(s). This is O(N) on the table size but works correctly.

### Single-column PK example (Campaign Analytics, PK = `id`)

From MAB SmartLead adapter commit `8507532` at
`~/Development/wttc/mab/impl/fc427017-adapters/02b4f492-experiment-platform/40550908-smartlead/0916029b-implementation.oql`:

```oql
(:- (FindCampaignAnalytics TableId CampaignId Analytics)
    (Qo.Public.OqlApi.Db.Prop/prop-vals-by-folder TableId _ ScanPvMap)
    (get-in ScanPvMap ["prop-vals" "id"] CampaignId)
    (get ScanPvMap "prop-vals" Analytics))
```

The `get-in` on `"id"` acts as an equality filter: `prop-vals-by-folder` iterates all rows and `get-in` only succeeds (binds) for the row where `id` equals the already-bound `CampaignId`. The outer query receives at most one solution.

For cold-start safety (when no row exists yet), gate the call with `if`:

```oql
(if (call FindCampaignAnalytics TableId CampaignId Analytics)
    ...found branch...
    ...not-found / zero branch...)
```

Phase 4 verification on 2026-04-29 confirmed this pattern end-to-end: 40-row drain at `limit=3` produced 40 unique experiment metrics rows with no phantom duplicates.

### Composite-PK example (Copy / Critiques, PK = `[treatment_id, draft]`)

```oql
(:- (FindCopyRow DbId TreatmentId Draft Row)
    (Qo.Public.OqlApi.Db.Prop/prop-vals-by-folder DbId _ PvMap)
    (get-in PvMap ["prop-vals" "treatment_id"] TreatmentId)
    (get-in PvMap ["prop-vals" "draft"] Draft)
    (get PvMap "prop-vals" Row))
```

Both `get-in` calls must succeed for a single `PvMap` solution — the conjunction acts as a two-column equality filter. This correctly scopes the result to the single row matching both key columns.

Reference implementations: `ReadCopyContent` and `WriteCritiqueNotes` in the MAB Critic helpers.

## What NOT to use

```oql
;; DO NOT use — returns all rows regardless of key
(Qo.Public.OqlApi.Db.Prop/row-index TableId [KeyValue] RowPid)
(Qo.Public.OqlApi.Db.Prop/sec-index TableId [KeyCol KeyValue] RowPid)
```

Both forms above silently fan out. There is no engine error — the failure is a wrong result count.

If the key comes from a bound symbol, also avoid writing the list inline. Bind it first:

```oql
(= RowKey [KeyValue])
(Qo.Public.OqlApi.Db.Prop/row-index TableId RowKey RowPid)
```

## Cross-references

- `omega-docs/gotchas/conventions.md` § Index matching rules — covers the general rule that index variables must match exactly; this doc covers the empirical failure mode where matching correctly still produces a full scan.
- `omega-docs/reference/built-ins.md` § Symbols in Literal Arguments — explains why inline map/list literals do not resolve bound symbols, including the `row-index` key-vector pattern documented above.
- MAB session 2026-04-29 Phase 1, commit `8507532` — single-column PK discovery on Campaign Analytics.
- `gap-sec-index-composite-key-fails` and `gap-sec-index-composite-key-fails-extension` in `omega-knowledge-base/gaps.md` — gap records with full fallback-source detail.
