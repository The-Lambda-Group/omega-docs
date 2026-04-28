[< Home](../README.md) | [Gotchas](README.md)

# json-stringify Parse Direction Is Strict — Rejects Raw Control Characters

## Symptom

A `json-stringify` parse-direction call (first arg unbound, second arg bound to a JSON string) fails with:

```
omega/jvm/Exception: JSON error (missing entry in object)
```

or

```
omega/jvm/Exception: JSON error (invalid array)
```

The error site is typically the internal `(json-stringify ParsedResponse RawText)` step inside a `call-llm` consumer, but it can occur anywhere a parse is run on a string the agent does not control end-to-end. The error message points at a structural symptom (missing entry, invalid array) rather than the actual byte-level cause, which makes the failure look like a payload-shape bug when it is really an encoding-strictness bug.

The same byte-for-byte string parses cleanly under Python's `json.loads`. Round-tripping internal OQL data through stringify-then-parse always works — the failure mode is only triggered by externally produced JSON, with **LLM output the dominant source in practice**.

This is **not** the truncation gotcha. Truncation throws `omega/jvm/EOFException: JSON error (end-of-file inside string)` and is caused by a mid-string cutoff at the LLM token ceiling — see [LLM response truncation in call-llm](llm-response-truncation.md). The strict-parse gotcha throws a *non*-EOF `JSON error` on a fully-formed response.

## Root Cause

OQL's `json-stringify` parse direction is implemented over `clojure.data.json`, which is RFC 8259 strict by default. RFC 8259 §7 says: "All Unicode characters may be placed within the quotation marks, except for the characters that MUST be escaped: quotation mark, reverse solidus, and the control characters (U+0000 through U+001F)." The parser enforces this — a literal `\n` (U+000A) or `\t` (U+0009) byte inside a JSON string value is rejected.

Python's `json.loads` is lenient by default and silently accepts raw control characters in string values. Most ad-hoc JSON validators (browser dev tools, `jq`, online formatters) follow Python here. The result: a payload that "looks fine" everywhere else can fail OQL's parser specifically.

LLM output is the empirical hot spot. Models routinely emit raw newlines and tabs inside JSON string values rather than the escape sequences `\n` and `\t` — especially in long, multi-paragraph fields like email body text, HTML blocks, or strategy narratives. The stricter the JSON shape the prompt asks for, the more often this surfaces.

### Empirical repro

Discovered during the MAB Copywriter V1 drain (2026-04-28). A 10,811-character LLM response (complete — `stop_reason: end_turn`, 2,771 of 8,000 tokens used, no truncation signal) caused `json-stringify` to throw `JSON error (missing entry in object)` while Python's `json.loads` parsed the same bytes cleanly. The diagnostic captured the failing payload to `/tmp/copywriter-debug-output.txt` for byte-level inspection. (That file is a session artifact and may not survive cleanup; the shape was a Copywriter response containing JSON-formatted email-body fields with embedded raw newlines from the LLM's prose paragraphs.)

The shape characteristics worth recognizing:

- Multi-kilobyte payload with verbose string fields (paragraph-length copy, HTML blocks).
- Fully-formed top-level JSON envelope (`{...}` matched, no truncation).
- Visible "looks like newlines" in the raw bytes when inspected with `cat -A` or a hex dump — `\x0A` rather than `\\n`.
- Parses fine in Python and in browser dev tools; fails only on the OQL parser.

## Fix

**The fix lives in the prompt, not in pre-cleaning the string.** Tell the LLM, in the system or user prompt, to emit escape sequences (`\n`, `\t`, `\"`, `\\`) inside JSON string values rather than raw control characters. This puts the data-cleaning at the source where it belongs — at the data-producing boundary, not at the consumer.

Concretely, add an instruction like:

> Inside any JSON string value, emit `\n` (backslash-n) for newlines, `\t` (backslash-t) for tabs, and `\"` for quotation marks. Do NOT emit raw newline (0x0A), raw tab (0x09), or other unescaped control characters inside string values. The output must be valid per RFC 8259 (strict).

For agents whose sibling agents (e.g. MAB Strategist, Researcher) parse identical-shape responses successfully, the diagnostic is also "what is different about *this* prompt that produced control-char-laden output" — usually it's a longer-form string field (multi-paragraph copy, HTML) where the model defaulted to raw newlines for human-readability.

### Do NOT pre-clean the string

Do not `string-replace` raw newlines into `\n` before parsing. Do not regex-scrub control characters. The reasons are documented in detail at [LLM output discipline](../explanation/llm-output-discipline.md), but in summary:

1. String-cleaning hides the real incompatibility — every new edge case (raw tab, unescaped quote, lone backslash) needs a new replace, and the list never converges.
2. Data should be clean at the boundary where it is produced, not scrubbed at consumption.
3. If the same parser works for sibling agents, the divergence is the prompt, and the prompt is the fix point.
4. If the parser is the actual problem (see Engine-level future work below), filing a parser-bug gap is the correct escalation — not papering over with replaces.

## Engine-level future work

The OQL parse path could be made more lenient to match the Python / `jq` / browser superset. Concretely: switch the parse direction of `json-stringify` to call `clojure.data.json/read-str` with options equivalent to Python `json.loads` defaults — accept raw control characters in string values, accept the optional `\/` solidus escape, and produce a byte-offset and surrounding-context excerpt in the error message instead of the bare structural label.

Until that lands, the agent-side rule above (fix the prompt, do not pre-clean) is the only correct workaround. This gotcha file doubles as the engine work-item record — when the parser is updated, this doc should be revised to describe the new behavior and link to the engine PR.

## Related

- [LLM response truncation in call-llm](llm-response-truncation.md) — the *other* `json-stringify` failure inside `call-llm`, distinguished by `EOFException` and an end-of-file-inside-string message. Same error site, different root cause (token-ceiling truncation vs strict-parse rejection of valid-but-control-char-laden output). Read both when triaging a `call-llm` JSON failure to identify which mode you are in.
- [`json-stringify` reference](../reference/built-ins.md) — language-reference home for the bidirectional parse/stringify built-in (see the `json-stringify` section under String Operations). The reference documents the binding-direction rule but does not currently document the strict-parse limitation; this gotcha is the limitation note.
- [LLM output discipline](../explanation/llm-output-discipline.md) — agent-side rule that data must be clean at the boundary (the prompt), not scrubbed at consumption. Read this before reaching for any pre-parse `string-replace`.
