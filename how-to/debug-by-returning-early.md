[< Home](../README.md) | [How-to](README.md)

# Debug an OQL query by returning early

When a query fails partway through — wrong HTTP status, missing row, mysterious unbound symbol — the fastest path to the root cause is to **comment out everything after the suspected failure point and return a snapshot of the intermediate state**. Re-run the query, inspect the snapshot, then un-comment one more clause and repeat.

## The recipe

1. Find the clause you suspect is broken.
2. Comment out every clause after it (including the original `Result` binding).
3. Bind `Result` to a **tuple of the variables you want to inspect**:

   ```oql
   (= Result [Var1 Var2 Var3])
   ```

4. Run the query. The solution table now shows one row per solution with the inspected values in columns.
5. Verify two things at once:
   - **Right number of rows** — does the count match what you expect?
   - **Right values in each row** — are all symbols bound to sensible data?
6. If the snapshot is clean, un-comment the next clause and re-run. If it's not, the break is at or before this point — move the snapshot earlier.

## Why a tuple, not `throw`

`throw` surfaces exactly one solution and aborts the query. A tuple bound to `Result` surfaces **every row in the solution table**, so you can verify you have the right number of solutions, not just the right shape of one of them. Multi-row visibility is usually what you actually need — most OQL bugs are "I have 5 rows when I expected 50" or "row 3 has a bad value."

Use `throw` only when you genuinely want a single-shot error with a tagged payload and you're sure there's just one solution. For everything else, prefer the tuple.

## HTTP debugging sub-recipe

When an HTTP call is returning an error, bind the **request map** alongside the response so you can see exactly what you sent:

```oql
(= Req {"url" Url "method" "POST" "headers" Headers "body" Body})
(http-request Req HttpStatus ResBody)

(= Result [Req HttpStatus ResBody])
; comment out everything below this line
```

Inspecting `Req` in the snapshot is how you catch the literal-symbol-leak bug (see [Symbols in Literal Arguments](../reference/built-ins.md#symbols-in-literal-arguments)) — if a literal symbol name like `"AgentLink"` leaked into the payload instead of its value, you'll see it immediately in the returned tuple.

## Iteration discipline

- **One change per run.** Comment out or un-comment a single clause at a time. If you move multiple clauses at once you lose the bisection.
- **Always include per-row identity in the tuple.** If each solution corresponds to a row keyed by e.g. `AgentLink`, put `AgentLink` first in the tuple so you can tell rows apart.
- **Don't delete the commented lines.** You need them to un-comment as you walk forward. Clean them up only after the bug is fixed.

**Note:** Scratch queries (not implementation files) require an explicit `(return [Result])` as the last line or you get a bare 500 — see [Write scratch queries](write-scratch-queries.md).
