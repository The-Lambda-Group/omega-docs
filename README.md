# Omega Documentation

> **Confidentiality: PUBLIC.** All content in this repository is public platform documentation. Private business information lives in the `omega-knowledge-base` (local-only, never pushed).

Public documentation for the [Omega](https://github.com/The-Lambda-Group) platform, organized as a Diataxis tree. Every top-level bucket indexes its own entries -- start here, then descend into the bucket that matches your question type.

## Navigation map

| Bucket | Question type | What you will find |
|--------|--------------|-------------------|
| [how-to/](how-to/README.md) | "How do I ..." | Step-by-step task recipes for OQL development workflows, debugging techniques, and common coding patterns. Covers the incremental push-run cycle, early-return debugging, scratch query structure, batch-size testing, row-drop idioms, and HTTP dispatch handling. |
| [reference/](reference/README.md) | "What is ..." / "Where is ..." | Exact signatures, syntax definitions, and protocol specifications. Covers OQL clause syntax, all built-in terms (arithmetic, maps, arrays, strings, HTTP), control flow semantics, datastore/functor/defterm definitions, reducer operations, query execution constraints, public API clause tables, and protocol IDs with JSON specs. |
| [explanation/](explanation/README.md) | "How does X work" / "Why is ..." | Design rationale, architecture, and conceptual models. Covers what Omega is, its philosophy and intellectual lineage, the OQL language model (terms, unification, maps), the execution model (solution sets, joins, clause boundaries, performance), the page tree and block types, the component/protocol/implementation architecture, and the plugin lifecycle with actor-model event flow. |
| [gotchas/](gotchas/README.md) | "Why did X fail" | Failure modes and debugging patterns for the public OQL language surface. Covers lexical scope rules (push-time vs runtime, symbol renaming, datastore survival), and boolean/return-value/index-matching conventions that cause silent failures. |

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
