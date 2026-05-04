[< Home](../README.md) | [Reference](README.md)

# Built-in Terms

## Primitives that look like they should exist but don't

OQL's built-in inventory is smaller than Clojure's or Common Lisp's. Several primitives that agents reach for by training default are not implemented. They will fail at runtime with no hint that a workaround exists. This section is the negative-space companion to the positive lists below — check here first when a name "feels obvious."

| Name agents reach for | Status | Canonical workaround |
|---|---|---|
| `string-length` | Does not exist. | No primitive returns string length. To inspect a string, bind it directly into Result and let the caller read it. There is no length-of-string operator. |
| `string-take` / `string-substr` / `substring` | None exist. | The string toolkit is exactly: `string-split`, `string-concat` (binary — chain calls), `string-contains`, `string-replace`, `re-find`, `json-stringify` (bidirectional). Slicing a string by index is not supported; use `string-split` on a delimiter or `re-find` on a regex pattern instead. |
| `keys` | Does not exist. | Use the K-binding-then-fold idiom: `(get Map K _) (fold [] K append KeysList) (length KeysList Count)`. Each `(get Map K _)` enumerates keys non-deterministically across the solution table; `fold` collects them. |
| `assoc` (as a fold reducer) | Does not exist. | When folding into a map, use `append` with key-value pair entries (the Enumerate `IntoSet` pattern): `(= Entry [El El]) (fold {} Entry append Set)`. The `append` reducer accepts pair entries when the seed is `{}`. |

If you find another "obvious" name that doesn't exist, add it here rather than letting the next agent rediscover it the hard way.

## Core

```oql
(= X 5)                              ;; unify X with 5
(defined X)                          ;; check if symbol is bound
(return [Result])                     ;; return from a query
(uuid Id)                             ;; generate UUID
(time-now Timestamp)                  ;; current ISO timestamp
```

## Arithmetic

```oql
(+ 1 2 Sum)                          ;; addition
(- 6 1 Diff)                         ;; subtraction
(* 3 4 Product)                      ;; multiplication
(/ 10 2 Quotient)                    ;; division
(> A B)                              ;; greater than (succeeds or fails)
(< A B)                              ;; less than
(>= A B)                             ;; greater than or equal
```

## Random

```oql
(rand Output)                        ;; uniformly random double in [0.0, 1.0)
```

**`rand`** binds `Output` to a uniformly random double in the half-open interval `[0.0, 1.0)`. It is a pure output term — there are no input arguments and no failure cases.

```oql
;; Generate a random probability weight
(rand Weight)

;; Use as a threshold for 30 % sampling
(rand R)
(< R 0.3)   ;; succeeds ~30% of the time; fails and drops the solution otherwise
```

**Randomness source:** `java.util.concurrent.ThreadLocalRandom` (JVM built-in). Thread-safe with no lock contention across query threads.

**Constraints and non-goals:**

| Property | Value |
|---|---|
| Range | `[0.0, 1.0)` — lower bound inclusive, upper bound exclusive |
| Output type | double |
| Seedable? | No — each call is independent, no `with-rand-seed` |
| Cryptographic? | No — not suitable for security-sensitive uses |
| Failure cases | None — `(rand Output)` never throws |

**Note on multi-solution queries:** in a query that produces N solutions, each solution gets its own independent call to `rand` — the same OQL expression appearing once in the body produces a different value per solution row. This is the expected behavior for sampling use cases.

### `rand-int` — random integer in a range

```oql
(rand-int Min Max Output)            ;; uniformly random integer in [Min, Max)
```

**`rand-int`** binds `Output` to a uniformly random integer in the half-open interval `[Min, Max)`. `Min` and `Max` are both required inputs; `Output` is the generated value.

```oql
;; Roll a six-sided die (1–6)
(rand-int 1 7 Roll)

;; Pick a random index into a 10-element array
(rand-int 0 10 Idx)

;; Negative ranges are supported
(rand-int -10 0 Offset)   ;; Offset in [-10, -1]

;; Single-value range — always returns Min
(rand-int 5 6 Fixed)      ;; Fixed = 5
```

**Signature:**

| Arg | Direction | Type | Notes |
|---|---|---|---|
| `Min` | input | integer | Inclusive lower bound. |
| `Max` | input | integer | Exclusive upper bound. Must be greater than `Min`. |
| `Output` | output | long | Random integer in `[Min, Max)`. |

**Validation and typed errors:**

`rand-int` validates its inputs at call time and throws a typed OQL exception on any violation. Unlike most existing arithmetic terms, it does not silently NPE on bad input.

| Condition | Exception type | When it fires |
|---|---|---|
| `Min` or `Max` is unbound at call time | `RAND_INT_UNBOUND_ARG` | A symbol was passed for `Min` or `Max` but it has not been bound in the current solution context. |
| `Min` or `Max` is not an integer | `RAND_INT_NON_INTEGER` | Floats, strings, maps, or any non-integer value passed as a bound. |
| `Min >= Max` | `RAND_INT_BAD_RANGE` | Empty or inverted range — no valid integer can satisfy `[Min, Max)`. |

**Constraints and non-goals:**

| Property | Value |
|---|---|
| Range | `[Min, Max)` — lower bound inclusive, upper bound exclusive |
| Output type | long |
| Seedable? | No — each call is independent, no `with-rand-seed` |
| Cryptographic? | No — not suitable for security-sensitive uses |

**Randomness source:** `java.util.concurrent.ThreadLocalRandom` (JVM built-in, same as `rand`). Thread-safe with no lock contention across query threads.

**Sibling term:** `rand` generates a random double in `[0.0, 1.0)` with no inputs and no failure cases — documented above.

## Map Operations

```oql
(get Map "key" Value)                 ;; read a key (unifies Value)
(get-in Map ["a" "b"] Value)          ;; nested read (unifies Value)
(set Map "key" NewVal Result)         ;; set a key (returns new map)
(merge Map1 Map2 Result)             ;; merge two maps (Map2 wins on conflict)
(select-keys Map ["k1" "k2"] Result) ;; return new map with only the listed keys
(zip Keys Values Map)                ;; create a map from parallel key/value arrays
```

**`merge`** combines two maps. When both maps have the same key, Map2's value wins:
```oql
(= A {"x" 1 "y" 2})
(= B {"y" 3 "z" 4})
(merge A B Result)   ;; Result = {"x" 1 "y" 3 "z" 4}
```

**`select-keys`** extracts a subset of keys from a map:
```oql
(= Data {"name" "Alice" "age" 30 "city" "NYC" "score" 95})
(select-keys Data ["name" "score"] Result)   ;; Result = {"name" "Alice" "score" 95}
```

**`zip`** creates a map from two parallel arrays (keys and values). Also works in reverse — given keys and a map, extracts the values:
```oql
;; Forward: keys + values -> map
(= Header ["name" "age" "city"])
(= Row ["Alice" 30 "NYC"])
(zip Header Row Result)   ;; Result = {"name" "Alice" "age" 30 "city" "NYC"}

;; Reverse: keys + map -> values
(= Data {"name" "Alice" "age" 30 "city" "NYC"})
(zip ["name" "city"] Values Data)   ;; Values = ["Alice" "NYC"]
```

## Array / List Operations

```oql
(get-list Map ["k1" "k2" "k3"] Values)   ;; extract values for keys as an array
(length Array Len)                         ;; array length
(append Array Item Result)                 ;; append item to array
(concat Array1 Array2 Result)              ;; concatenate two arrays
(partition Array N Chunks)                 ;; split array into chunks of size N
```

**`get-list`** extracts multiple values from a map or array by key/index, returning them as an ordered array:
```oql
(= Data {"name" "Alice" "age" 30 "score" 95})
(get-list Data ["name" "score"] Values)   ;; Values = ["Alice" 95]

;; Also works on arrays by index
(= Row ["Alice" 30 "NYC"])
(get-list Row [0 2] Values)   ;; Values = ["Alice" "NYC"]
```

## String Operations

```oql
(string-split Path "/" Segments)          ;; split string
(string-concat "Hello " Name Greeting)   ;; concatenate strings (strictly binary — two inputs + output)
(string-contains Str "sub" Result)        ;; check substring
(string-replace Str "," "" Clean)         ;; replace substring
(re-find Str "\\d+" Match)               ;; regex find
(json-stringify Obj Str)                  ;; JSON: object <-> string (bidirectional)
```

**`string-concat` is binary, not variadic.** It always takes exactly two input strings and an output. To concatenate three or more strings, chain calls and thread an intermediate variable through each step:

```oql
(string-concat "https://api.example.com/contacts/?locationId=" LocationId Url0)
(string-concat Url0 "&limit=100" Url)
```

**`json-stringify` is bidirectional — there is no `json-parse`.** Direction is determined by which argument is bound:

- **Stringify (object to string):** bind the first arg, leave the second unbound.
  ```oql
  (= Data {"name" "Alice" "age" 30})
  (json-stringify Data JsonStr)        ;; JsonStr = "{\"name\":\"Alice\",\"age\":30}"
  ```

- **Parse (string to object):** leave the first arg unbound, bind the second.
  ```oql
  (get Res "body" RawBody)             ;; RawBody is a JSON string from an HTTP response
  (json-stringify Parsed RawBody)      ;; Parsed is now a map/array you can (get ...) into
  ```

There is no `json-parse` built-in. Calling `(json-parse Body Result)` produces a 500 with no hint that you need `json-stringify` with reversed binding.

**Probing validity: use a named symbol, not `_`.** To test whether a string is valid JSON without caring about the parsed value, you might reach for the wildcard. It doesn't work — `(json-stringify _ Text)` always fails, even on known-valid JSON. Use a named (but otherwise unused) symbol instead:

```oql
;; WRONG — always fails, even for valid JSON
(when-not (json-stringify _ Text)
  (throw "PARSE_ERROR" ParseErr))

;; RIGHT — named probe symbol, succeeds iff Text parses
(when-not (json-stringify ParseProbe Text)
  (throw "PARSE_ERROR" ParseErr))
```

Likely cause: `_` lets the engine pick a direction, and it chooses stringify (first arg as source) — stringifying an unbound wildcard into the already-bound string fails unification. A named symbol unifies cleanly in the parse direction. This mirrors the general OQL rule that `_` is a discard slot, not a free variable — see [Lexical scope: The `_` Symbol](../gotchas/lexical-scope.md#the-_-symbol).

## Data Transformation Patterns

The terms `merge`, `get-list`, `zip`, and `select-keys` (documented in Map Operations and Array Operations above) form the primary data manipulation toolkit. Prefer these over extracting fields one-by-one with `(get Map "key" Val)`.

### Preparing data for `write-table`

`write-table` expects columnar input: `{"file-id" FolderId "data" {"header" [...] "rows" [...]}}`. The efficient pattern is:

1. Define the header array once (this is the column order contract).
2. For each record, use `get-list` or `zip` (reverse) to extract values in that exact order.
3. Pass `{"header" Header "rows" [Values]}` to `write-table`.

```oql
;; Define the column order
(= Header ["name" "email" "status"])

;; Extract values from an API response record in column order
(get-list Record ["name" "email" "status"] Row)

;; Build the write-table input
(= Input {"file-id" PageId "data" {"header" Header "rows" [Row]}})
(Qo.Public.OqlApi.Db.Prop/write-table Input _)
```

This replaces the verbose anti-pattern of extracting each field individually:

```oql
;; Don't do this — verbose, fragile, easy to get column order wrong
(get Record "name" Name)
(get Record "email" Email)
(get Record "status" Status)
```

### When to use which

| Need | Term |
|------|------|
| Combine fields from two sources | `merge` |
| Extract values in column order (for `write-table`) | `get-list` or `zip` (reverse) |
| Convert row array + header to map | `zip` (forward) |
| Drop unwanted fields from a map | `select-keys` |

## Symbols in Literal Arguments

Symbol bindings do not resolve inside literal map or list arguments passed directly to any term. This is a general OQL behavior, not specific to particular primitives — it affects `throw`, `run-page`, `set-data`, and every other term that accepts a map or list in an argument position. The contents of the literal are read at parse time, before the row context binds symbols — so `{"key" MySymbol}` becomes a literal map with the *text* `"MySymbol"`, not the value of `MySymbol`.

**The fix is always the same:** bind the map to a variable with `(=)` first, then pass the variable.

```oql
;; WRONG — Payload stays as a literal symbol name
(throw "error-tag" {"payload" Payload "count" 42})

;; RIGHT — bind the map to a symbol first, then pass the symbol
(= Data {"payload" Payload "count" 42})
(throw "error-tag" Data)
```

Same pattern for `run-page`:

```oql
(= Args {"row-id" RowId "status" Status})
(run-page "some-page" Args)
```

Same pattern for datastore operations like `set-data`:

```oql
;; WRONG — ResearcherInputStr is stored as the literal text "ResearcherInputStr"
(set-data Page "Config" {"name" "Config" "input-pages" ResearcherInputStr} _)

;; RIGHT — bind first, then pass the variable
(= Config {"name" "Config" "input-pages" ResearcherInputStr})
(set-data Page "Config" Config _)
```

Any term accepts its arguments as values, not as expressions — the argument position is a value slot. When you write a literal map directly in the call site, OQL reads it as a literal value before evaluating expressions. Only when the map is bound to a symbol via `(=)` does the binder first evaluate the expressions inside the map and then assign the result.

This also applies to inline key vectors passed to index terms. If you call `row-index` or `sec-index` with a list literal that contains a bound symbol, the symbol does not resolve inside the inline list. Bind the key list first, then pass the symbol:

```oql
;; WRONG — Tid stays symbolic inside the inline list
(Qo.Public.OqlApi.Db.Prop/row-index StrategiesDbId [Tid] RowPageId)

;; RIGHT — bind the key vector first
(= RowKey [Tid])
(Qo.Public.OqlApi.Db.Prop/row-index StrategiesDbId RowKey RowPageId)
```

For composite keys, use the same pattern:

```oql
(= RowKey [TreatmentId Draft])
(Qo.Public.OqlApi.Db.Prop/row-index CopyDbId RowKey RowPageId)
```

If `row-index` or `sec-index` appears to fan out across many rows, check this literal-argument rule before assuming the table has duplicates or the index itself is broken. For the broader lookup reliability issue, see [sec-index and row-index do not reliably filter rows](../gotchas/sec-index-composite-key-fails.md).

There is also an observed list-wrapper edge case that is distinct from the separate 9+ map-literal bug. A bound map can still leak through as a symbolic placeholder when you wrap it in an inline one-element list literal:

```oql
;; Empirically unsafe for map payloads like next-args
(= NextArg {"step" "Load Subjects" "skip" NextSkipStr "limit" BatchSize})
(= NextArgs [NextArg])
```

In the 2026-05-04 MAB Critic Task 15 result envelope, the safe workaround was to bind the map first and then build the list through `fold` + `append`:

```oql
(= NextArg {"step" "Load Subjects" "skip" NextSkipStr "limit" BatchSize})
(fold [] NextArg append NextArgs)
(= Result {"has-more" "true" "next-args" NextArgs})
```

Use this pattern when a term or return envelope needs a list of maps and the list would otherwise be written as an inline literal around a bound map. This is **not** the same failure as the 9+ entry map-literal threshold bug: that bug affects large map binds and is worked around by splitting the map and merging the parts, while this list-wrapper case can happen with a single small map.

## HTTP

**`http-request`** is the general-purpose HTTP term and the only safe HTTP primitive. It takes a request map and returns a response map. The request map is passed directly to the underlying HTTP client ([clj-http](https://github.com/dakrone/clj-http)), so all clj-http options are supported.

```oql
;; GET with query params
(= Req {"method" "get"
         "url" "https://api.example.com/contacts"
         "headers" {"Authorization" "Bearer token123"
                    "Content-Type" "application/json"}
         "query-params" {"locationId" "abc" "limit" "100"}})
(http-request Req Res)
(get Res "status" Status)    ;; 200
(get Res "body" Body)        ;; response body as string

;; POST with JSON body
(= Req {"method" "post"
         "url" "https://api.example.com/contacts"
         "headers" {"Authorization" "Bearer token123"
                    "Content-Type" "application/json"}
         "body" "{\"firstName\": \"Jane\"}"})
(http-request Req Res)

;; DELETE
(= Req {"method" "delete"
         "url" "https://api.example.com/contacts/123"
         "headers" {"Authorization" "Bearer token123"}})
(http-request Req Res)
```

**Request map keys:**

| Key | Description |
|-----|-------------|
| `method` | HTTP method: `"get"`, `"post"`, `"put"`, `"delete"` |
| `url` | Full URL |
| `headers` | Map of header name to value |
| `query-params` | Map of query parameter name to value (appended to URL) |
| `body` | Request body (string) |

**Response map keys:**

| Key | Description |
|-----|-------------|
| `status` | HTTP status code (number) |
| `body` | Response body (string) |
| `headers` | Response headers (map) |

### Deprecated method-specific terms

`http-get-request` and `http-post-request` are **deprecated**. They do not capture HTTP error codes (e.g. 500s, 400s). Any clause using them that hits a non-2xx response will silently fail to bind the expected response fields, leaving the calling clause with no way to distinguish success from error.

```oql
;; Deprecated — use http-request with "method" "get" instead
(http-get-request Url {"headers" Headers} Res)
(http-post-request Url {"headers" Headers "body" Body} Res)
(http-delete-request Url {"headers" Headers} Res)
```

These take `(Url Config Res)` instead of a single request map. They do not support `query-params` — you must build the URL manually. When reading existing OQL, treat these as a latent bug: the clause cannot correctly handle error responses.

For a clean error-handling contract, always branch on the response's status field BEFORE parsing the body — do not call `json-stringify` on the body and `get-in` into it unconditionally.
