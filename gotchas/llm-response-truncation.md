[< Home](../README.md) | [Gotchas](README.md)

# LLM Response Truncation in call-llm

## Symptom

A `call-llm` invocation fails with:

```
omega/jvm/EOFException: JSON error (end-of-file inside string)
```

The stack trace points at `(json-stringify ParsedResponse RawText)` inside the `call-llm` component. Wall-clock to the throw is typically 30s-2min — long enough that the LLM was clearly generating output, not failing fast on input.

The error reads like a JSON parse bug. It is not. The LLM emitted a response that was cut off mid-string (before the closing braces and quotes that would make the payload valid JSON), so `json-stringify` runs out of input while still inside an open string and throws `EOFException`.

## Root Cause

LLM output token ceiling. Most models cap a single response at ~4-8k tokens (some higher, none unlimited). When the prompt asks for a large fan-out response — many entries × verbose per-entry payload, especially when each entry is HTML or other token-heavy markup, plus any memory blob the prompt threads in — the response runs into the ceiling and is truncated mid-token. The truncated bytes are not valid JSON, so the in-component parse throws.

Discovered during MAB Strategist V1 cold start (2026-04-28). Strategist sent 27 unwritten `treatment_id`s to the LLM in a single call, expecting full HTML strategies for each. After 1m14s the response was truncated mid-string and `json-stringify` threw.

## Fix (in order of preference)

### 1. Batch the LLM call

Process N items per invocation and use a `has-more` return flag to drive caller-side re-invocation until the input list is drained. This is the canonical fix and aligns with the idempotent / incremental design philosophy: each invocation is small, deterministic, and independently re-runnable.

Implementation sketch:

- Pick the first N of the input list via index access (`(get InputList 0 X0) ... (get InputList N-1 X-N-1)`).
- Wrap the slicing inside a `with-table-if (>= count N)` gate so a short input list still works.
- Return `has-more` as `"true"` when more items remain, `"false"` when drained. Caller loops until `"false"`.

MAB Strategist V1 hardcodes N=5. Pick N empirically — small enough to stay under the ceiling for the worst-case per-item verbosity, large enough that the round-trip count stays reasonable.

### 2. Reduce per-item verbosity in the prompt

Add an explicit conciseness instruction. Example:

> Keep each strategy concise (3-6 sentences in 1-2 short paragraphs) — total response output should fit comfortably within standard JSON output limits.

Cheaper than batching but only delays the problem. Use alongside batching, not instead of it.

### 3. Set explicit max_tokens (only if call-llm exposes it)

If the `call-llm` invocation accepts a max-tokens parameter, raising it can buy headroom — but the underlying issue is still that the prompt asks for too much in one shot. This is can-kicking, not a fix.

## Idempotency Note

The throw happens *before* the parse returns to the caller. There are no partial writes from the truncated response. Callers using deterministic input filters (e.g., MAB's "unwritten treatment ids") get clean retry semantics on the next invocation — re-running picks up the same input set and tries again. This is what makes the batched fix safe: if a batch happens to truncate, no state is corrupted, and a smaller batch on the next run will succeed.

## Status

Resolved. The batched-LLM pattern with `has-more` is the canonical workaround. No engine-side change is planned — the truncation is a property of the LLM, not of `call-llm`.

## Related

- [add-or-get-by-name binds the same block across multiple solution rows](add-or-get-by-name-multi-row.md) — another runtime gotcha whose error surface (a `KEY_UPDATE_CONFLICT` write) does not read as the actual root cause (universal quantifier over solution set). Same general lesson: read the error site as a clue, not a verdict.
- [call inside with-table-if scales poorly](call-inside-with-table-if.md) — companion runtime-limitation gotcha. Both use bounded-batch sizing as the workaround.
