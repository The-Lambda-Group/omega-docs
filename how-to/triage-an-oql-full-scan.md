[< How-to](README.md)

# Triage `omega/query/full-scan`

Use this doc when an OQL runtime error tells you to read `query-omega-oql/docs/how-to/triage-an-oql-full-scan.md` but you arrived through the public docs tree.

This page is the **public re-anchor**. The full-scan-specific methodology lives in two places:

1. [narrow-an-oql-failure.md](narrow-an-oql-failure.md) -- owns the diagnostic loop: minimum repro first, shrink the frame, keep a ruled-in/ruled-out table, and do not pattern-match a prior fix.
2. [query-omega-oql/docs/how-to/triage-an-oql-full-scan.md](../../query-omega-oql/docs/how-to/triage-an-oql-full-scan.md) -- owns the Query Omega OQL specialization: compare bound args to the defterm pkey or `write-index` field set, restructure the caller first, and ask before using bookends or adding storage-layer indexes.

If you can read the repo-local doc, read it with `narrow-an-oql-failure.md` open. If you only have the public docs tree, keep reading here and then switch to the repo-local how-to for the worked examples and clause-level fix shapes.

## The routing answer

When the runtime error names:

```text
query-omega-oql/docs/how-to/triage-an-oql-full-scan.md
```

go to:

- [narrow-an-oql-failure.md](narrow-an-oql-failure.md) first
- then [../../query-omega-oql/docs/how-to/triage-an-oql-full-scan.md](../../query-omega-oql/docs/how-to/triage-an-oql-full-scan.md)

The public docs own the general triage discipline. The `query-omega-oql` repo owns the OQL-library-specific full-scan repair guidance.

## What to do before editing anything

1. Reproduce the failure with the smallest scratch query or stored-impl call that still throws.
2. Record the throwing term name, arity, and bound selector positions from the error payload.
3. Read the term's pkey and any `write-index` field sets.
4. Compare the bound args to those field sets.

If the bound args are wider than every existing pkey or index field set, the default fix is **caller restructure**: make the lookup match an existing pkey or index, then move any extra semantic check into `and`-joined `get` predicates after the fetch.

## What requires user permission

Do not do either of these as the default agent action:

- add `(enable-full-scan)` / `(disable-full-scan)` bookends
- add a new `write-index` or `defterm`

Those are storage-surface or behavior-scope changes. The repo-local how-to explains the escalation rule and why "add a defterm" is especially dangerous on populated data.

## Information ownership

This doc exists so the natural public-docs path resolves during active debugging. It does **not** replace the authoritative `query-omega-oql` how-to:

- `omega-docs/how-to/` owns the public debugging loop and routing
- `query-omega-oql/docs/how-to/` owns Query Omega OQL clause-level diagnosis, examples, and fix shapes

## Related

- [narrow-an-oql-failure.md](narrow-an-oql-failure.md)
- [../../query-omega-oql/docs/how-to/triage-an-oql-full-scan.md](../../query-omega-oql/docs/how-to/triage-an-oql-full-scan.md)
- [../reference/defterm.md](../reference/defterm.md)
