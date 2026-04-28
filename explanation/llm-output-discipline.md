[< Home](../README.md) | [Explanation](README.md)

# LLM output discipline — never scrub at the consumption boundary

When an LLM-driven agent step pipes a model response into OQL — typically through `(json-stringify Parsed RawText)` to land structured data into the solution set — the response sometimes won't parse. The reflexive instinct is to add a `string-replace` or regex-clean pass *before* the `json-stringify` call, normalising the malformed bytes into something the parser will accept.

**Don't.** This doc explains why string-scrubbing at the consumption boundary is the wrong response, and what the right two responses are.

This is a meta-discipline doc — sibling to [agent-step-plan-shape.md](agent-step-plan-shape.md). It is not about how to call `json-stringify`; it is about how to react when `json-stringify` rejects an LLM payload.

## The rule

If an LLM response can't be parsed by OQL's `json-stringify`:

1. **Do not** insert a pre-parse cleaning step (`string-replace`, regex normalisation, character stripping, etc.) into the implementation.
2. **Do** fix the producer — tighten the LLM prompt so the model emits parser-clean output the first time.
3. **Or** file a parser-bug gap (see [gap-json-stringify-strict-parse](#related)) if the producer can't reasonably be tightened, and stop. Don't ship a workaround in the meantime.

That's it. Two acceptable moves, both at the boundary that owns the data shape. No scrubbing layer in between.

## Why scrubbing is wrong

Four reasons, in order of how visible the damage is:

### 1. It hides a real interface mismatch

`json-stringify` (parse direction) failing on an LLM response is a signal: the producer and the consumer disagree on the wire format. That's diagnostic. It tells you exactly where to look — at the prompt that produced the malformed payload, or at the parser that's stricter than expected.

A `string-replace "\\n" "\n"` step — or whatever the patch happens to be — silences the signal. The implementation now appears to work, but the underlying disagreement is unaddressed and will resurface the next time the LLM emits a slightly different malformed payload.

### 2. It doesn't generalise

Each scrub pattern handles exactly the bytes you've already seen fail. A new edge case — a raw tab character, an unescaped form feed, a stray BOM, a Windows line ending the model decided to emit — needs a new `string-replace` line. The cleaning layer becomes a drift-prone whack-a-mole that no one fully understands.

The producer fix doesn't have this property. "The LLM emits RFC-8259-compliant JSON" is one rule. Whether the next failure mode involves tabs or CRs or Unicode escapes, the rule is the same and the fix is the same: tighten the prompt's output-format instructions until the producer obeys.

### 3. It violates "clean at the boundary, never at consumption"

This is the deeper architectural point. Data shape is owned at one of two places:

- **The boundary** — where the data first enters the system (the LLM call site, an HTTP ingestion handler, a CSV loader). The boundary is responsible for emitting (or normalising) data into the system's canonical shape.
- **The producer** — the upstream system that the boundary calls into. If you control the producer, you can tighten it; if you don't, the boundary normalises.

The consumer is *not* one of those places. Once data is in the solution set, downstream terms assume it's already clean. Adding a scrubbing step at consumption inverts the responsibility model: now every consumer has to know which scrubs to apply, and any consumer that forgets to scrub gets corrupted data silently.

For LLM-driven agent steps the producer (the LLM via the prompt) is something you fully control. Tightening the prompt is the cheapest fix and lands the responsibility in the right place.

### 4. Sibling agents prove the prompt is the fix point

In a multi-agent workflow (MAB's Strategist / Researcher / Critic / Analyst / Execute), every agent step uses the same `json-stringify` parser on the same kind of LLM output. If one agent's responses fail to parse and the others' don't, the parser isn't the problem — that one agent's prompt is. The prompts that *do* produce parser-clean JSON are the existence proof: parser-clean output is achievable from this LLM, this parser, this pipeline. The failing agent's prompt needs to converge on whatever the working prompts already do.

If *every* agent's responses start failing on the same payload pattern, that's the parser-bug case (response 3 below) — file the gap and stop, don't ship a scrubbing patch under the table.

## What "tighten the prompt" looks like

When the parser rejects the LLM's payload, the producer-side fix is almost always one of:

- **State the wire format explicitly.** "Emit valid RFC 8259 JSON. Inside string values, all control characters MUST be escaped (`\n`, `\t`, `\r`, `\b`, `\f`). Do not emit raw newlines or tabs inside string values."
- **Show, don't tell.** Include a JSON exemplar in the system prompt that already handles the failure mode the way you want. LLMs imitate examples more reliably than they follow rules in prose.
- **Constrain the schema, not the prose.** Specify exact key names, value types, and array structures. The more underspecified the shape, the more variance in the output.
- **Reject the response and retry on parse failure.** If the LLM occasionally emits malformed output even with a tight prompt, the right response is a retry-on-parse-failure loop, not a scrub-and-hope layer. Retries land the responsibility back at the producer; scrubbing pretends the producer is fine.

The discipline is: every LLM-pipeline failure mode either has a prompt-tightening fix, OR is a real parser-side limitation that warrants a tracked engine gap. There is no third bucket called "preprocess the bytes."

## When to file a parser-bug gap instead

A small slice of failure modes are genuine parser limitations — the LLM output is valid by a reasonable standard (e.g., Python's `json.loads` accepts it, `jq` accepts it, the JavaScript `JSON.parse` accepts it) but OQL's parser rejects it. Symptoms:

- Multiple sibling agents hit the same failure shape.
- The malformed payload round-trips through other JSON parsers cleanly.
- Tightening the prompt is possible but increasingly contrived (the prompt becomes a list of "don't do X, don't do Y" defensive instructions for things every other JSON parser handles).

In that case the producer is fine and the parser is the divergence. File a gap (see [gap-json-stringify-strict-parse](#related)) capturing the empirical repro — the failing payload, what other parsers accept it, what error OQL produces — and **stop**. Don't ship a scrubbing workaround "until the engine fix lands." Workarounds calcify; the next agent assumes the scrub is load-bearing and propagates it.

If the workflow is genuinely blocked, the right move is to surface the block to the user. Not to paper over it.

## Worked failure mode — MAB Copywriter V1 drain (2026-04-28)

A diagnostic run of MAB's Copywriter agent step failed when its 10,811-character LLM response was piped into `(json-stringify Parsed RawText)`. The response was complete (`stop_reason: end_turn`, no truncation) and Python's `json.loads` parsed it cleanly. OQL's `json-stringify` rejected it with `omega/jvm/Exception "JSON error (missing entry in object)"`.

The reflexive subagent response was: "add `string-replace "\\/" "/"` (or some other normalisation) before `json-stringify`." The drain was blocked, the patch looked tactical, and the temptation was to ship it.

What actually happened:

1. The patch was rejected on the discipline grounds in this doc — scrubbing the bytes at consumption hides the real issue, doesn't generalise, and violates the boundary-ownership rule.
2. The Copywriter prompt was inspected against sibling agents (Strategist, Researcher) whose `json-stringify` calls worked fine on similar-volume LLM output. The Copywriter prompt's output-format section was looser — it didn't explicitly forbid raw control characters in string values.
3. The actionable next step was: tighten the Copywriter prompt's "emit valid RFC 8259 JSON, escape all control chars in strings" clause to match what Strategist's prompt already says.
4. Separately, the parser strictness itself was filed as `gap-json-stringify-strict-parse` — an engine-level work item to make OQL's parser accept the lenient superset Python's `json.loads` does. That fix is not a precondition for the agent step working; the prompt-tightening is.

Two responses, both at the right place. Zero `string-replace` lines added.

## Why this lives in explanation/

The "rule" is a one-liner; it would fit on a checklist. But the *force* of the rule lives in understanding why each of the four objections matters and recognising the situations they apply to. A new agent dispatched to "fix the parse failure" will reach for `string-replace` unless they've internalised that scrubbing at consumption is an architectural mistake, not a stylistic one. That internalisation is what `explanation/` exists to provide.

The how-to of "tighten an LLM prompt to emit parser-clean JSON" is generic prompt engineering — there's nothing Omega-specific to document there. The Omega-specific content is the rule itself and the architectural reasoning behind it, both of which belong here.

## Related

- [agent-step-plan-shape.md](agent-step-plan-shape.md) — sibling meta-discipline doc. The plan-shape doc covers structural discipline (flat-sequential helpers, OnEvent body shape); this doc covers data-pipeline discipline (where to fix malformed inputs).
- [reference/built-ins.md § json-stringify](../reference/built-ins.md) — bidirectional parse/stringify contract, the parse-probe pattern (named symbol vs `_`), and the "no `json-parse` exists" trap.
- [how-to/develop-oql-implementations.md](../how-to/develop-oql-implementations.md) — the push-run-verify discipline. When `json-stringify` fails on a real LLM payload, the run-verify step is where the failure surfaces; this doc explains why the next move is producer-side, not consumer-side.
- `gap-json-stringify-strict-parse` (in `omega-knowledge-base/gaps.md` if still open, otherwise resolved into `gotchas/json-stringify-strict-parse.md`) — the parser-side companion concern. Tracks the engine work to make OQL's JSON parser as lenient as Python's `json.loads` (raw control characters in strings, optional `\/` escapes, etc.).
