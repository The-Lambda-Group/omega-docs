[< Gotchas](README.md)

# with-count Over a Captured Helper Produces NullPointerException

> **Confidentiality: PUBLIC.** All content in this file is public platform documentation.

`with-count` over a captured in-memory helper — especially one containing a `prop-vals-by-folder` scan — produces a `NullPointerException` at runtime (`Cannot invoke "Object.toString()" because "s" is null`). The query fails hard; no count is returned.

## Known NPE patterns

Three distinct empirical occurrences have been observed as of 2026-04-30. Whether they share a root cause or are independent engine bugs is **Unknown** (see below).

### Pattern 1 — sec-index with trailing wildcard

```oql
;; NPE — with-count over a sec-index call with a wildcard output position
(with-count Cnt (Qo.Public.OqlApi.Db.Prop/sec-index TableId [Key] _))
```

Observed (MAB Enumerate session, session-handoff.md line 345). Hypothesis at time of observation: the trailing `_` wildcard in the output position may be related to the NPE. Root cause not confirmed.

### Pattern 2 — `call` on a captured in-memory helper

```oql
;; NPE — with-count over a call to a captured helper
(with-count Cnt (call CapturedHelper arg1 arg2))
```

Observed (MAB Manager session, session-handoff.md line 460). `reducers.md` § with-count documents this case and recommends "wrap the predicate in an in-memory clause and count over the clause invocation" — but that workaround is exactly Pattern 3 below, which also NPEs.

### Pattern 3 — captured helper containing `prop-vals-by-folder`

```oql
;; NPE — with-count over a captured helper that internally does prop-vals-by-folder
(:- (SaMatchesArm SaTableId Rev ExpId Pid)
    (Qo.Public.OqlApi.Db.Prop/prop-vals-by-folder SaTableId Pid Pv)
    (get-in Pv ["prop-vals" "revision"] Rev)
    (get-in Pv ["prop-vals" "experiment_id"] ExpId))

;; BAD — NPE at runtime
(:- (CountSaForArm SaTableId Rev ExpId SaCount {"capture" [SaMatchesArm]})
    (with-count SaCount (SaMatchesArm SaTableId Rev ExpId _)))
```

Observed (MAB Allocate V1, 2026-04-30, commit `5341753`). The captured helper is in-memory and contains a `prop-vals-by-folder` scan as its main predicate. Full stack trace was not captured at the time of observation.

Note: this pattern appears to be the suggested workaround for Pattern 2 — "wrap the predicate in a clause, count over the invocation" — yet it also NPEs. The `reducers.md` workaround is therefore insufficient when the wrapping clause contains a `prop-vals-by-folder` scan.

## Root cause

**Unknown.** All three patterns produce a NullPointerException at the `with-count` layer. Whether the engine failure is:

- related to wildcard output positions in the inner term,
- related to captured-helper context resolution, or
- related to how `prop-vals-by-folder`'s solution structure interacts with `with-count`'s counting mechanism

...is not confirmed. Engine maintainers should investigate. The three patterns may be the same underlying bug or independent NPE sites.

**Until the root cause is confirmed and fixed:** treat `with-count` as unreliable whenever its inner term is anything other than a simple single-clause datastore call with no wildcards in non-binding positions and no captured-helper context. Use the workaround below.

## Canonical workaround: with-table-if + fold-append-per-row-unique + length

Replace `with-count` entirely with a three-part pattern:

1. `with-table-if` — runs the compound predicate as its condition; if zero rows match, the else-branch binds the accumulator to `[]`.
2. `fold [] PerRowUniqueSymbol append List` in the then-branch — folds a symbol that is unique per row (e.g., the page-id returned by `prop-vals-by-folder`, which is a CouchDB document id and naturally unique per row). Per `reducers.md` § "fold folds over unique values, not rows": because the fold symbol is unique per row, fold produces one entry per matching row with no deduplication loss.
3. `(length List Count)` outside the `with-table-if` — computes the count from the accumulated list.

```oql
;; GOOD — avoids with-count entirely
(:- (CountSaForArm SaTableId Rev ExpId SaCount)
    (with-table-if (and (Qo.Public.OqlApi.Db.Prop/prop-vals-by-folder SaTableId SaPid SaPv)
                        (get-in SaPv ["prop-vals" "revision"] Rev)
                        (get-in SaPv ["prop-vals" "experiment_id"] ExpId))
      [SaTableId Rev ExpId SaPid SaPv SaList]
      (fold [] SaPid append SaList)
      (= SaList []))
    (length SaList SaCount))
```

Why each piece is necessary:

- **`with-table-if` condition runs the compound predicate inline** — no captured helper needed. The condition is an `(and ...)` conjunction; the else-branch guarantees a binding when zero rows match (so `length` always has something to operate on).
- **The header `[SaTableId Rev ExpId SaPid SaPv SaList]` must include all symbols referenced in condition or branch bodies** — including the datastore symbols (`SaTableId`) and all data variables. See [with-table-if header must include captured clause symbols](with-table-if-capture-header.md).
- **`SaPid` is the per-row-unique symbol** — `prop-vals-by-folder` binds the CouchDB page-id in its second argument position. Page-ids are unique per row by definition, so folding on `SaPid` produces one accumulator entry per matching row.
- **`(length SaList SaCount)` lives outside `with-table-if`** — `with-table-if` always produces exactly one solution row (either the then-branch result or the else-branch result), so `length` operates on the single bound `SaList` value.

Reference implementation: `CountSaForArm` in
`~/Development/wttc/mab/impl/a24d83d6-workflows/c6ca77cd-allocation/05a97b4e-mab-allocate/992aad8a-implementation.oql`
lines 154-161 (MAB Allocate V1, commit `5341753`, 2026-04-30).

## When to use which

| Situation | Recommendation |
|-----------|---------------|
| Counting rows matching a simple single-clause call, no wildcards, no captured helper | `with-count` (standard form) |
| Counting rows matching a compound predicate or captured-helper invocation | Use the `with-table-if` + `fold`-per-row-unique + `length` pattern above |
| Any new code requiring "count rows matching predicate" where captured helpers or `prop-vals-by-folder` are involved | Use the workaround by default until the `with-count` NPE is resolved |

## Cross-references

- [reducers.md § with-count](../reference/reducers.md#with-count) — canonical reference for `with-count` semantics; documents the compound-predicate NPE and the "wrap in clause" workaround (which is insufficient in Pattern 3).
- [reducers.md § fold folds over unique values, not rows](../reference/reducers.md#fold-folds-over-unique-values-not-rows) — explains why folding a per-row-unique symbol (like a page-id) produces one entry per row.
- [with-table-if header must include captured clause symbols](with-table-if-capture-header.md) — the header rule that applies to the workaround pattern's condition symbols.
- `omega-knowledge-base/projects/mab/reference/session-handoff.md` line 345 — Pattern 1 occurrence (sec-index trailing-wildcard NPE).
- `omega-knowledge-base/projects/mab/reference/session-handoff.md` line 460 — Pattern 2 occurrence (`call CapturedHelper` NPE).
