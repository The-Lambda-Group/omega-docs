# Omega Documentation

> **Confidentiality: PUBLIC.** All content in this repository is public platform documentation. Private business information lives in the `omega-knowledge-base` (local-only, never pushed).

Public documentation for the [Omega](https://github.com/The-Lambda-Group) platform, organized as a Diataxis tree. Every top-level bucket indexes its own entries -- start here, then descend into the bucket that matches your question type.

## Methodology (read first)

One doc encodes the entire OQL loop — authoring and triage in both directions. It is methodology, not a peer of task-specific how-tos -- read it before descending into the buckets below:

- [how-to/develop-oql.md](how-to/develop-oql.md) -- The OQL operating manual. Covers the probe-don't-build mindset, the `Result`-as-inspection-channel mechanic, the authoring loop (forward: push → run → verify one transformation at a time), the triage loop (backward: six artifact-producing rules for diagnosing failures without shipping a pattern-matched non-fix), LLM-specific failure-mode framing, the wall-clock-as-cardinality-signal heuristic, the safe restore pattern (use a minimal OnEvent stub between plan tasks instead of the production OnEvent to avoid silent LLM calls or DB writes), and two worked examples. Read before writing **or** debugging any OQL. Contains an "Operating manual (re-read every session)" 8-rule TLDR at the top — if you only have time for one thing, read that section.

## Navigation map

| Bucket | Question type | What you will find |
|--------|--------------|-------------------|
| [how-to/](how-to/README.md) | "How do I ..." | Step-by-step task recipes for OQL development workflows, debugging techniques, and common coding patterns. Covers the incremental push-run cycle, early-return debugging, data-table integrity checks, scratch query structure, batch-size testing, row-drop idioms, and HTTP dispatch handling. |
| [reference/](reference/README.md) | "What is ..." / "Where is ..." | Exact signatures, syntax definitions, protocol specifications, and workspace conventions. Covers OQL clause syntax, all built-in terms (arithmetic, random, maps, arrays, strings, HTTP), control flow semantics, datastore/functor/defterm definitions, reducer operations, query execution constraints, public API clause tables, protocol IDs with JSON specs, and the `Docs/` folder convention for synced project doc trees. |
| [explanation/](explanation/README.md) | "How does X work" / "Why is ..." | Design rationale, architecture, and conceptual models. Covers what Omega is, its philosophy and intellectual lineage, the OQL language model (terms, unification, maps), the execution model (solution sets, joins, clause boundaries, performance), the page tree and block types, the component/protocol/implementation architecture, the plugin lifecycle with actor-model event flow, the discipline for shaping multi-task agent-step plans (flat-sequential helper extraction, orchestrator review, plan-authorship evidence-before-assertion — OQL code in plan tasks must be cargo-culted from existing impls, not recalled from training data), the three write-half patterns every workflow agent step must pick correctly (uniqueness validation against a source array, text-column vs HTML-block writes, agent-scoped Memory cold-start gates), the strategy for unit-testing `qo` component libraries (generalising the proven `test-llm-resp` substitution precedent to all side-effect boundaries, scratch-queries-with-a-convention as the test runner, complementing rather than replacing push-run-verify), and the README convention for making OmegaAI page trees navigable by agents without guessing (README at every folder level, 2-README routing rule, the OmegaAI equivalent of the KB's "Every README must route correctly" rule). |
| [gotchas/](gotchas/README.md) | "Why did X fail" | Failure modes and debugging patterns for the public OQL language surface. Covers lexical scope rules (push-time vs runtime, symbol renaming, datastore survival), boolean/return-value/index-matching conventions that cause silent failures, public-API surface traps (path-style page lookups that force a full scan, helper APIs that don't exist as named — `page-by-name`, `child-page-path`), and environment/tooling footguns (`omega-cli` defaulting to `localhost:3001` while `qo` reads `OMEGA_URL` from `.env`, producing silent zero-solutions on scratch queries). |

## Information ownership

This repo is the **public root** of a two-root documentation system:

- **This repo owns:** public platform documentation -- OQL language reference, public API signatures, platform architecture explanations, development workflows, and public-surface gotchas. Anything a user or agent needs to understand and use the Omega platform.
- **Private root (`omega-knowledge-base`) owns:** company-private information -- business strategy, customer negotiation, stakeholder dynamics, cross-project coordination, and project-specific internal details. Agents should start navigation at the private root and descend from there.
- **Project repos own:** project-specific technical docs that live in each project's own `/docs/` tree (e.g., `query-omega-cli/docs/` for CLI command reference).

## If you are an agent

This repo is the public half of the Omega knowledge base. Start at the private root at `~/Development/omega/omega-knowledge-base/README.md` and descend from there -- the private root links back into this repo. The taxonomy is described in detail at `omega-knowledge-base/contributor/reference/diataxis.md`.

## Related

- [query-omega-cli commands](https://github.com/The-Lambda-Group/query-omega-cli/blob/master/docs/reference/commands.md) -- `qo` command reference (lives in that repo's `/docs/` tree)
- [MCP Server](https://github.com/The-Lambda-Group/query-omega-mcp)
