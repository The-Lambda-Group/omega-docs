[< Home](../README.md) | [How-to](README.md)

# Handle HTTP 422 responses in status dispatch

How to prevent 422 (and other unexpected HTTP status codes) from leaking through `with-table-if` dispatch branches and causing `WRITING_SYMBOLS` errors.

## The problem

A dispatch that explicitly handles 2xx and 400 doesn't match 422. A `with-table-if (or (<= 200 HttpStatus 299) (= 400 HttpStatus))` branch leaves 422 (and every other non-200/400 status) unmatched. Those rows fall through to the downstream table with their branch-scoped symbols unbound.

`write-table` then tries to materialize those rows, can't resolve the unbound symbols, and errors with `WRITING_SYMBOLS`.

## The fix

Add an explicit 422 filter *before* the write gate, using the [row-drop idiom](drop-rows-with-forced-unification-failure.md):

```oql
; ... http-request produces HttpStatus ...

(with-table-if [AgentLink HttpStatus] (= HttpStatus 422)
  (= true false))  ; drop 422 rows

; ... normal 2xx / 400 dispatch ...

(write-table ...)
```

The 422 rows are dropped before they reach the dispatch, so they never leak into the success path.

## General rule

Any dispatch on HTTP status **must** either:

1. Explicitly enumerate every status your API can return (2xx, 4xx known, 4xx unknown, 5xx), and handle each with either a write or a drop.
2. Have a catch-all "drop every row where HttpStatus is not 200" filter before the dispatch.

## How to diagnose `WRITING_SYMBOLS` errors

1. Check the failing rows -- are any of their symbols unbound?
2. If yes, trace back to where those symbols should have been bound. Usually that's a `with-table-if` branch.
3. If the branch condition doesn't match those rows, you have a dispatch gap. Add a drop.
