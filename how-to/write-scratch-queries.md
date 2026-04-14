[< Home](../README.md) | [How-to](README.md)

# How to write scratch queries

Scratch queries are bare top-level term sequences with no clause mechanism. They are run with `qo run` or `qo query`.

## Always end with `(return [Result])`

Unlike implementation files (where clauses return via the head binding), scratch queries have no implicit return path. If you bind `Result` but never call `(return [Result])`, the query runner produces no result and the API returns a bare HTTP 500 with no error message, no stack trace, and nothing actionable.

```oql
;; WRONG — bare 500, no diagnostics
(= Result "hello")

;; RIGHT
(= Result "hello")
(return [Result])
```

## Why the bare 500 is hard to debug

The 500 has no details, so the natural debugging path is: "what specifically failed?" leads to checking error shapes, then datastore paths, then reading more code. The actual problem — a structurally incomplete query — is never on that path because the query *looks* valid. You have to step back to "is my query structurally complete?" which is a basic language concern, not a runtime concern.

If you get a bare 500 from a scratch query, check for `(return [...])` first.
