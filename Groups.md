# Grouping

Reducers and Groupings are Omega's was of expression relationships across solutions. This is the only place in which solutions aren't treated in isolation. Using reducers, we get aggregate operations such as SQL functions including `AVG`, `SUM`, and `MAX`.

Groupings allow OQL solution sets to be partitioned by a specific key. Reducers operate across all solution sets in the current partition.

```clj
(= DS (datastore "https://db.omegadb.io/examples/blockchain"))

(:- (current-token-balance Token Holder Block Balance)
    (DS/token-balance TokenBalance)
    (destructure TokenBalance
                 {"token": Token,
                  "holder": Holder,
                  "balance": AnyBalance,
                  "block_number": BalanceBlock})
    (<= BalanceBlock Block)
    (group-by
     [Token, Holder, Block]
     (reduce max BalanceBlock CurrentBalanceBlock))
    (when (= CurrentBalanceBlock BalanceBlock)
      (= AnyBalance Balance)))
```

## Ordering

### Ordered Reducers

Some reducers by their very nature require order. And sometimes ordering is just more performant. Consider the clause `(max-at-i List I MaxAtI)` which returns a lists maximum element up to a particular index. Whether you're computing a single `I` for a list or every `I` for a list, it's still going to be best to order this operation by the `I` value. These two definitions are logically equivilent, but the ordered version is much more efficient.

```clj
(:- (max-at-i List I MaxAtI)
    (nth List J Element)
    (<= J I)
    (group-by
     [List, I]
     (reduce Element max MaxAtI)))

(:- (max-at-i List I MaxAtI)
    (nth List J Element)
    (<= J I)
    (group-by
     [List, I]
     (order-by
      J
      (reduce Element max MaxAtI))))
```

### Unordered Reducers

Other reducers aren't concerned much with order at all. This is true for any reducing operation that has an associative property.

```clj
(:- (max-of-list List Max)
    (group-by
     [List]
     (reduce List max Max)))
```

In the case of the `(max-of-list List Max)` clause, the logic doesn't depend much on order at all. In this case, the engine can partition the list and perform this operation in parralel, this is especially performant as you scale across multiple lists. Any time a reducer doesn't specify an ordering, the solutions may be partitioned any way the query engine deems performant.
