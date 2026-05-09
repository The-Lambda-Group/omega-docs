[< Home](../README.md) | [How-to](README.md)

# Triage `omega/query/no-return`

`omega/query/no-return` is a single error name that covers structurally different failure modes. The doc you actually need depends on which mode you are in. If you reach for the wrong one, you will edit code that is not the cause and conclude the impl is broken when the impl never even ran -- or you will spend hours auditing registration metadata for a failure that was always inside the body.

> If you have not already, read [Narrow an OQL failure](narrow-an-oql-failure.md) for the general methodology. This doc is the no-return-specific branch.

## The three modes

| Mode | Trigger | Where the call is | First check |
| --- | --- | --- | --- |
| **Scratch-file no-return** | A scratch query (`qo run <file>` / `qo query`) that binds `Result` but never calls `(return [Result])` | Top-level term sequence, no clause wrapping | Add `(return [...])` -- see [Write scratch queries](write-scratch-queries.md) |
| **Dispatch-layer no-return** | A dispatched method call where the dispatcher could not select a clause to run | Inside the dispatch chain (`run-page-by-id` -> `Qo.Run.Impl.Meth/run-page-by-id` -> `Qo.Impl.Meth/page-method`), body never entered | Check arity, Clauses map, protocol spec -- see Dispatch-layer below |
| **Body-level zero-solutions no-return** | A dispatched method call where the body ran but a term inside it produced zero solutions through unification (no exception thrown) | Inside the clause body, output variable (`Result`) never bound | Bisect the body -- see Body-level below |

The diagnostic distinguisher between the first mode and the other two is the **caller**: if you ran a scratch file, you are in the first mode. If you invoked a stored impl through dispatch, you are in one of the other two.

**The dispatch-layer and body-level modes share an identical envelope** — both surface as `omega/query/no-return` with `:failed-at nil`, the same misleading "Scratch queries require `(return [...])`" message, and no JVM stack trace. You cannot tell them apart from the response alone. The discipline is to walk the dispatch-layer checks first (cheaper), and if all pass, drop into body bisection. See [omega-db/docs/explanation/dispatch-layer-no-stack-trace.md](https://github.com/The-Lambda-Group/omega-db/blob/master/docs/explanation/dispatch-layer-no-stack-trace.md) for why both modes share the envelope.

## Dispatch-layer no-return: the silent zero-solutions trap

When the dispatch chain `Qo.Public.Api.Run/run-page-by-id` -> `Qo.Run.Impl.Meth/run-page-by-id` -> `Qo.Impl.Meth/page-method` -> `(call Clause Exec Result)` cannot find a matching clause (because the caller's args list does not match the protocol spec function arity, or because the Clauses map keys the dispatcher is looking up do not exist), it does **not** throw a specific arity-mismatch error. It silently produces zero solutions, which bubble up the entire chain and surface at the outer caller as `omega/query/no-return`.

The error message is misleading. It looks like a return-statement problem, so an agent triaging it reaches for "add `(return [...])`." But the impl is a clause, not a scratch file -- there is no return statement to add. In the dispatch-layer mode, the body never entered evaluation; in the body-level mode (covered next), the body ran but produced no result.

### The shared signal -- and why it is not a distinguisher between dispatch and body

`:failed-at nil` plus `omega/query/no-return` is the signature of "zero solutions reached the outer query boundary without any exception having been thrown." That includes the dispatch-layer failure described here **and** the body-level zero-solution failure described in the next section. A body-level failure has frames in `:failed-at` only when something inside the body **threw** an exception (e.g. `find-docs` throwing `omega/query/full-scan`, a JVM `NullPointerException`, an explicit `(throw ...)`). A body-level failure where every term simply produced zero solutions through unification has no exception and therefore no `:failed-at` frame -- same envelope as the dispatch case.

**Operational rule:** `:failed-at nil` plus `omega/query/no-return` means walk the four dispatch-layer checks first (they are cheaper), and if all four pass, drop into body bisection. Do not assume the body never ran -- verify dispatch first, then verify body.

(See [reading-oql-stack-traces](https://github.com/The-Lambda-Group/query-omega-oql/blob/master/docs/reference/reading-oql-stack-traces.md) for the full `:failed-at` shape and how to read frames when they *do* exist. See [omega-db/docs/explanation/dispatch-layer-no-stack-trace.md](https://github.com/The-Lambda-Group/omega-db/blob/master/docs/explanation/dispatch-layer-no-stack-trace.md) for the engine-side reason both modes share the envelope.)

### The four dispatch-layer checks, in order

Run these against the impl that *should* have matched. The first that fails is your culprit. If all four pass, the failure is body-level — drop into the bisection section below (#5).

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

For the structure of an impl `.oql` file (datastore declarations, clause definition, Clauses registration, file-level return), see [omega-ai-components/docs/how-to/write-implementation-impl-file](https://github.com/The-Lambda-Group/omega-ai-components/blob/master/docs/how-to/write-implementation-impl-file.md).

#### 4. Impl actually stored

Verify the impl is on the page you think it is. `qo describe <page-id>` should list it. If `qo push` did not commit the implementation (e.g. the push itself errored on the file-level return -- see [Write scratch queries](write-scratch-queries.md)), the dispatcher has no clause to call.

#### 5. Body-level no-result bubbling to the dispatch boundary

If checks 1–4 all pass and the no-return is still occurring, the failure is **not** in the dispatch layer — it is body-level. Some line in the clause body is producing zero solutions, `Result` never binds, and the dispatch boundary surfaces that as `omega/query/no-return` with `:failed-at nil` (the same envelope as the four dispatch-layer causes — the engine cannot tell them apart from the response shape alone). The reason `:failed-at` is still nil is that **no exception fired** — failed unification in OQL produces an empty relation, not a `Throwable`, so there is nothing for `run-exp` to attach a frame to.

Body-level causes the envelope hides include:

- An undefined clause call (typo in a fully-qualified name, stale namespace after a rename, an upstream API never deployed to the target host). OQL does not throw on missing predicates -- a missing predicate is an empty relation.
- A clause that resolved but returned zero solutions for the given args (common with pkey-style lookups: `row-index`, `prop-vals`, single-row gets where the input matches no row).
- An output variable never bound on some control-flow path (a `with-table-if` branch that succeeds but doesn't assign `Result`; a binding step gated behind a condition that didn't hold).
- `fold` over zero upstream solutions does not bind its accumulator.

**Triage: bisect the body.** Strip the body to a constant `Result`-bind that is known to dispatch successfully:

```oql
(:- (Method Exec Result)
    (= Result {"status" "ok"}))
```

Re-push and re-run. This must dispatch -- if it doesn't, you are in dispatch-layer mode after all (re-walk checks 1–4). If it does dispatch, add the body's lines back one at a time. The first line whose addition reintroduces the `:failed-at nil` envelope is the failure site. Once located, debug that single term as a minimum repro -- run the failing call from a scratch file with the same args and inspect the result. See [Narrow an OQL failure](narrow-an-oql-failure.md) for the methodology and [Debug by returning early](debug-by-returning-early.md) for the tuple-snapshot recipe to inspect intermediate state.

**Empirical case (2026-04-26).** A Seed-protocol impl was bisected from a working baseline:

| Body shape | Outcome |
|---|---|
| `(:- (Seed Exec Result) (= Result {"status" "ok"}))` | Dispatches successfully |
| Above plus a Manager-page HTML write block (`page-by-id` + `add-or-get-by-name` + `set-content`) | Dispatches successfully |
| Above plus a `write-table` call (rows actually written) | Dispatches successfully |
| Above plus a single `(Qo.Public.OqlApi.Db.Prop/row-index InstrTableId ["0" "manager"] MPageId)` call | `omega/query/no-return :failed-at nil` |

Dispatch infrastructure was working throughout. The trigger was the `row-index` call -- whether due to resolution, args, or downstream binding, the envelope did not say. Bisection located the line; the line was then debugged as a minimum repro. The 3-hour audit of registration metadata that preceded this bisection was wasted effort -- the dispatch checks 1–4 had all passed.

> **Do not pattern-match against a "captured-helper rule."** Earlier docs claimed `(call CapturedHelper ...)` from a captured orchestrator silently fails to dispatch and that swapping to `(run-term ...)` was the fix. **That diagnosis was wrong.** Bisection in the same session showed both `(call ...)` and `(run-term ...)` failed identically when the body contained a particular `row-index` call, and a flat body with no helpers failed the same way. The captured-helper distinction was not the structural difference. See [query-omega-oql/docs/explanation/call-vs-run-term-on-captured-helpers.md](https://github.com/The-Lambda-Group/query-omega-oql/blob/master/docs/explanation/call-vs-run-term-on-captured-helpers.md) (Correction section) for the bisection evidence.

For the engine-side framing of why both classes (dispatch-layer and body-level zero-solutions) share the `:failed-at nil` envelope, see [omega-db/docs/explanation/dispatch-layer-no-stack-trace.md](https://github.com/The-Lambda-Group/omega-db/blob/master/docs/explanation/dispatch-layer-no-stack-trace.md).

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

Same no-return. The body cannot be wrong -- it is a single binding that always succeeds. So the failure is upstream of the body. `:failed-at nil` is consistent with dispatch-layer (and also with body-level zero-solutions, but the body here cannot produce zero solutions). Walk the dispatch-layer checks.

Check 1: protocol spec arity. `EventHandler.on-event` is `{"name": "on-event", "args": ["self", "event"]}` -> spec arity 2 -> caller args length 1. The command had no `-a` flag, so caller args length is 0. Mismatch.

Fix:

```bash
qo run /apps/foo/bar -p Qo.Proto.EventHandler -m on-event -a '[{}]'
# -> ok
```

The body never needed touching.

## When `:failed-at` has frames

If your `:failed-at` is **not** nil -- it has frames -- the body ran *and a term inside it threw an exception* (e.g. `find-docs` throwing `omega/query/full-scan`, a JVM `NullPointerException`, an explicit `(throw ...)`). This is a different failure class from the `:failed-at nil` cases above. The frames pin the throwing term; triage that exception directly rather than treating it as a no-return:

- Read the topmost frame; identify the term that threw.
- Use [Debug by returning early](debug-by-returning-early.md) to bind `Result` to a tuple of intermediate variables and confirm the path that reaches the throwing term.
- Triage the exception itself per its `type` (e.g. `omega/query/full-scan` -> see [no-mango-by-default](https://github.com/The-Lambda-Group/omega-db/blob/master/docs/explanation/no-mango-by-default.md); `omega/jvm/...` -> a host-side bug, see [native-jvm-exceptions](https://github.com/The-Lambda-Group/omega-db/blob/master/docs/explanation/native-jvm-exceptions.md)).

A body that produces zero solutions through pure unification failure (no exception fired) does **not** end up here -- it ends up in the body-level zero-solutions mode covered in check 5 above, with `:failed-at nil` like the dispatch-layer cases.

## Related

- [Narrow an OQL failure](narrow-an-oql-failure.md) -- general OQL triage methodology that this doc specialises for the no-return error class.
- [Write scratch queries](write-scratch-queries.md) -- the scratch-file mode of no-return (missing `(return [Result])`).
- [Debug by returning early](debug-by-returning-early.md) -- tuple-snapshot recipe for inspecting intermediate body state, used by check 5 (body-level bisection) and by the exception-throwing body case.
- [reading-oql-stack-traces](https://github.com/The-Lambda-Group/query-omega-oql/blob/master/docs/reference/reading-oql-stack-traces.md) -- the `:failed-at` frame structure and the bound-vs-free convention for reading dispatch-side traces.
- [call-vs-run-term-on-captured-helpers](https://github.com/The-Lambda-Group/query-omega-oql/blob/master/docs/explanation/call-vs-run-term-on-captured-helpers.md) -- `run-term` vs `call` solution-scoping (per-row freshness, partition-before-fold). Includes the Correction section retracting the misdiagnosed "captured-helper dispatch" rule that earlier versions of this doc and that explanation propagated.
- [dispatch-layer-no-stack-trace](https://github.com/The-Lambda-Group/omega-db/blob/master/docs/explanation/dispatch-layer-no-stack-trace.md) -- engine-side companion: why both dispatch-layer and body-level zero-solution failures share the `:failed-at nil` envelope and have no JVM trace (zero solutions are not exceptions, so there's no `Throwable` and no frame to attach).
