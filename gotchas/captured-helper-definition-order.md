[< Home](../README.md) | [Gotchas](README.md)

# Captured Helpers Must Be Defined Before They Are Captured

## Symptom

A stored implementation pushes successfully, but `qo run` fails at runtime with:

```text
omega/jvm/IllegalArgumentException
No implementation of method: :run-with-args ... found for class: clojure.lang.Symbol
```

The deepest useful stack frame is usually a captured-helper call:

```oql
(call SomeCapturedHelper ... Result)
```

## Root Cause

The clause that captures `SomeCapturedHelper` was defined before `SomeCapturedHelper` itself.

This is a **forward capture**:

```oql
;; WRONG — ProcessPage captures RunPhase before RunPhase is defined
(:- (ProcessPage Exec Result) {"capture" [RunPhase]}
    (call RunPhase Exec Result))

(:- (RunPhase Exec Result)
    (= Result {"ok" "yes"}))
```

Push time may accept this shape, but the stored implementation can later treat the captured helper value as a plain symbol instead of a runnable clause. At runtime, `(call ...)` then reaches `clojure.lang.Symbol` instead of an implementation with `:run-with-args`.

## Why Agents Misdiagnose It

The stack points at the helper call site, so it is easy to blame:

- the helper body
- upstream LLM or data inputs
- the map payload passed into the helper

That is often the wrong layer. If the captured value itself is a symbol, changing the helper body will not move the failure.

## Triage Pattern

Use this when the error appears at `(call SomeCapturedHelper ...)` inside a stored implementation:

1. Find the deepest `(call SomeCapturedHelper ...)` frame in the stack.
2. Replace `SomeCapturedHelper`'s body with a constant result.
3. Push and run the stored implementation again.
4. If the same Symbol failure remains at the same `(call SomeCapturedHelper ...)` frame, stop debugging the helper body.
5. Check definition order: every helper listed in the caller's `{"capture" [...]}` vector must appear earlier in the file than the capturing clause.
6. Reorder helpers so captured helpers are defined first, then push and run again.

The key signal is: **constant-body substitution does not move the error**. That means the body is not the cause; capture resolution is.

## Prevention Checklist

- No forward captures. Every symbol in a `{"capture" [...]}` vector must be defined earlier in the file.
- Keep orchestration clauses below the helpers they capture.
- Before final `qo run` verification, do a static scan of each capture vector against definition order.
- When a scratch copy passes but the stored implementation fails, distrust the scratch result. Capture resolution must be verified through the stored `qo push` + `qo run` path.

## Fix Pattern

Reorder the file so helper definitions come first:

```oql
;; CORRECT — RunPhase is defined before the clause that captures it
(:- (RunPhase Exec Result)
    (= Result {"ok" "yes"}))

(:- (ProcessPage Exec Result) {"capture" [RunPhase]}
    (call RunPhase Exec Result))
```

No helper-body change is required if definition order was the problem.

## Related

- [Lexical scope](lexical-scope.md) — capture syntax, scope boundaries, and the rule that captured helpers are invoked with `call`.
- [with-table-if header must include captured clause symbols](with-table-if-capture-header.md) — another scope-boundary rule for captured clause symbols after capture has succeeded.
