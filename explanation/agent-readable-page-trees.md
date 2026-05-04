[< Home](../README.md) | [Explanation](README.md)

# Agent-Readable Page Trees

## The Problem

`qo doc push` creates **bare folder pages** — folder nodes with no content, just a name and children. An agent running `qo ls "Docs/MAB"` sees child names:

```
reference
how-to
explanation
```

But it has no signal about what is in each folder without descending blind. The agent cannot tell from the listing alone whether `reference` contains protocols, data models, command signatures, or something else entirely. To answer a question, it must either guess and descend, or exhaustively list every subtree — which is slow, brittle, and produces hallucination pressure when listings return unfamiliar names.

## The Fix: README Pages at Every Folder Level

The solution is a `README.html` file at every directory level in the `pages/<project>/` source tree. When pushed to OmegaAI, each `README.html` becomes a doc page named `README`, a child of its parent folder — not the folder itself.

```
pages/mab/
  README.html          → Docs/MAB/README
  reference/
    README.html        → Docs/MAB/reference/README
    protocols.html     → Docs/MAB/reference/protocols
  how-to/
    README.html        → Docs/MAB/how-to/README
    configure-max-active-subjects.html → Docs/MAB/how-to/configure-max-active-subjects
```

An agent navigating this tree reads `Docs/MAB/README` first. The README describes every child — subfolder names, what each contains, what questions each answers. The agent can route to the right subfolder after reading a single page instead of descending through every branch.

## The Convention

**Each README describes the current level** — what is here and what question each child answers.

The root README lists every direct child with a 1–2 sentence summary:

```html
<h1>MAB Documentation</h1>
<p>Documentation for the MAB (Multi-Armed Bandit) agent system and allocation pipeline.</p>
<ul>
  <li><a href="reference/README">reference/</a> — Exact specifications: protocols, data models, component layout, connector status</li>
  <li><a href="how-to/README">how-to/</a> — Step-by-step task recipes: configuring steps, adding arms, debugging drains</li>
  <li><a href="explanation/README">explanation/</a> — Design rationale: why the pipeline works the way it does</li>
</ul>
```

Each subfolder's README describes its immediate children:

```html
<h1>MAB — Reference</h1>
<ul>
  <li><a href="protocols">protocols</a> — Protocol definitions for all agent step interfaces</li>
  <li><a href="allocation-workflow-data-model">allocation-workflow-data-model</a> — The full data model for the allocation pipeline</li>
  <li><a href="component-layout">component-layout</a> — Page layout for all MAB components in the workspace</li>
</ul>
```

**The 2-README rule:** An agent should be able to route to the right doc after reading at most 2 READMEs — the root README to identify the right subtree, then the subtree's README to identify the right leaf doc.

## Why This Is the OmegaAI Equivalent of the KB Rule

The Omega knowledge base has a hard navigation rule: "Every README must route correctly." The same principle applies here, adapted for page trees navigated via `qo ls` + `qo read` rather than the filesystem.

| Context | Navigation surface | README rule |
|---|---|---|
| `omega-knowledge-base/` | Filesystem + navigator skill | Every README must route — descend from root |
| OmegaAI page tree | `qo ls` + `qo read` | Every folder level has a README that routes |

In the knowledge base, an agent descends from `README.md` at the root and reads each README to navigate. In an OmegaAI page tree, the same agent reads `Docs/MAB/README` first, then descends once to the subfolder's README. The two-hop limit is the same as the knowledge base's "route correctly in at most two steps" principle.

## What Happens Without READMEs

Without README pages, an agent asking "what protocols does the MAB system use?" does this:

1. `qo ls "Docs/MAB"` → sees `reference`, `how-to`, `explanation`
2. Must guess: does `reference` have protocols? `how-to`?
3. Tries `qo ls "Docs/MAB/reference"` → sees `protocols`, `component-layout`, `allocation-workflow-data-model`, ...
4. Reads `protocols` — correct, but the routing was a guess

With README pages:

1. `qo read "Docs/MAB/README"` → sees that `reference/` contains "Protocol definitions for all agent step interfaces"
2. Reads `Docs/MAB/reference/protocols` — zero guessing

## v1 Convention — Future Enforcement

This is explicitly a v1 convention. The pattern is established but not yet enforced by tooling. A future `qo doc check` command could verify README presence at every folder level before or after a push, surfacing missing READMEs as warnings.

Until enforcement lands, README presence is a manual authoring discipline — the same way it is in the knowledge base today.

## Related

- [publish-docs-for-agent-access.md](../../query-omega-cli/docs/how-to/publish-docs-for-agent-access.md) — step-by-step how-to for converting markdown and pushing to OmegaAI, including the README.html template (Step 3).
- [docs-folder-convention.md](../reference/docs-folder-convention.md) — where in OmegaAI synced doc trees live (`Docs/` at workspace root) and subfolder naming conventions.
- [Pages](pages.md) — the page model; page tree structure, `qo ls`, and page path conventions.
