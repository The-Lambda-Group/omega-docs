[< Home](../README.md) | [How-to](README.md)

# Triage `omega/query/no-return`

`omega/query/no-return` is a single error name that covers two structurally different failure modes. The doc you actually need depends on which mode you are in. If you reach for the wrong one, you will edit code that is not the cause and conclude the impl is broken when the impl never even ran.

> If you have not already, read [Narrow an OQL failure](narrow-an-oql-failure.md) for the general methodology. This doc is the no-return-specific branch.

## The two modes

| Mode | Trigger | Where the call is | First check |
| --- | --- | --- | --- |
| **Scratch-file no-return** | A scratch query (`qo run <file>` / `qo query`) that binds `Result` but never calls `(return [Result])` | Top-level term sequence, no clause wrapping | Add `(return [...])` -- see [Write scratch queries](write-scratch-queries.md) |
| **Dispatch no-return** | A dispatched method call (`qo run` against a stored impl, or any caller into the `Qo.Public.Api.Run/*` family) | Inside the dispatch chain (`run-page-by-id` -> `Qo.Run.Impl.Meth/run-page-by-id` -> `Qo.Impl.Meth/page-method`) | Check arity, Clauses map, protocol spec -- see below |

The diagnostic distinguisher is the **caller**: if you ran a scratch file, you are in the first mode. If you invoked a stored impl through dispatch, you are in the second.

## Dispatch no-return: the silent zero-solutions trap

When the dispatch chain `Qo.Public.Api.Run/run-page-by-id` -> `Qo.Run.Impl.Meth/run-page-by-id` -> `Qo.Impl.Meth/page-method` -> `(call Clause Exec Result)` cannot find a matching clause (because the caller's args list does not match the protocol spec function arity, or because the Clauses map keys the dispatcher is looking up do not exist), it does **not** throw a specific arity-mismatch error. It silently produces zero solutions, which bubble up the entire chain and surface at the outer caller as `omega/query/no-return`.

The error message is misleading. It looks like a return-statement problem, so an agent triaging it reaches for "add `(return [...])`." But the impl is a clause, not a scratch file -- there is no return statement to add. The body that the agent then starts bisecting is innocent: it never ran.

### The signal that you are in this mode

`:failed-at nil` in the error trace. A body-level failure has frames in `:failed-at`; a dispatch-layer failure has nothing to print there because no clause body ever entered evaluation. **`:failed-at nil` plus `omega/query/no-return` from a dispatched call is the dispatch-layer signature.**

(See [reading-oql-stack-traces](https://github.com/The-Lambda-Group/query-omega-oql/blob/master/docs/reference/reading-oql-stack-traces.md) for the full `:failed-at` shape and how to read frames when they *do* exist.)

### The four checks, in order

Run these against the impl that *should* have matched. The first that fails is your culprit.

#### 1. Caller-args / spec-arity mismatch

The protocol arity convention is **caller args = spec arity - 1**. The minus-one is `self`, which the spec lists but the caller does not pass.

For `qo run -a <json-args>`, this maps to:

| Protocol spec arity | What `-a` should be |
| --- | --- |
| 1 (e.g. `Installable.install` with args `["self"]`) | `-a '[]'` (or omit `-a` entirely) |
| 2 (e.g. `Query.run` with args `["self", "input"]` or `EventHandler.on-event` with args `["self", "event"]`) | `-a '[{}]'` |
| 3 | `-a '[{}, {}]'` |

When this is wrong, `Qo.Impl.Meth/page-method` cannot unify the caller's arity with the spec's arity, produces zero solutions, and you get no-return.

> Concrete reproduction (2026-04-26): stripping an `OnEvent` body to a single `(= Result {...})` predicate still produced no-return when `-a` was wrong. The body never ran -- dispatch never matched the clause.

For the underlying mechanism (why both sides of the unification have to agree before any clause can be selected), see the related explanation [qo-run-arity-mismatch-silent](https://github.com/The-Lambda-Group/query-omega-cli/blob/master/docs/explanation/qo-run-arity-mismatch.md) once that doc lands.

#### 2. Clauses map keys

The impl's Clauses map looks like:

```oql
(= Clauses {"method-name" {"<arity>" ClauseSym}})
```

Both keys are looked up by the dispatcher. The arity key is a JSON-stringified integer matching the protocol spec arity (e.g. `"1"` for `Installable.install`, `"2"` for `Query.run`). A wrong arity key (`"2"` for an arity-1 spec, or vice versa) gives the dispatcher nothing to call -- zero solutions, no-return.

The method-name key must also match the protocol spec function name exactly. Off-by-one in either key produces the same silent failure.

#### 3. Protocol spec format

The protocol spec must be:

```json
{"functions": [{"name": "...", "args": <arity>}]}
```

If the spec is malformed (missing `functions` key, wrong shape, args is a list instead of an integer), the dispatcher cannot read the spec arity to compare against the caller's arity. Same outcome: zero solutions.

For the structure of an impl `.oql` file (datastore declarations, clause definition, Clauses registration, file-level return), see [omega-ai-components/docs/how-to/write-implementation-impl-file](https://github.com/The-Lambda-Group/omega-ai-components/blob/master/docs/how-to/write-implementation-impl-file.md) once that gap is filled.

#### 4. Impl actually stored

Verify the impl is on the page you think it is. `qo describe <page-id>` should list it. If `qo push` did not commit the implementation (e.g. the push itself errored on the file-level return -- see [Write scratch queries](write-scratch-queries.md)), the dispatcher has no clause to call.

## Worked example

```bash
qo run /apps/foo/bar -p Qo.Proto.EventHandler -m on-event
# -> {"type":"omega/query/no-return","data":{...},":failed-at":null}
```

The agent first stripped the `on-event` impl body to:

```oql
(:- (OnEvent Exec Result)
    (= Result {"status" "ok"}))
```

Same no-return. The body cannot be wrong -- it is a single binding. So the failure is upstream of the body. `:failed-at nil` confirms dispatch-layer.

Check 1: protocol spec arity. `EventHandler.on-event` is `{"name": "on-event", "args": ["self", "event"]}` -> spec arity 2 -> caller args length 1. The command had no `-a` flag, so caller args length is 0. Mismatch.

Fix:

```bash
qo run /apps/foo/bar -p Qo.Proto.EventHandler -m on-event -a '[{}]'
# -> ok
```

The body never needed touching.

## When this doc does not apply

If your `:failed-at` is **not** nil -- it has frames -- the body did run and the no-return is from inside the body (e.g. a downstream term producing zero solutions and `Result` never getting bound). That is body-level triage:

- Use [Debug by returning early](debug-by-returning-early.md) to bind `Result` to a tuple of intermediate variables and find the bisection point.
- Common body-level cause: `fold` over zero upstream solutions does not bind its accumulator -- see the canonical empty-handling discussion in the engine reducers reference.

## Related

- [Narrow an OQL failure](narrow-an-oql-failure.md) -- general OQL triage methodology that this doc specialises for the no-return error class.
- [Write scratch queries](write-scratch-queries.md) -- the scratch-file mode of no-return (missing `(return [Result])`).
- [Debug by returning early](debug-by-returning-early.md) -- body-level no-return triage when `:failed-at` is non-nil.
- [reading-oql-stack-traces](https://github.com/The-Lambda-Group/query-omega-oql/blob/master/docs/reference/reading-oql-stack-traces.md) -- the `:failed-at` frame structure and the bound-vs-free convention for reading dispatch-side traces.
