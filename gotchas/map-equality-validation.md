[< Gotchas](README.md)

# Direct Map Equality Is Not a Safe Validation Primitive for Key Sets

> **Confidentiality: PUBLIC.** All content in this file is public platform documentation.

When you need to validate that two OQL result-key sets are the same, do **not** rely on direct map equality as the proof step. During MAB Critic validation work on 2026-05-04, a probe comparing two small maps gave a misleading "equal" result at the point where the validator needed a stricter key-set check. The shipped fix avoided direct map equality and instead validated against **sorted unique key lists**.

This is an **empirical gotcha**, not a confirmed engine-semantics statement. The observed behavior is enough to make direct map equality unsafe for validator design, but the underlying equality semantics for maps are **Unknown**.

## Why this matters

Key-set validation is usually trying to prove an exact match:

- every requested key was returned
- no extra key was returned
- duplicate rows do not accidentally count as a successful match
- the empty case is handled explicitly

Direct map equality is the wrong surface for that proof. Even if two maps "look like sets", the validation step becomes opaque: you are depending on undocumented map-equality behavior rather than on a normalized representation you can inspect.

## Safe validation surface: sorted unique key lists

Normalize both sides to the same representation, then compare that representation:

1. Build the key list you actually care about.
2. Remove duplicates so the comparison is duplicate-neutral.
3. Sort deterministically so order stops mattering.
4. Compare the normalized key lists, not the maps.

In the MAB Critic fix, the validator compared the requested page keys and the LLM-returned critique keys only after both had been converted to sorted unique key lists.

## Recommended pattern

If your input already starts as key arrays, keep the validator on arrays all the way through.

If your input starts as rows or maps:

1. Extract the key string or tuple per row.
2. Deduplicate into a set-like structure.
3. Materialize the deduped keys back into a list.
4. Use `with-order-by` to make the list deterministic.
5. Compare the normalized lists as your equality check.

The load-bearing idea is not the exact helper names. The load-bearing idea is: **normalize to sorted unique keys before comparing**.

## What not to do

```oql
;; Avoid using direct map equality as the validator proof step.
(= InputSet OutputSet)
```

That shape was the original MAB Critic plan, and it was replaced after debugging showed that small-map equality probes were not a trustworthy validation primitive for this use case.

## Scope of the claim

What is known:

- A validation probe for small maps was misleading in live OQL work on 2026-05-04.
- The safe fix was to compare sorted unique key lists instead.
- This pattern handles empty and duplicate-neutral validation more explicitly than map equality.

What is **Unknown**:

- The precise engine semantics for map equality.
- Whether the misleading behavior is universal or limited to certain probe shapes.
- Whether the behavior is specific to one-entry maps or to a broader class of map comparisons.

Until the engine semantics are documented, treat direct map equality as unsafe for key-set validation.

## Cross-references

- [Built-in terms](../reference/built-ins.md) -- map and array operations (`merge`, `select-keys`, `get-list`, `append`)
- [Solution table operations (reducers)](../reference/reducers.md) -- `fold`, `with-order-by`, and dedupe-adjacent reducer patterns
- `~/Development/wttc/mab/docs/superpowers/retrospectives/2026-05-04-critic-full-page-audit-report.md` -- recorded the observed issue and the sorted-key-list fix
