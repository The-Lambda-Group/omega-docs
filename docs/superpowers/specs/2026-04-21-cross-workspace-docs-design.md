# Cross-Workspace Documentation — Design

> Draft. 2026-04-21.
> Scope: one spec. Implementation plan follows in a separate doc once this is approved.

## Motivation

The in-app agent can read a workspace's pages (via `qo ls` / `qo read`), but can't reach the `omega-knowledge-base` — it lives on Ryan's filesystem and agent containers can't see it. For the agent to feel like a real documentation companion, workspaces need to be the canonical home for docs, and workspaces need to be able to read each other's published docs.

The existing `published` flag on pages (a per-page boolean stored in `Qo.Data.Page.Published`) is the right primitive to build on. It's currently used for public-URL viewing with no cascade and no cross-workspace discovery. That's the half-baked bone the full design extends.

## Scope

**In scope:**

- A small but extensible permission model on pages
- `qo publish` / `qo unpublish` with a recursive variant
- Cross-workspace read of published pages by page-id (and by `workspace/path` form)
- Cross-workspace search across published pages (new `qo search`)
- Agent-side rendering polish (markdown output, HTML-block rendering, page-context in system prompt)

**Out of scope (deferred to sibling specs):**

- Group-based sharing beyond a placeholder slot
- Protocol / component publishing scope (separate marketplace spec)
- Plan 9–style pledge/unveil for agent sessions (future hardening)
- Third-party identity / auth (uses the existing auth model)

## Conceptual model

The right mental model isn't strict unix, AWS IAM, or Azure RBAC. It borrows from each:

- **Unix** gives us the user-facing API shape (`chmod`-like verbs, nine-bit mental model).
- **Azure RBAC** gives us the right concept for hierarchical resources: scoped grants that cascade along a folder tree.
- **Plan 9** gives us per-principal namespaces: if you lack `r` on a page it is absent from your view, not returned with a 403. Less leaky, less discovery surface.
- **OpenBSD** gives us a future direction: agent sessions can eventually pre-commit a permission envelope (`pledge`/`unveil` analog) so a prompt-injected agent can't escalate. Not in v1, but the model should leave room.

The key performance constraint: **OmegaDB sits on CouchDB. Reads must be cheap. Writes can be expensive.** That means grants materialize at publish time, not cascade at read time.

### Permission tuple

Every page carries a tuple of the form `(owner, group, other) × (r, w, x)`:

| slot | meaning |
|---|---|
| owner | the workspace (`AppId`) that created the page |
| group | reserved slot — stored but unused in v1 |
| other | world — callers outside the owner workspace |
| `r` | read page metadata, blocks, row data |
| `w` | modify blocks, schema, create sub-pages |
| `x` | traverse folder / run implementation — stored but unenforced in v1 |

**v1 enforcement:** only `other +r` is enforced for cross-workspace reads. Owner always has full access to its own pages (unchanged from today). Other modes are stored so they can be flipped on later without a data migration.

**Defaults:** new page gets `rwx------` — workspace-private by default. Publishing is always an explicit action.

### Cascading grants, materialized at write time

Publishing a folder cascades `other +r` to every descendant, but the cascade happens at **publish time** as a batch write, not at **read time** as a walk. After publishing, a read is a single record lookup.

This is the inverse of Azure RBAC's usual cascade-at-read pattern — appropriate for our read-heavy / publish-rare workload on CouchDB.

### Invisibility over forbidden

Cross-workspace `qo ls` / `qo search` / `qo read` return only what the caller can see. Pages without `other +r` are absent from results, not present-but-403. This is the Plan 9 principle: your view of the tree is what you're entitled to, period.

## Data model

### `Qo.Data.Page.Policy` (new datastore)

Replaces the current `Qo.Data.Page.Published` boolean. One record per `(page-id, subject)`:

```
{
  "app-id":    <owning-workspace-id>,
  "page-id":   <page-id>,
  "subject":   "owner" | "group" | "other",
  "modes":     "rwx" | "r-x" | "r--" | ...,
  "scope":     "self"              ; v1 only writes self-scoped records.
                                   ; a future "subtree" value could lazy-cascade.
}
```

v1 writes two records per page:

- `{subject: "owner", modes: "rwx"}` — implicit today; make it explicit in the data model.
- `{subject: "other", modes: "---"}` by default; flipped to `"r--"` on publish.

The owner record is always present. The other record flips between `"---"` and `"r--"`. Group records are never written in v1 but the datastore shape accepts them.

**Migration:** existing `Qo.Data.Page.Published` rows map to a `{subject: "other", modes: "r--"}` policy. Implemented as an idempotent migration script so `published` can be deprecated.

### Cross-workspace search index

A new datastore `Qo.Data.Page.WorldIndex` tracks the subset of pages that currently have `other +r`. Written at publish time; deleted at unpublish time. Fields:

```
{
  "page-id":      <page-id>,
  "app-id":       <owning-workspace-id>,
  "name":         <page name>,
  "text-excerpt": <first ~500 chars of concatenated block text, for search ranking>,
  "indexed-at":   <timestamp>
}
```

The index is the whole cross-workspace visibility surface. If a page isn't in it, no one outside the owning workspace can see it.

Keeping the index as a separate datastore (rather than querying `Qo.Data.Page.Policy` at search time) matches the "cheap reads" constraint — searches scan a narrow index, not the full policy table.

## Public API (`Qo.Public.Api.Page`)

The existing `set-published` / `published` endpoints are preserved as thin wrappers for back-compat (they map to `set-policy!` with `subject: "other"`, `modes: "r--"` or `"---"`). Nothing calling the old API breaks. The real surface becomes:

- `set-policy!` — set modes for a `(page-id, subject)` pair.
- `set-policy-recursive!` — same, but applied to page + all descendants in one transaction. Also writes to `Qo.Data.Page.WorldIndex` where modes include `other +r`.
- `policy!` — read policies on a page. Returns empty / owner-only if caller can't see the page.
- `search!` — query the world index by name + text, returns matching `{page-id, app-id, name}` tuples.
- `read-world!` — read a page's blocks by `page-id` if the caller has access. Scoped to pages in the world index; doesn't need the caller to be in the owning workspace.

### CLI surface (`qo`)

Minimal, Unix-flavored:

| command | behavior |
|---|---|
| `qo publish <page>` | set `other +r` on the one page |
| `qo publish -R <page>` | set `other +r` on the page + all descendants |
| `qo unpublish <page>` | clear `other +r` on the one page |
| `qo unpublish -R <page>` | clear `other +r` on page + all descendants |
| `qo search <query>` | search across all world-visible pages in OmegaAI |
| `qo read <page-id>` | read page blocks by direct page-id. Works across workspaces for pages with `other +r`; falls back to today's in-workspace path resolution when given a path. |

v1 keeps cross-workspace addressing to page-ids only. `qo search` returns `page-id` fields, so the natural flow is search → read. A future `workspace-slug/path` addressing form can land once workspaces have a stable slug field; deferred.

A future `qo chmod` can land when modes beyond `other +r` become enforceable.

## Agent rendering polish

Independent of the permission model, three chat-side changes to make documentation navigation feel good:

1. **Markdown rendering in agent text output.** Today agent responses render as plain text. Switch to a markdown renderer so headings, lists, code, and links format naturally.

2. **HTML-block rendering.** When `qo read` returns a block of type `HtmlPageBlock`, the chat currently shows raw HTML tags in the JSON. Two changes:
   - Agent-side: convert HTML to markdown before passing to Claude (so the agent sees structured content, not tag soup).
   - Client-side: if the agent emits HTML/markdown in a response, render it rather than showing raw.

3. **Page context in the session system prompt.** Today the agent only knows the current `pageId` UUID. Pass at session start (and on any page change): page name, path breadcrumbs, immediate siblings. The agent can answer "what page am I on?" without tool calls.

These changes are all in the agent service (`server.ts`) and the AI panel (`AiPanel*`, `events/TextBlock.tsx`). No new OQL or datastore changes.

## End-to-end flows

### Publishing the KB

1. Ryan migrates `omega-knowledge-base` into an OmegaAI workspace (separate content-migration script; not in this spec).
2. From that workspace: `qo publish -R /` — sets `other +r` on every page and writes world-index entries.
3. Any other workspace's agent can now `qo search "descend from root"` and get back a result pointing at the KB. It can `qo read` the result.

### Sharing a single customer doc

1. Redefine workspace writes `Data/Copy Strategy` page.
2. `qo publish Data/Copy Strategy` — one page becomes world-readable.
3. Marc's workspace's agent, or the public web, can read that page without seeing the `Data` folder's other contents.

### Agent answering "what is MAB?"

1. User asks in the AI panel.
2. Agent sends `qo search mab` — hits the world index.
3. Results include `omega-knowledge-base/projects/mab/README` (published) and possibly `wttc/Component Installs/MAB Agent System/Brief` (if published).
4. Agent `qo read`s the top result, composes an answer.
5. Markdown response renders cleanly in the panel.

## Tradeoffs and open questions

- **Publishing a leaf in an unpublished folder.** By design: allowed. The leaf's name shows up in search; its path does not leak. A consumer opening the leaf sees the leaf's own breadcrumbs only as far as the published chain extends. We may want a UX warning on the first such publish ("This page will be world-visible but its parents are private. Breadcrumbs will be truncated.").

- **Write cost at `qo publish -R` scale.** A KB with ~500 pages costs 500 + 500 write ops (policy + index). CouchDB handles batched writes, but this will be slower than a single-record flip. Acceptable for infrequent publish actions; worth measuring.

- **`qo ls` cross-workspace.** Out of scope for v1 — you reach other workspaces via `qo search` or direct `qo read`. A cross-workspace `ls` would need a cross-workspace path-walker; deferred.

- **Namespace semantics for `qo ls` within a workspace.** Today `ls` shows all children. With this model it should start filtering by visibility-for-caller too — but since owner can always see own pages, this only matters when cross-workspace `ls` arrives. Defer with the `ls` change.

- **Group subject.** Stored but unused in v1. Open: what does a "group" resolve to? Candidates: a named set of workspaces, a set of user identities, a role. Revisit when the use case drives it; don't pre-design.

- **`x` enforcement.** Stored, unenforced. When we add "run implementation" permissions, `x` on the page gates it. Defer.

- **Deep-link back to workspace page from agent response.** Not in this spec; orthogonal UX improvement.

## Phasing

Phase 1 (this spec's implementation plan):

- `Qo.Data.Page.Policy` + `Qo.Data.Page.WorldIndex` datastores
- Migration from `Qo.Data.Page.Published`
- `set-policy` / `set-policy-recursive` / `policy` / `search` / `read-world` OQL clauses
- `qo publish` / `qo publish -R` / `qo unpublish` / `qo unpublish -R` / `qo search` / extended `qo read`
- Agent rendering polish (markdown out, HTML-block conversion, page context in system prompt)

Phase 2 (follow-on):

- Cross-workspace `qo ls`
- UI surfaces in query-omega: publish/unpublish button on pages, cross-workspace search in the AI panel
- Batch UX (publish warnings, publish-with-children confirmation)

Phase 3+:

- Group subject with a concrete semantic
- `x` enforcement when implementation-run permissions land
- OpenBSD-style per-session agent pledge
- Full marketplace (protocols + components publishing scope) — separate spec

## Testing

- Policy migration round-trips the existing `Qo.Data.Page.Published` state.
- `publish -R` on a 3-level folder writes `N` policy records and `N` world-index records, reads back correctly.
- `unpublish -R` removes both.
- `search` returns results only for currently-published pages; unpublished pages vanish from index immediately.
- `read-world` from workspace A reads a published page in workspace B; fails cleanly for an unpublished one.
- Agent end-to-end: from a non-owner workspace, `qo search` + `qo read` resolves to real content across workspaces.

## Notes on what this is NOT

This is not a marketplace spec. A marketplace (protocols and components with scoped visibility, install flows, versioning) is a sibling project that will benefit from the primitives this spec establishes. Tonight's spec just makes documents first-class cross-workspace content.
