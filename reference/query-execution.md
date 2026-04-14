[< Home](../README.md) | [Reference](README.md)

# Query Execution

## with-skip / with-limit only work on view queries

`with-skip` and `with-limit` thread `skip` and `limit` into the CouchDB view POST body as native parameters. They are the only way to paginate results inside OQL.

### The constraint

They **only** work when the runtime resolves the term through a **view index** (e.g. `FolderIndex` on `args.1`).

When the runtime cannot match a view index for the bound args, it falls back to a **Mango scan**. In that code path, `with-skip` and `with-limit` are **silently ignored** — the query returns all matching rows with no error or warning.

### Why this matters

The page-query default limit is **1,000,000** — effectively unbounded. If a caller passes `with-limit 10` but the bound args don't match a view index, the query quietly returns up to one million rows instead of ten. This is a silent correctness bug.

### What callers must do

Ensure the bound args in the clause call match a declared view index. If the args don't match, the runtime falls back to Mango and skip/limit become no-ops.

For example, `FolderIndex` indexes on `args.1`. A clause call that binds `args.1` to a folder ID will resolve through `FolderIndex` and skip/limit will work. A clause call that only binds `args.2` won't match `FolderIndex`, will fall back to Mango, and skip/limit will be ignored.

### Future plans

There are plans to make the runtime throw an error when it falls back to a Mango scan, which would turn this silent bug into an explicit failure. Until that change lands, callers are responsible for verifying index alignment.
