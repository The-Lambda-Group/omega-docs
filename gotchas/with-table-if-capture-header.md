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
