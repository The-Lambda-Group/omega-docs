# Grouping

Grouping Clauses allow OQL solution sets to be divided by a specific key.

```clj
(:- (current-token-balance Token Holder Block Balance) 
    (token-transfer Transfer)
    (destructure Transfer {"to_addr" Holder})
    (token-balance TokenBalance)
    (destructure TokenBalance
                 {"token" Token,
                  "holder" Holder,
                  "balance" Balance
                  "block_number" BalanceBlock})
    (<= BalanceBlock Block)
    (group-by
     [Token]
     (every
      (Set/contains BalanceBlockSet BalanceBlock))
     (Set/max BalanceBlockSet MaxBlock))
    (= MaxBlock BalanceBlock))
```

Inside of the group-by clause, universal quantifiers are restricted to that particular grouping.

```clj
(group-by
 [Token]
 (every
  (Set/contains BalanceBlockSet BalanceBlock))
 (Set/max BalanceBlockSet MaxBlock))
 ```