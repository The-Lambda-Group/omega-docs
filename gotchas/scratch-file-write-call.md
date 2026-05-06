[< Gotchas](README.md)

# Calling a !-suffixed clause from a scratch file invokes the write pathway

**Symptom:** A scratch query calls a stored clause using `(Ds/term! Args)` and throws `WRITING_SYMBOLS` instead of returning results.

**Root cause:** The `!` suffix controls the read/write pathway at the call site, not just at the definition site. A `(:- (Ds/term! Args) ...)` definition stores the clause. But a bare call `(Ds/term! Args)` in a query body tells the engine to **write** — it attempts to store a new clause with `Args` as the head arguments. If `Args` contains unbound symbols (which it almost always does in a scratch query), the engine throws `WRITING_SYMBOLS` because it cannot write a clause with unresolved head variables.

## The convention

```oql
;; In a scratch file or query body:
(Ds/term Args)   ;; READ  — queries the stored clause, produces solutions
(Ds/term! Args)  ;; WRITE — stores a clause (rarely what you want in a scratch query)
```

This is the same convention used at definition time (`:-` bodies), but it applies equally to call sites. See [clauses.md § Stored Clauses](../reference/clauses.md) for the full explanation.

## Minimal example

**Wrong** — throws `WRITING_SYMBOLS`:

```oql
;; Stored clause definition (lives in the datastore):
;; (:- (Qo.Page/visible! PageId) ...)

;; Scratch query — WRONG: ! suffix triggers write, not read
(datastore Qo.Page "omega/query-omega/page")

(Qo.Page/visible! PageId)  ;; ERROR: engine tries to store a clause with unbound PageId
(return [PageId])
```

**Correct** — reads from the stored clause:

```oql
(datastore Qo.Page "omega/query-omega/page")

(Qo.Page/visible PageId)   ;; READ: queries the stored clause, binds PageId per solution
(return [PageId])
```

## Fix

Remove the `!` from the call. The `!` belongs only on the definition head (`(:- (Ds/term! ...) ...)`), not on calls that query the clause.

## Cross-reference

- [clauses.md § Stored Clauses](../reference/clauses.md) — full read/write convention documentation and examples.
