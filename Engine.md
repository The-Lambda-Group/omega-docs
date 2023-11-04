# Omega Query Engine

## Evaluation Model
```clj
(assert!
 (balance "BTC" "John" 1 1)
 (balance "BTC" "John" 2 2)
 (balance "BTC" "John" 3 1)
 (balance "BTC" "Dave" 4 3)
 (balance "BTC" "Dave" 1000 5))

(:- (current-balance Token Block Holder Balance) 
    (balance Token Holder BalanceBlock _)
    (<= BalanceBlock Block)
    (with-group-by Holder
      (max BalanceBlock CurrentBlock)) 
    (balance Token Holder CurrentBlock Balance))

(eval (and (current-balance "BTC" 100 Holder Balance)
           (return [Holder Balance]))
      (and {Block 100
            Token "BTC"}
           (symbol-table [Holder BalanceBlock]
                         ["John" 1]
                         ["John" 2]
                         ["John" 3]
                         ["Dave" 4])
           (or {Holder "John"
                CurrentBlock 3}
               {Holder "Dave"
                CurrentBlock 4})
           (or {Holder "John"
                CurrentBlock 3
                Balance 1}
               {Holder "Dave"
                CurrentBlock 4
                Balance 5})
           (= Return [["John" 1]
                      ["Dave" 5]])))

(:- (max-holder Token Block Holder MaxBalance)
    (current-balance Token Block AnyHolder CurrentBalance)
    (with-group-by Token
      (max CurrentBalance MaxBalance))
    (current-balance Token Block Holder MaxBalance))

(eval (and (max-holder "BTC" 100 Holder Balance)
           (return [Holder Balance]))
      (and {Block 100
            Token "BTC"}
           (symbol-table [AnyHolder CurrentBalance]
                         [["John" 1]
                          ["Dave" 5]])
           {MaxBalance 5}
           {Holder "Dave"}
           (= Return [["Dave" 5]])))
```

## Predicate Refactoring

The predicate refactoring engine in omega leverages boolean algebra laws to refactor query expressions into more optimal queries in three phases. Predicate Expansion, Predicate Compaction, then Predicate Reordering.

### Problem Definition

```clj
(:- (sub2 String)
    (slice String Start End Slice)
    (length Slice 2))
```

### Predicate Expansion

```clj
(:- (sub2 String)
    (slice String Start End Slice)
    (length Slice 2))

(infers (slice String Start End Slice)
        (length String Length_String)
        (range Start 0 Length_String)
        (range End 0 Length_String)
        (length Slice Length_Slice)
        (- End Start Length_Slice))

(infers (range Start 0 Length_String)
        (>= Start 0))

(infers (range End 0 Length_String)
        (>= End 0))

(infers (and (length Slice Length_Slice)
             (length Slice 2))
        (= Length_Slice 2))

(infers (and (= Length_Slice 2)
             (- End Start Length_Slice))
        (- End Start 2))

(infers (and (- End Start 2)
             (>= Start 0))
        (>= End 2))

(infers (and (range End 0 Length_String)
             (>= End 2))
        (range End 2 Length_String))

(all-inferences (sub2 String)
                (slice String Start End Slice)
                (length Slice 2)
                (length String Length_String)
                (range Start 0 Length_String)
                (range End 0 Length_String)
                (length Slice Length_Slice)
                (- End Start Length_Slice)
                (>= Start 0)
                (= Length_Slice 2)
                (- End Start 2) 
                (>= End 2) 
                (range End 2 Length_String))
```

## Predicate Compaction

```clj

(compacts (and (range End 0 Length_String)
               (range End 2 Length_String)
               (>= End 0)
               (<= End Length_String))
          (range End 2 Length_String))

(compacts (and (range Start 0 Length_String)
               (>= Start 0)
               (<= Start Length_String))
          (range Start 0 Length_String))

(compacts (and (- End Start Length_Slice)
               (- End Start 2))
          (- End Start 2))

(compacts (and (length Slice 2)
               (length Slice Length_Slice))
          (length Slice 2))

(compacts (and (- End Start 2)
               (slice String Start End Slice)
               (length slice 2))
          (and (- End Start 2)
               (slice String Start End Slice)))

(all-compactions (subs2 String)
                 (range End 2 Length_String)
                 (length String Length_String)
                 (- End Start 2)
                 (slice String Start End Slice))
```

```clj
(solutions [(range End 2 Length_String)
            (length String Length_String)
            (- End Start 2)
            (slice String Start End Slice)]
           {String: BigO/Constant}
           [BigO/Quadratic,
            BigO/Constant,
            BigO/Infinite,
            BigO/Infinite])

(min-predicate [(range End 2 Length_String)
                (length String Length_String)
                (- End Start 2)
                (slice String Start End Slice)]
               {String: BigO/Constant}
               (length String Length_String))

(new-solutions (length String Length_String)
               {String: BigO/Constant}
               {Length_String: BigO/Constant})

(solutions [(range End 2 Length_String) 
            (- End Start 2)
            (slice String Start End Slice)]
           {String: BigO/Constant,
            Length_String: BigO/Constant}
           [BigO/Linear,
            BigO/Infinite,
            BigO/Quadratic])

(min-predicate [(range End 2 Length_String) 
                (- End Start 2)
                (slice String Start End Slice)]
               {String: BigO/Constant,
                Length_String: BigO/Constant}
               (range End 2 Length_String))

(new-solutions (length String Length_String) 
               {String: BigO/Constant,
                Length_String: BigO/Constant}
               {End: BigO/Linear})

(solutions [(- End Start 2)
            (slice String Start End Slice)]
           {String: BigO/Constant,
            Length_String: BigO/Constant,
            End: BigO/Linear}
           [BigO/Linear,
            BigO/Quadratic])

(min-predicate [(- End Start 2)
                (slice String Start End Slice)]
               {String: BigO/Constant,
                Length_String: BigO/Constant,
                End: BigO/Linear}
               (- End Start 2))

(new-solutions (- End Start 2)
               {String: BigO/Constant,
                Length_String: BigO/Constant,
                End: BigO/Linear}
               {Start: BigO/Constant})

(new-solutions (slice String Start End Slice)
               {String: BigO/Constant,
                Length_String: BigO/Constant,
                End: BigO/Linear,
                Start: BigO/Constant}
               {Slice: BigO/Constant})

(total-new-solutions [(range End 2 Length_String)
                      (length String Length_String)
                      (- End Start 2)
                      (slice String Start End Slice)]
                     {String: BigO/Constant}
                     {Length_String: BigO/Constant,
                      End: BigO/Linear,
                      Start: BigO/Constant,
                      Slice: BigO/Constant})

(total-solutions [(range End 2 Length_String)
                  (length String Length_String)
                  (- End Start 2)
                  (slice String Start End Slice)]
                 {String: BigO/Constant}
                 BigO/Linear)
```
