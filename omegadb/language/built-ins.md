[< Language](../language.md) | [OmegaDB](../../README.md#omegadb)

# Built-in Terms

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

**`json-stringify` is bidirectional.** When the second arg is a symbol, it stringifies the first (object → JSON string). When the first arg is a symbol, it parses the second (JSON string → object). The existing query implementations rely on both directions.

## HTTP

**`http-request`** is the general-purpose HTTP term. It takes a request map and returns a response map. The request map is passed directly to the underlying HTTP client ([clj-http](https://github.com/dakrone/clj-http)), so all clj-http options are supported.

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

These still work but prefer `http-request` for new code:

```oql
;; Deprecated — use http-request with "method" "get" instead
(http-get-request Url {"headers" Headers} Res)
(http-post-request Url {"headers" Headers "body" Body} Res)
(http-delete-request Url {"headers" Headers} Res)
```

These take `(Url Config Res)` instead of a single request map. They do not support `query-params` — you must build the URL manually.
