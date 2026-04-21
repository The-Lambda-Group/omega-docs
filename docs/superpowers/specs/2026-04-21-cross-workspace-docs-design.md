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

A recursive `set-perm -R` cascades the policy to every descendant as a batch write at permission-set time, not at read time as a walk. After the write, a read is a single record lookup.

This is the inverse of Azure RBAC's usual cascade-at-read pattern — appropriate for our read-heavy / permission-rare workload on CouchDB.

### Invisibility over forbidden

Cross-workspace `qo ls` / `qo search` / `qo read` return only what the caller can see. Pages without `other +r` are absent from results, not present-but-403. This is the Plan 9 principle: your view of the tree is what you're entitled to, period.

## Data model

### `Qo.Data.Page.Policy` (new datastore)

Records are **exceptions to the default**. A page with no record uses the default policy. Matches the existing OmegaDB pattern where `folder-pkey` defaults to `[]` and page component defaults to `"Page"` when no record is written.

One record per non-default page:

```
{
  "app-id":   <owning-workspace-id>,
  "page-id":  <page-id>,
  "mode":     "rwx------"          ; 9-char unix-literal string
                                   ; positions: owner(3) group(3) other(3)
}
```

**Default when no record exists:** `"rwx------"` — workspace-private. Applied in the `Qo.Page/effective-policy!` resolver:

```
(:- (Qo.Page/effective-policy! AppId PageId Policy)
    (if (Qo.Data.Page.Policy/policy AppId PageId Policy)
      (= 1 1)
      (= Policy "rwx------")))
```

**No migration.** Existing pages stay untouched; their effective policy is the default. Only pages that are explicitly changed (published, restricted) gain a record. Scales cleanly to millions of pages.

**Back-compat with `published`:** the existing `Qo.Public.Api.Page/set-published!` wrapper maps `"true"` → writes mode `"rwx---r--"`, `"false"` → deletes the policy record (returns to default). The existing `published!` endpoint reads the effective policy's `other` bit and reports `"true"`/`"false"`. Nothing calling the old API breaks; it's just a thin alias.

**Group slot** in the mode string (positions 4-6) is always `"---"` in v1. Reserved for future group grants.

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

Two new clauses; the existing `set-published` / `published` wrappers remain:

- `perm!` — read a page's effective policy (from record or default).
- `set-perm!` — write a policy record. Overwrites any prior record. Deleting the record (equivalent to restoring default) is either an explicit `clear-perm!` or passing the default mode; v1 just overwrites.
- `set-perm-recursive!` — write the same policy to the target page and every descendant in one transaction. Flat write — no partial/additive application. Also updates `Qo.Data.Page.WorldIndex` for each affected page.
- `search!` — query the world index by name + text, returns matching `{page-id, app-id, name}` tuples.

**Cross-workspace read is not a new clause** — the existing `page-by-id!` and `page-blocks!` gain an inline permission check: if the caller's `AppId` is not the owning workspace, require `effective-policy[other] includes "r"`. If the page isn't readable, behave as if the page doesn't exist (Plan 9 invisibility, not 403).

### CLI surface (`qo`)

Unix-literal:

| command | behavior |
|---|---|
| `qo perm <page>` | print the page's effective policy (e.g. `rwx---r--`) |
| `qo set-perm <page> <mode>` | write a policy record for the page. `<mode>` is the 9-char string (e.g. `rwx---r--`) or the subset of symbolic ops we choose to support (`o+r`, `o-r`) |
| `qo set-perm -R <page> <mode>` | write the same policy to target + every descendant in one transaction. Flat write, no partial application |
| `qo search <query>` | search across all world-visible pages in OmegaAI |
| `qo read <page-id>` | read page blocks by direct page-id. Works across workspaces for pages with `other +r`; falls back to today's in-workspace path resolution when given a path |

v1 keeps cross-workspace addressing to page-ids only. `qo search` returns `page-id` fields, so the natural flow is search → read. A future `workspace-slug/path` addressing form can land once workspaces have a stable slug field; deferred.

**Publish as convention, not a verb.** "Publishing" a page in v1 is just `qo set-perm <page> rwx---r--` (or whatever symbolic form we support). No dedicated `qo publish` verb. The old `Qo.Public.Api.Page/set-published` wrapper stays for frontend back-compat; the CLI goes pure unix.

Open choice on `<mode>` format: pick 9-char absolute only (simpler to parse, less ambiguous) vs. chmod-style (`o+r`, `u=rwx`, `755`). My lean for v1: absolute only. Symbolic ops can come later without breaking consumers.

## Agent rendering polish

Two chat-side changes, both small. Knock them out first since they're trivial relative to the permission work:

1. **Markdown rendering in agent text output.** Today agent responses render as plain text. Switch to a markdown renderer so headings, lists, code, and links format naturally. Claude already emits markdown by default; we just need the client to render it.

2. **Page context in the session system prompt.** The agent currently only knows the current `pageId` UUID. Pass at session start (and on page change): page name, path breadcrumbs, immediate siblings. Lets the agent answer "what page am I on?" without tool calls.

No HTML preprocessing needed — Claude reads HTML fine and restates in markdown naturally.

These changes are in the agent service (`server.ts`) and the AI panel (`events/TextBlock.tsx`). No new OQL or datastore changes.

## End-to-end flows

### Publishing the KB

1. Ryan migrates `omega-knowledge-base` into an OmegaAI workspace (separate content-migration script; not in this spec).
2. From that workspace: `qo set-perm -R / rwx---r--` — writes the policy on every page and updates world-index entries. Pages that had no prior policy record now get one.
3. Any other workspace's agent can now `qo search "descend from root"` and get back a result pointing at the KB. It can `qo read <page-id>` to fetch.

### Sharing a single customer doc

1. Redefine workspace writes `Data/Copy Strategy` page.
2. `qo set-perm "Data/Copy Strategy" rwx---r--` — one page becomes world-readable.
3. Marc's workspace's agent, or the public web, can read that page without seeing the `Data` folder's other contents.

### Agent answering "what is MAB?"

1. User asks in the AI panel.
2. Agent sends `qo search mab` — hits the world index.
3. Results include `omega-knowledge-base/projects/mab/README` (published) and possibly `wttc/Component Installs/MAB Agent System/Brief` (if published).
4. Agent `qo read`s the top result, composes an answer.
5. Markdown response renders cleanly in the panel.

## Tradeoffs and open questions

- **Publishing a leaf in an unpublished folder.** By design: allowed. The leaf's name shows up in search; its path does not leak. A consumer opening the leaf sees the leaf's own breadcrumbs only as far as the published chain extends. We may want a UX warning on the first such publish ("This page will be world-visible but its parents are private. Breadcrumbs will be truncated.").

- **Write cost at `qo set-perm -R` scale.** A KB with ~500 pages costs ~500 + 500 write ops (policy + index). CouchDB handles batched writes, but this will be slower than a single-record flip. Acceptable for infrequent actions; worth measuring once.

- **`qo ls` cross-workspace.** Out of scope for v1 — you reach other workspaces via `qo search` or direct `qo read`. A cross-workspace `ls` would need a cross-workspace path-walker; deferred.

- **Namespace semantics for `qo ls` within a workspace.** Today `ls` shows all children. With this model it should start filtering by visibility-for-caller too — but since owner can always see own pages, this only matters when cross-workspace `ls` arrives. Defer with the `ls` change.

- **Group subject.** Stored but unused in v1. Open: what does a "group" resolve to? Candidates: a named set of workspaces, a set of user identities, a role. Revisit when the use case drives it; don't pre-design.

- **`x` enforcement.** Stored, unenforced. When we add "run implementation" permissions, `x` on the page gates it. Defer.

- **Deep-link back to workspace page from agent response.** Not in this spec; orthogonal UX improvement.

## Phasing

Phase 1 (this spec's implementation plan):

- `Qo.Data.Page.Policy` + `Qo.Data.Page.WorldIndex` datastores
- `Qo.Page/effective-policy!` resolver with default fallback
- `set-perm!` / `set-perm-recursive!` / `perm!` / `search!` OQL clauses
- Inline permission check in existing `page-by-id!` / `page-blocks!`
- Back-compat: old `set-published!` / `published!` map to the new model
- `qo perm` / `qo set-perm` / `qo set-perm -R` / `qo search` / extended `qo read`
- Agent rendering polish: client-side markdown, page context in system prompt

Phase 2 (follow-on):

- Cross-workspace `qo ls`
- UI surfaces in query-omega: permission controls on pages, cross-workspace search in the AI panel
- Symbolic mode ops (`o+r`, `u=rwx`, `755`) in `qo set-perm` if the absolute-only form proves awkward

Phase 3+:

- Group subject with a concrete semantic
- `x` enforcement when implementation-run permissions land
- OpenBSD-style per-session agent pledge
- Full marketplace (protocols + components publishing scope) — separate spec

## Testing

- `Qo.Page/effective-policy!` returns the default `"rwx------"` for pages with no record.
- `set-perm!` writes a record; `effective-policy!` reflects it on next read.
- `set-perm-recursive!` on a 3-level folder writes policy + world-index records for all affected pages.
- Removing `other +r` by overwriting with a restricted policy clears the page from the world index.
- Back-compat: `set-published! {published: "true"}` results in effective policy `"rwx---r--"`; `{published: "false"}` restores default.
- Cross-workspace read: a caller in workspace A gets page-data for a published page in workspace B; gets page-not-found for an unpublished one (not a 403).
- Agent end-to-end: from a non-owner workspace, `qo search` + `qo read <page-id>` resolves to real content across workspaces.

## Notes on what this is NOT

This is not a marketplace spec. A marketplace (protocols and components with scoped visibility, install flows, versioning) is a sibling project that will benefit from the primitives this spec establishes. Tonight's spec just makes documents first-class cross-workspace content.
