[< Home](../README.md) | [Gotchas](README.md)

# with-table-if Header Must Include Captured Clause Symbols

## Symptom

Calling a captured in-memory clause inside a `with-table-if` branch fails silently or errors — the clause symbol is unbound inside the branch.

## Why

`with-table-if` scopes its branches to only the symbols listed in the header array. If an in-memory clause symbol is not included in the header, it is not available inside the branch body.

## Fix

Add the clause symbol to the `with-table-if` header array:

```oql
;; WRONG — SortedConcat not in header, unavailable in branch
(with-table-if (= F0 F1) [F0 F1 T0 T1 Combined]
    (= "true" "false")
    (call SortedConcat T0 T1 Combined))  ;; SortedConcat is unbound

;; CORRECT — SortedConcat included in header
(with-table-if (= F0 F1) [F0 F1 T0 T1 Combined SortedConcat]
    (= "true" "false")
    (call SortedConcat T0 T1 Combined))  ;; works
```

## General Rule

Any symbol you reference inside a `with-table-if` branch — whether it is a data variable or a clause symbol — must appear in the header array. This applies to all captured values, not just clause symbols. The header defines the scope boundary.

## Related

- [Clauses — Closures / Capture](../reference/clauses.md#closures--capture) — the language-reference home for capture semantics. Explains the `{"capture" [...]}` signature syntax for threading outer-scope bindings (including clause symbols) into a clause body, and the warning about known edge cases with capturing clauses inside other captured clauses. The `with-table-if` header rule documented here is one specific manifestation of the broader "captured symbols must be made available at the scope boundary" pattern.
- [Lexical scope — Capture Only Works With call](lexical-scope.md#capture-only-works-with-call) — the dual rule on the invocation side: a captured clause symbol must be invoked with `call`, not as a bare functor.
