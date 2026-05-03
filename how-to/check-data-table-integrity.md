[< Home](../README.md) | [How-to](README.md)

# Check Data Table Integrity Before Debugging Helpers

Use this when a table-backed lookup fails, a row reads as empty, or a helper probe produces surprising row-page results. Before diagnosing helper logic, prove whether the property table's storage contract is intact.

The common invariant is:

```text
logical row key -> row-index entry -> row page -> required row content
```

If any link is missing, helper probes that depend on the same table are not clean evidence about the helper.

## When to run this check

Run this before write-helper debugging when:

- `qo read <table>/<row-key>` returns `[]` for a row you expect to exist.
- A pkey or row-page lookup returns no result, too many results, or a stale row page.
- A row page exists but an expected HTML block such as `Content` or `Notes` no-returns.
- A write helper that should upsert one row appears to fan out over multiple row pages.
- The table is disposable and you are considering a drop/rebuild.

Do not use a destructive reset as the diagnostic. Prove the corruption predicate first, then ask for reset approval.

## Proof Ladder

### 1. Resolve the table folder id

Use the project-specific page walk or `qo describe` path that normally resolves the table. Record the folder id in the debug note or plan output.

```text
Table folder id: 8438c151-3c27-4d5a-bc04-2a6a8d2dfedb
```

### 2. Query the row-index for the failing row key

Use the direct property row-index datastore. Row keys are arrays. For a single-column primary key, pass `[Key]`, not `Key`.

```oql
(datastore Qo.Data.Db.Prop.RowIndex "omega/query-omega/data/database/property/row-index")

(= FolderId "8438c151-3c27-4d5a-bc04-2a6a8d2dfedb")
(= RowKey ["<exact-row-key-string>"])
(Qo.Data.Db.Prop.RowIndex/row-index FolderId RowKey PageId)
(return [PageId])
```

If this no-returns while another path has shown a row page or property values for the same logical row, the table is inconsistent. Stop helper debugging and inspect the table.

### 3. Scan the row-index for the whole folder

Bind the folder id and leave the row key unbound.

```oql
(datastore Qo.Data.Db.Prop.RowIndex "omega/query-omega/data/database/property/row-index")

(= FolderId "8438c151-3c27-4d5a-bc04-2a6a8d2dfedb")
(Qo.Data.Db.Prop.RowIndex/row-index FolderId RowKey PageId)
(= Entry [RowKey PageId])
(fold [] Entry append Rows)
(length Rows Count)
(return [Count Rows])
```

Compare `Count` to the number of rows the workflow should have written. If you know the exact expected keys, compare those too. Missing keys mean the table cannot support row-key lookups reliably.

### 4. Verify row-page content

If the table stores prose or rich text on row pages, verify the required block on every indexed page.

```oql
(datastore Qo.Data.Db.Prop.RowIndex "omega/query-omega/data/database/property/row-index")
(datastore Qo.Public.OqlApi.Page "omega/query-omega/public/oql-api/page")
(datastore Qo.Public.OqlApi.Page.Bl.Html "omega/query-omega/public/oql-api/page/block/html")

(= FolderId "8438c151-3c27-4d5a-bc04-2a6a8d2dfedb")
(Qo.Data.Db.Prop.RowIndex/row-index FolderId RowKey PageId)
(Qo.Public.OqlApi.Page/page-by-id PageId Page)
(Qo.Public.OqlApi.Page.Bl.Html/get-content Page "Content" Html)
(= Entry [RowKey PageId])
(fold [] Entry append Rows)
(length Rows Count)
(return [Count Rows])
```

The count here should match the row-index count when every indexed row requires that block. If it is lower, the table may have index entries but incomplete row pages.

### 5. Compare property values and row-index entries

When you suspect property rows exist without matching index rows, scan the table's property values using the public table API available to the project and compare logical keys against the direct row-index scan.

For table-specific code, prefer the project's canonical read helper or `prop-vals-by-folder` scan plus field filters. The important comparison is:

```text
keys present in property values == keys present in row-index
```

If property values exist but row-index entries are missing, write-helper probes that start from row-index are testing corrupted storage, not the helper's intended behavior.

## Reset Gate

Only request a destructive reset when at least one concrete corruption predicate is proven:

- A property row exists but the matching row-index entry is missing.
- A row-index entry points to a row page missing required content.
- Row-index count disagrees with the expected/generated row count.
- Known required row keys are absent from row-index after the producing workflow has completed.

The reset request should name the evidence and scope:

```text
The table is disposable. I verified row-index count is 22 but expected 27, and two known required row keys are missing from row-index. Requesting permission to drop/rebuild this table only.
```

After rebuild, verify the full table contract, not just the originally blocked row:

- Row-index count matches expected row count.
- Every indexed row page has required blocks.
- The originally blocked row reads successfully.
- Any sampled candidate rows read successfully.
- The implementation file has no leftover debug diff.

## Incident Reference

This proof ladder comes from the 2026-05-03 MAB Strategist table investigation. `Data/Strategies` had some property rows and row pages, but the row-index was missing entries and affected row pages had no `Content` block. A single-row `WriteStrategy` probe against that table was misleading because it assumed row-index integrity, which was the dependency under test. Direct row-index inspection proved table corruption; a user-authorized drop and production backfill restored 27/27 indexed rows with `Content`.

## Related

- [Develop OQL](develop-oql.md) -- general probe-first debugging discipline.
- [Triage no-return](triage-no-return.md) -- how body-level zero-solution failures surface.
- [Execution model](../explanation/oql-execution-model.md) -- no transactions; writes before a later failure are not rolled back.
- [sec-index and row-index do not reliably filter rows](../gotchas/sec-index-composite-key-fails.md) -- row-index lookup caveats and scan/filter workaround.
