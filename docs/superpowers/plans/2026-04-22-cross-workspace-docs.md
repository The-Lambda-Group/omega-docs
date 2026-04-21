# Cross-Workspace Documentation Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Ship the cross-workspace documentation system: per-page permission model with unix-shaped API, cross-workspace search and read for published pages, and agent-side rendering polish.

**Architecture:** New `Qo.Data.Page.Policy` datastore stores non-default per-page policies (owner/group/other × rwx as a 9-char mode string). Default policy `rwx------` applied when no record exists — matches the existing folder-pkey fallback pattern. Separate `Qo.Data.Page.WorldIndex` datastore maintained at permission-write time for cheap cross-workspace search. Public API gains `perm!` / `set-perm!` / `set-perm-recursive!` / `search!`; existing `page-by-id!` and `page-blocks!` get inline permission checks. Existing `set-published!` / `published!` stay as back-compat wrappers. CLI gains `qo perm` / `qo set-perm` / `qo search` and extends `qo read` to accept a page-id. Agent service injects page context (name + siblings) into the system prompt; AI panel renders agent text output as markdown.

**Tech Stack:** OQL / OmegaDB / CouchDB (permission + index datastores, clauses), Node.js (query-omega-cli, omega-ai-agent-service), React / MUI (query-omega AI panel).

**Source spec:** [2026-04-21-cross-workspace-docs-design.md](../specs/2026-04-21-cross-workspace-docs-design.md)

**Related:** [2026-04-22-namespace-rbac-design.md](../specs/2026-04-22-namespace-rbac-design.md) (sibling; not implemented by this plan).

---

## File structure

**query-omega-oql — data + internal clauses:**
- Create: `oql/app/page/policy.oql` — Policy datastore definition + `Qo.Page/effective-policy!`, `Qo.Page/set-policy!`, `Qo.Page/set-policy-recursive!`
- Create: `oql/app/page/world-index.oql` — WorldIndex datastore + add/remove helpers
- Modify: `oql/app/page.oql` — re-export / wire new policy clauses if needed

**query-omega-api — public surface:**
- Modify: `oql/app/public/api/page.oql` — add `perm!`, `set-perm!`, `set-perm-recursive!`, `search!`; permission check in existing `page-by-id!` + `page-blocks!`; rewrite `set-published!` / `published!` as thin wrappers over the new model
- Modify: `oql/app/public/oql-api/page.oql` — mirror clauses where the oql-api variant exists

**query-omega-cli — user-facing CLI:**
- Create: `app/cmd/perm.js` — `qo perm <page>`
- Create: `app/cmd/set-perm.js` — `qo set-perm <page> <mode>` (with `-R` flag)
- Create: `app/cmd/search.js` — `qo search <query>`
- Modify: `app/cmd/read.js` — accept a page-id as an alternative to a path
- Modify: `index.js` — register new commands

**omega-ai-agent-service — page context:**
- Modify: `src/server.ts` — fetch page name + siblings for the current pageId and include in the system-prompt context

**query-omega — client rendering:**
- Modify: `src/components/ai-panel/events/TextBlock.tsx` — render agent text output as markdown
- Modify: `package.json` — add markdown renderer dependency (`react-markdown`)

---

## Phase 1 — Rendering polish (easy wins, ship first)

### Task 1: Client-side markdown rendering in the AI panel

**Files:**
- Modify: `/home/ryan/Development/omega/query-omega/package.json`
- Modify: `/home/ryan/Development/omega/query-omega/src/components/ai-panel/events/TextBlock.tsx`

- [ ] **Step 1: Add the renderer dependency**

```bash
cd /home/ryan/Development/omega/query-omega
npm install react-markdown
```

Verify `react-markdown` appears under `dependencies` in `package.json`.

- [ ] **Step 2: Replace plain-text rendering with markdown for agent messages**

Read the current [TextBlock.tsx](../../../../query-omega/src/components/ai-panel/events/TextBlock.tsx). It renders `event.content` as plain `<Typography>`. Change it so that when `event.type === AiEventType.AGENT_MESSAGE` the content is passed to `<ReactMarkdown>`. User messages (`USER_MESSAGE`) keep the current plain rendering — no need to parse user input.

Imports to add:

```ts
import ReactMarkdown from "react-markdown";
```

Conditionally render based on event type. Leave existing styles intact; ReactMarkdown emits normal HTML that the existing theme will style.

- [ ] **Step 3: Verify in the browser**

Start the dev frontend against the local agent:

```bash
cd /home/ryan/Development/omega/query-omega
./bin/dev-with-local-agent.sh
```

Open the AI panel, ask the agent any question that will produce a response with headings or a list (e.g. "summarize the MAB pipeline"). Confirm the response renders with formatted headings / lists / code, not raw `#` / `*` characters.

- [ ] **Step 4: Commit**

```bash
cd /home/ryan/Development/omega/query-omega
git add package.json package-lock.json src/components/ai-panel/events/TextBlock.tsx
git commit -m "feat(ai-panel): render agent text output as markdown"
```

---

### Task 2: Page context in the agent system prompt

**Files:**
- Modify: `/home/ryan/Development/omega/omega-ai-agent-service/src/server.ts`

Currently the agent receives only `pageId` (UUID) in the per-message context. Add the page's name and immediate sibling names so the agent can answer "what page am I on?" and "what's nearby?" without a tool call.

- [ ] **Step 1: Add a helper that fetches page name + siblings via the public API**

In `src/server.ts`, add a function that calls `Qo.Public.OqlApi.Page/page-by-id!` (or the equivalent HTTP endpoint) to get the page's name and its parent's children. Since this runs per user.message, keep it fast — one query that returns both, fail gracefully to an empty context on error.

```ts
async function fetchPageContext(
  omegaUrl: string,
  appId: string,
  pageId: string,
): Promise<{ name: string; siblings: string[] } | null> {
  try {
    const query = `
(datastore Qo.Public.OqlApi.Page "omega/query-omega/public/oql-api/page")
(Qo.Public.OqlApi.Page/page-by-id "${pageId}" Page)
(get-in Page ["page-data" "name"] PageName)
(get Page "folder-id" ParentId)
(Qo.Public.OqlApi.Page/pages-by-folder ParentId Sibling)
(get-in Sibling ["page-data" "name"] SiblingName)
(fold [] SiblingName append Siblings)
(= Result {"name" PageName "siblings" Siblings})
(return [Result])
    `;
    const res = await fetch(`${omegaUrl}/query_clj`, {
      method: "POST",
      headers: { "Content-Type": "text/plain" },
      body: query,
    });
    const json = await res.json();
    const row = json?.result?.data?.[0]?.[0];
    if (!row) return null;
    return { name: row.name, siblings: row.siblings || [] };
  } catch {
    return null;
  }
}
```

`omegaUrl` needs to come from config/env. Pick it up from `OMEGA_URL` with a sensible default of `https://db.queryomega.com`.

- [ ] **Step 2: Integrate the context into the per-message prefix**

In `server.ts`, where the code currently builds `pagePrefix`, extend it with the page name and siblings when available:

```ts
const pageContext = pageId ? await fetchPageContext(OMEGA_URL, workspaceId, pageId) : null;
const pagePrefix = pageId
  ? `[Current page: ${pageContext?.name ?? pageId} (id: ${pageId})${
      pageContext?.siblings?.length
        ? `\nSiblings of current page: ${pageContext.siblings.join(", ")}`
        : ""
    }]\n\n`
  : "";
```

- [ ] **Step 3: Build and verify locally**

```bash
cd /home/ryan/Development/omega/omega-ai-agent-service
npm run build
source scripts/_env.sh
docker build --build-arg NPM_REGISTRY="${NPM_REGISTRY}" --build-arg NPM_AUTH_TOKEN="${NPM_AUTH_TOKEN}" -t omega-ai-agent-service:latest .
docker stop agent && docker rm agent
docker run -d --name agent --network agent-net -p 8080:8080 \
  -e AGENT_AUTH_TOKEN=local-test-token \
  -e ANTHROPIC_BASE_URL=http://proxy:9090 \
  -e ANTHROPIC_API_KEY=sk-ant-placeholder \
  -e OMEGA_URL=https://db.queryomega.com \
  omega-ai-agent-service:latest
```

Open the AI panel on a known page. Ask "what page am I on?" — expect a confident answer using the page's name, no tool call required.

- [ ] **Step 4: Commit**

```bash
cd /home/ryan/Development/omega/omega-ai-agent-service
git add src/server.ts
git commit -m "feat(agent): inject current page name + siblings into system context"
```

---

## Phase 2 — Permission data layer (OQL)

### Task 3: Create the Policy datastore

**Files:**
- Create: `/home/ryan/Development/omega/query-omega-oql/oql/app/page/policy.oql`
- Modify: `/home/ryan/Development/omega/query-omega-oql/oql/app/page.oql`

Model records in `Qo.Data.Page.Policy` as `{app-id, page-id, mode}`. Default when no record exists is `"rwx------"`.

- [ ] **Step 1: Write the datastore file with the raw datastore declaration and accessor clauses**

Create `oql/app/page/policy.oql` with this content, following the shape of other datastore files in this repo (see `oql/app/page.oql` for reference — same imports pattern):

```
(datastore Qo.Data.Page.Policy "omega/query-omega/data/page/policy")
(datastore Qo.Data.Page.Data   "omega/query-omega/data/page")

;; Read a page's explicit policy record. Fails if none exists.
(:- (Qo.Data.Page.Policy/policy! AppId PageId Mode)
    (Qo.Data.Page.Policy/by-app-and-page AppId PageId Record)
    (get Record "mode" Mode))

;; Write or overwrite a policy record.
(:- (Qo.Data.Page.Policy/set! AppId PageId Mode)
    (Qo.Wr/write-data {"app-id" AppId "page-id" PageId "mode" Mode} _))

;; Delete a policy record (returns to default).
(:- (Qo.Data.Page.Policy/clear! AppId PageId)
    (Qo.Data.Page.Policy/delete! (by-app-and-page AppId PageId)))

(return ["omega.query-omega.data.page.policy"])
```

Note: the exact datastore syntax and `Qo.Data.Page.Policy/by-app-and-page` accessor shape follow this repo's conventions — look at `oql/app/page.oql` lines around the `Qo.Data.Page.Published` datastore (our model replaces this one) for how an analogous datastore is declared. Mirror that shape.

- [ ] **Step 2: Add the effective-policy resolver with default fallback**

Append to the same file:

```
(:- (Qo.Page/effective-policy! AppId PageId Mode)
    (if (Qo.Data.Page.Policy/policy AppId PageId Mode)
      (= 1 1)
      (= Mode "rwx------")))
```

This is the load-bearing clause. Everywhere else calls this to read a page's effective policy; the fallback means pages without records just work.

- [ ] **Step 3: Verify by running a scratch query against a dev workspace**

Against a dev workspace (any page you own), write a scratch query:

```
(datastore Qo.Page "omega/query-omega/page")
(Qo.Page/effective-policy "<workspace-app-id>" "<some-page-id>" Mode)
(return [Mode])
```

Expected result: `["rwx------"]` (the default, because no record has been written yet).

- [ ] **Step 4: Commit**

```bash
cd /home/ryan/Development/omega/query-omega-oql
git add oql/app/page/policy.oql
git commit -m "feat(oql): Qo.Data.Page.Policy datastore + effective-policy resolver"
```

---

### Task 4: Create the WorldIndex datastore

**Files:**
- Create: `/home/ryan/Development/omega/query-omega-oql/oql/app/page/world-index.oql`

- [ ] **Step 1: Write the datastore file**

Create `oql/app/page/world-index.oql`:

```
(datastore Qo.Data.Page.WorldIndex "omega/query-omega/data/page/world-index")
(datastore Qo.Page.Bl              "omega/query-omega/page/block")

;; Record shape: {app-id, page-id, name, text-excerpt, indexed-at}

(:- (Qo.Data.Page.WorldIndex/entry! AppId PageId Record)
    (Qo.Data.Page.WorldIndex/by-app-and-page AppId PageId Record))

(:- (Qo.Data.Page.WorldIndex/add! AppId PageId Name TextExcerpt Timestamp)
    (Qo.Wr/write-data
      {"app-id" AppId
       "page-id" PageId
       "name" Name
       "text-excerpt" TextExcerpt
       "indexed-at" Timestamp}
      _))

(:- (Qo.Data.Page.WorldIndex/remove! AppId PageId)
    (Qo.Data.Page.WorldIndex/delete! (by-app-and-page AppId PageId)))

(return ["omega.query-omega.data.page.world-index"])
```

Keep the record fields minimal: `name` and `text-excerpt` are what `search!` matches against.

- [ ] **Step 2: Verify the datastore compiles**

Run whatever OQL compile / validation command this repo uses (see `package.json` scripts). Expect no errors.

- [ ] **Step 3: Commit**

```bash
cd /home/ryan/Development/omega/query-omega-oql
git add oql/app/page/world-index.oql
git commit -m "feat(oql): Qo.Data.Page.WorldIndex datastore for cross-workspace search"
```

---

### Task 5: `Qo.Page/set-policy!` — single-page write with world-index maintenance

**Files:**
- Modify: `/home/ryan/Development/omega/query-omega-oql/oql/app/page/policy.oql`

A single-page write that also maintains the WorldIndex: if the new mode includes `other +r`, ensure the page is in the index; if not, ensure it is removed.

- [ ] **Step 1: Add the helper for "does this mode include other +r"**

Append to `policy.oql`:

```
;; Returns true iff the mode string grants 'other +r' (position 6 = 'r').
(:- (Qo.Page/mode-world-readable! Mode Readable)
    (get-char Mode 6 Ch)
    (if (= Ch "r")
      (= Readable true)
      (= Readable false)))
```

If `get-char` isn't available in OQL, use `substring` / equivalent — look at `oql/app/util.oql` for string helpers.

- [ ] **Step 2: Implement `Qo.Page/set-policy!`**

Append:

```
;; Write a policy for one page, and maintain the world index accordingly.
(:- (Qo.Page/set-policy! AppId PageId Mode)
    (Qo.Data.Page.Policy/set AppId PageId Mode)
    (Qo.Page/mode-world-readable Mode Readable)
    (if (= Readable true)
      (Qo.Page/world-index-add AppId PageId)
      (Qo.Data.Page.WorldIndex/remove AppId PageId)))

;; Helper: compute the index entry from the page and write it.
(:- (Qo.Page/world-index-add! AppId PageId)
    (Qo.Public.OqlApi.Page/page-by-id PageId Page)
    (get-in Page ["page-data" "name"] Name)
    (Qo.Page.Bl/page-blocks AppId PageId Blocks)
    (Qo.Page/blocks->text-excerpt Blocks Excerpt)
    (now-ms Timestamp)
    (Qo.Data.Page.WorldIndex/add AppId PageId Name Excerpt Timestamp))
```

`Qo.Page/blocks->text-excerpt` is a helper to flatten text from blocks to ~500 chars. If there's no existing analog, stub it as returning empty string for v1 — we can improve later. Look at `oql/app/page.oql` for existing block-to-text patterns.

- [ ] **Step 3: Verify end-to-end on a test page**

Scratch query — set a policy on a test page you own:

```
(datastore Qo.Page "omega/query-omega/page")
(Qo.Page/set-policy! "<app-id>" "<page-id>" "rwx---r--")
(return [done])
```

Then read it back:

```
(Qo.Page/effective-policy "<app-id>" "<page-id>" Mode)
(return [Mode])
```

Expect `["rwx---r--"]`. And query the world index:

```
(datastore Qo.Data.Page.WorldIndex "...")
(Qo.Data.Page.WorldIndex/by-app-and-page "<app-id>" "<page-id>" Entry)
(return [Entry])
```

Expect an entry with the page's name.

- [ ] **Step 4: Commit**

```bash
cd /home/ryan/Development/omega/query-omega-oql
git add oql/app/page/policy.oql
git commit -m "feat(oql): Qo.Page/set-policy! with world-index maintenance"
```

---

### Task 6: `Qo.Page/set-policy-recursive!` — batch write down a tree

**Files:**
- Modify: `/home/ryan/Development/omega/query-omega-oql/oql/app/page/policy.oql`

Flat policy overwrite applied to target + every descendant.

- [ ] **Step 1: Implement recursive walk + batch write**

Append:

```
;; Walk the tree under RootPageId and apply Mode to every descendant + the root.
(:- (Qo.Page/set-policy-recursive! AppId RootPageId Mode)
    (Qo.Page/set-policy AppId RootPageId Mode)
    (Qo.Page/descendants AppId RootPageId Descendant)
    (get Descendant "page-id" DescendantId)
    (Qo.Page/set-policy AppId DescendantId Mode))

;; Helper: yield every descendant page under RootPageId.
(:- (Qo.Page/descendants! AppId RootPageId Descendant)
    (Qo.Public.OqlApi.Page/pages-by-folder RootPageId Child)
    (or (= Descendant Child)
        (and (get Child "page-id" ChildId)
             (Qo.Page/descendants AppId ChildId Descendant))))
```

The `descendants!` recursion relies on `pages-by-folder` — which we confirmed works at ≥3 levels deep in earlier sessions. If the recursion doesn't terminate cleanly in OQL, use an iterative walk via `fold`.

- [ ] **Step 2: Verify on a small test subtree**

Pick a folder in a dev workspace with 3-5 sub-pages. Run:

```
(Qo.Page/set-policy-recursive "<app-id>" "<folder-page-id>" "rwx---r--")
```

Then verify each descendant has mode `"rwx---r--"` and is present in the world index.

- [ ] **Step 3: Commit**

```bash
cd /home/ryan/Development/omega/query-omega-oql
git add oql/app/page/policy.oql
git commit -m "feat(oql): Qo.Page/set-policy-recursive! — batch write down tree"
```

---

### Task 7: Back-compat — rewrite `set-published!` and `published!` over the new model

**Files:**
- Modify: `/home/ryan/Development/omega/query-omega-api/oql/app/public/api/page.oql`

The existing `Qo.Public.Api.Page/set-published!` and `/published!` clauses become thin wrappers so the frontend keeps working. Old `Qo.Data.Page.Published` datastore is deprecated — remove its usage in these clauses but leave the datastore declaration in place so nothing else breaks if it's referenced elsewhere (follow up cleanup separately).

- [ ] **Step 1: Read the current implementation**

Open `/home/ryan/Development/omega/query-omega-api/oql/app/public/api/page.oql` and locate the existing `Qo.Public.Api.Page/published!` and `/set-published!` clauses (around line 64-78 from earlier exploration).

- [ ] **Step 2: Rewrite the published clauses**

Replace them with:

```
(:- (Qo.Public.Api.Page/published! Input Response)
    (get Input "page-id" PageId)
    (get Input "app-id" AppId)
    (Qo.Page/effective-policy AppId PageId Mode)
    (Qo.Page/mode-world-readable Mode Readable)
    (if (= Readable true)
      (= Response {"published" "true"  "app-id" AppId "page-id" PageId})
      (= Response {"published" "false" "app-id" AppId "page-id" PageId})))

(:- (Qo.Public.Api.Page/set-published! Input Response)
    (get Input "page-id" PageId)
    (get Input "app-id" AppId)
    (get Input "published" Published)
    (when (= Published "true")
      (Qo.Page/set-policy AppId PageId "rwx---r--"))
    (when (= Published "false")
      (Qo.Page/set-policy AppId PageId "rwx------"))
    (Qo.Public.Api.Page/published {"app-id" AppId "page-id" PageId} Response))
```

- [ ] **Step 3: Verify the frontend still works**

From the React app, open a page, click the publish toggle (wherever it exists in the UI) and confirm the page flips state cleanly. No user-visible change from before this task.

- [ ] **Step 4: Commit**

```bash
cd /home/ryan/Development/omega/query-omega-api
git add oql/app/public/api/page.oql
git commit -m "feat(api): rewrite set-published/published over new policy model (back-compat)"
```

---

## Phase 3 — Public API clauses

### Task 8: `Qo.Public.Api.Page/perm!`

**Files:**
- Modify: `/home/ryan/Development/omega/query-omega-api/oql/app/public/api/page.oql`

- [ ] **Step 1: Add the clause**

Append to `page.oql`:

```
(:- (Qo.Public.Api.Page/perm! Input Response)
    (get Input "page-id" PageId)
    (get Input "app-id" AppId)
    (Qo.Page/effective-policy AppId PageId Mode)
    (= Response {"app-id" AppId "page-id" PageId "mode" Mode}))
```

- [ ] **Step 2: Verify via scratch query**

```
(datastore Qo.Public.Api.Page "omega/query-omega/public/api/page")
(Qo.Public.Api.Page/perm {"app-id" "<app-id>" "page-id" "<page-id>"} Response)
(return [Response])
```

Expect the mode string in the response.

- [ ] **Step 3: Commit**

```bash
cd /home/ryan/Development/omega/query-omega-api
git add oql/app/public/api/page.oql
git commit -m "feat(api): Qo.Public.Api.Page/perm! — read effective policy"
```

---

### Task 9: `Qo.Public.Api.Page/set-perm!` and `/set-perm-recursive!`

**Files:**
- Modify: `/home/ryan/Development/omega/query-omega-api/oql/app/public/api/page.oql`

- [ ] **Step 1: Add both clauses**

```
(:- (Qo.Public.Api.Page/set-perm! Input Response)
    (get Input "page-id" PageId)
    (get Input "app-id" AppId)
    (get Input "mode" Mode)
    (Qo.Page/set-policy AppId PageId Mode)
    (Qo.Public.Api.Page/perm {"app-id" AppId "page-id" PageId} Response))

(:- (Qo.Public.Api.Page/set-perm-recursive! Input Response)
    (get Input "page-id" PageId)
    (get Input "app-id" AppId)
    (get Input "mode" Mode)
    (Qo.Page/set-policy-recursive AppId PageId Mode)
    (Qo.Public.Api.Page/perm {"app-id" AppId "page-id" PageId} Response))
```

- [ ] **Step 2: Verify both via scratch queries**

Single-page:

```
(Qo.Public.Api.Page/set-perm {"app-id" "..." "page-id" "..." "mode" "rwx---r--"} R)
(return [R])
```

Recursive on a folder with known descendants — verify all descendants have the new mode.

- [ ] **Step 3: Commit**

```bash
cd /home/ryan/Development/omega/query-omega-api
git add oql/app/public/api/page.oql
git commit -m "feat(api): Qo.Public.Api.Page/set-perm! + set-perm-recursive!"
```

---

### Task 10: Permission check in `page-by-id!` and `page-blocks!`

**Files:**
- Modify: `/home/ryan/Development/omega/query-omega-api/oql/app/public/api/page.oql`
- Modify: `/home/ryan/Development/omega/query-omega-api/oql/app/public/oql-api/page.oql`

When a caller's `AppId` is not the owning workspace, require `effective-policy[other] includes "r"`. Invisibility: if the check fails, behave as if the page doesn't exist.

- [ ] **Step 1: Wrap the existing page-by-id clause**

Rewrite the existing `Qo.Public.Api.Page/page-by-id!` clause (reference the current implementation in the file) so it branches on owner-vs-cross-workspace:

```
(:- (Qo.Public.Api.Page/page-by-id! Input Response)
    (get Input "page-id" PageId)
    (get Input "app-id" CallerAppId)
    (Qo.Page/page-object _ _ PageId Page)
    (get Page "app-id" OwnerAppId)
    (if (= CallerAppId OwnerAppId)
      (= Response Page)
      (and (Qo.Page/effective-policy OwnerAppId PageId Mode)
           (Qo.Page/mode-world-readable Mode true)
           (= Response Page))))
```

If the cross-workspace branch fails (not world-readable), the whole clause fails — the caller sees "no such page," matching Plan 9 invisibility.

- [ ] **Step 2: Mirror the check in page-blocks**

Similar wrap around the existing `page-blocks!` clause.

- [ ] **Step 3: Verify with a scratch query from workspace A for a page in workspace B**

On a workspace A test session, attempt to fetch a page in workspace B:
- First unpublished: expect failure (page appears to not exist).
- Publish the page (`set-perm` to `"rwx---r--"` from workspace B).
- Retry from workspace A: expect the page to be returned.

- [ ] **Step 4: Commit**

```bash
cd /home/ryan/Development/omega/query-omega-api
git add oql/app/public/api/page.oql oql/app/public/oql-api/page.oql
git commit -m "feat(api): enforce effective-policy[other +r] on cross-workspace reads"
```

---

### Task 11: `Qo.Public.Api.Page/search!` — cross-workspace search

**Files:**
- Modify: `/home/ryan/Development/omega/query-omega-api/oql/app/public/api/page.oql`

- [ ] **Step 1: Add the clause**

```
(:- (Qo.Public.Api.Page/search! Input Response)
    (get Input "query" Q)
    (Qo.Data.Page.WorldIndex/entry _ _ Entry)
    (get Entry "name" Name)
    (get Entry "text-excerpt" Excerpt)
    (or (string-contains-ci Name Q)
        (string-contains-ci Excerpt Q))
    (get Entry "app-id" AppId)
    (get Entry "page-id" PageId)
    (= Match {"app-id" AppId "page-id" PageId "name" Name})
    (fold [] Match append Matches)
    (= Response {"results" Matches}))
```

`string-contains-ci` may need to be a helper in util.oql; check existing utilities first. If case-insensitive substring isn't available, use `string-contains` (case-sensitive) in v1 and note as a future improvement.

- [ ] **Step 2: Verify search returns results**

After publishing some pages, run:

```
(Qo.Public.Api.Page/search {"query" "brief"} R)
(return [R])
```

Expect a non-empty `results` array with `{app-id, page-id, name}` entries.

- [ ] **Step 3: Commit**

```bash
cd /home/ryan/Development/omega/query-omega-api
git add oql/app/public/api/page.oql
git commit -m "feat(api): Qo.Public.Api.Page/search! — cross-workspace search"
```

---

## Phase 4 — CLI

### Task 12: `qo perm <page>` command

**Files:**
- Create: `/home/ryan/Development/omega/query-omega-cli/app/cmd/perm.js`
- Modify: `/home/ryan/Development/omega/query-omega-cli/index.js`

Mirrors the structure of `describe.js`. Takes a page path, resolves to a page-id, returns the effective policy.

- [ ] **Step 1: Create the command file**

```js
const { runQuery } = require("../api");
const { getWorkspaceId } = require("../workspace");
const { buildPathWalk } = require("../path");
const { outputResult, outputError, progress } = require("../output");

const permQuery = (rootPageId, pathSegments, appId) => {
  const { oql: pathWalk, targetVar } = buildPathWalk(pathSegments);
  return `
(datastore Qo.Public.Api.Page "omega/query-omega/public/api/page")
(datastore Qo.Public.OqlApi.Page "omega/query-omega/public/oql-api/page")

(= RootPageId "${rootPageId}")
${pathWalk}
(Qo.Public.Api.Page/perm {"app-id" "${appId}" "page-id" ${targetVar}} Response)
(return [Response])
`;
};

const permCommand = async (pagePath, options) => {
  const { host, envName } = options;
  const rootPageId = getWorkspaceId();
  const appId = process.env.QO_APP_ID;
  if (!appId) {
    outputError("perm", "QO_APP_ID not set in .env");
    return;
  }

  const segments = pagePath.split("/").filter(Boolean);
  if (segments.length === 0) {
    outputError("perm", "page path is required");
    return;
  }

  progress(`Reading permissions for ${pagePath}...`);
  const query = permQuery(rootPageId, segments, appId);
  const result = await runQuery({ host, envName, query, command: "perm" });
  const item = result[0];
  if (outputResult("perm", { path: pagePath, mode: item.mode })) return;
  console.log(`${item.mode}  ${pagePath}`);
};

module.exports.permCommand = permCommand;
```

- [ ] **Step 2: Wire into `index.js`**

Add near the other command registrations:

```js
const { permCommand } = require("./app/cmd/perm");
// ...
program
  .command("perm")
  .argument("<page-path>", "Page path")
  .description("Print the page's effective permission mode (owner group other)")
  .option(...hostOption)
  .option(...envOption)
  .action(permCommand);
```

- [ ] **Step 3: Verify against a dev workspace**

```bash
cd ~/Development/wttc/mab
qo perm "Component Installs/MAB Agent System/Brief"
```

Expect: `rwx------  Component Installs/MAB Agent System/Brief` (or whatever the current policy is).

- [ ] **Step 4: Commit**

```bash
cd /home/ryan/Development/omega/query-omega-cli
git add app/cmd/perm.js index.js
git commit -m "feat(cli): qo perm — show effective page policy"
```

---

### Task 13: `qo set-perm <page> <mode>` with `-R`

**Files:**
- Create: `/home/ryan/Development/omega/query-omega-cli/app/cmd/set-perm.js`
- Modify: `/home/ryan/Development/omega/query-omega-cli/index.js`

- [ ] **Step 1: Create the command file**

```js
const { runQuery } = require("../api");
const { getWorkspaceId } = require("../workspace");
const { buildPathWalk } = require("../path");
const { outputResult, outputError, progress } = require("../output");

const MODE_REGEX = /^[rwx-]{9}$/;

const setPermQuery = (rootPageId, pathSegments, appId, mode, recursive) => {
  const { oql: pathWalk, targetVar } = buildPathWalk(pathSegments);
  const clause = recursive
    ? "Qo.Public.Api.Page/set-perm-recursive"
    : "Qo.Public.Api.Page/set-perm";
  return `
(datastore Qo.Public.Api.Page "omega/query-omega/public/api/page")
(datastore Qo.Public.OqlApi.Page "omega/query-omega/public/oql-api/page")

(= RootPageId "${rootPageId}")
${pathWalk}
(${clause} {"app-id" "${appId}" "page-id" ${targetVar} "mode" "${mode}"} Response)
(return [Response])
`;
};

const setPermCommand = async (pagePath, mode, options) => {
  const { host, envName, recursive } = options;
  const rootPageId = getWorkspaceId();
  const appId = process.env.QO_APP_ID;
  if (!appId) {
    outputError("set-perm", "QO_APP_ID not set in .env");
    return;
  }
  if (!MODE_REGEX.test(mode)) {
    outputError("set-perm", `invalid mode: ${mode} (expected 9-char string, e.g. "rwx---r--")`);
    return;
  }

  const segments = pagePath.split("/").filter(Boolean);
  if (segments.length === 0) {
    outputError("set-perm", "page path is required");
    return;
  }

  progress(`Setting permissions ${mode} on ${pagePath}${recursive ? " (recursive)" : ""}...`);
  const query = setPermQuery(rootPageId, segments, appId, mode, !!recursive);
  const result = await runQuery({ host, envName, query, command: "set-perm" });
  const item = result[0];
  if (outputResult("set-perm", { path: pagePath, mode: item.mode, recursive: !!recursive })) return;
  console.log(`${item.mode}  ${pagePath}`);
};

module.exports.setPermCommand = setPermCommand;
```

- [ ] **Step 2: Wire into `index.js`**

```js
const { setPermCommand } = require("./app/cmd/set-perm");
// ...
program
  .command("set-perm")
  .argument("<page-path>", "Page path")
  .argument("<mode>", 'Mode string, e.g. "rwx---r--"')
  .description("Set the page's permission mode. Use -R for recursive.")
  .option(...hostOption)
  .option(...envOption)
  .option("-R, --recursive", "Apply to the page and every descendant")
  .action(setPermCommand);
```

- [ ] **Step 3: Verify both forms**

```bash
qo set-perm "Component Installs/MAB Agent System/Brief" rwx---r--
qo perm     "Component Installs/MAB Agent System/Brief"
# expect rwx---r--
qo set-perm -R "Component Installs/MAB Agent System" rwx---r--
# verify descendants all got the new mode
```

- [ ] **Step 4: Commit**

```bash
cd /home/ryan/Development/omega/query-omega-cli
git add app/cmd/set-perm.js index.js
git commit -m "feat(cli): qo set-perm (with -R) — write page permissions"
```

---

### Task 14: `qo search <query>` command

**Files:**
- Create: `/home/ryan/Development/omega/query-omega-cli/app/cmd/search.js`
- Modify: `/home/ryan/Development/omega/query-omega-cli/index.js`

- [ ] **Step 1: Create the command file**

```js
const { runQuery } = require("../api");
const { outputResult, outputError, progress } = require("../output");

const searchQuery = (q) => `
(datastore Qo.Public.Api.Page "omega/query-omega/public/api/page")
(Qo.Public.Api.Page/search {"query" "${q.replace(/"/g, '\\"')}"} Response)
(return [Response])
`;

const searchCommand = async (query, options) => {
  const { host, envName } = options;
  if (!query || !query.trim()) {
    outputError("search", "query is required");
    return;
  }

  progress(`Searching for "${query}"...`);
  const result = await runQuery({ host, envName, query: searchQuery(query), command: "search" });
  const first = result[0];
  const results = first?.results || [];
  if (outputResult("search", { query, results })) return;
  for (const r of results) {
    console.log(`${r["page-id"]}\t${r["app-id"]}\t${r.name}`);
  }
};

module.exports.searchCommand = searchCommand;
```

- [ ] **Step 2: Wire into `index.js`**

```js
const { searchCommand } = require("./app/cmd/search");
// ...
program
  .command("search")
  .argument("<query>", "Text to search for")
  .description("Search across all world-visible pages in OmegaAI")
  .option(...hostOption)
  .option(...envOption)
  .action(searchCommand);
```

- [ ] **Step 3: Verify**

After publishing some pages (`qo set-perm ... rwx---r--`), run:

```bash
qo search brief
```

Expect one or more result lines with `page-id<TAB>app-id<TAB>name`.

- [ ] **Step 4: Commit**

```bash
cd /home/ryan/Development/omega/query-omega-cli
git add app/cmd/search.js index.js
git commit -m "feat(cli): qo search — cross-workspace text search"
```

---

### Task 15: Extend `qo read` to accept a page-id

**Files:**
- Modify: `/home/ryan/Development/omega/query-omega-cli/app/cmd/read.js`

Today `qo read <page-path>` only works within the current workspace. Extend so a single argument shaped like a UUID is treated as a page-id and read directly — which, thanks to the permission check in Task 10, works cross-workspace as long as the page is world-readable.

- [ ] **Step 1: Detect page-id vs path**

Modify `readCommand` in `app/cmd/read.js`:

```js
const UUID_REGEX = /^[0-9a-f]{8}-[0-9a-f]{4}-[0-9a-f]{4}-[0-9a-f]{4}-[0-9a-f]{12}$/i;

const readCommand = async (pagePathOrId, options) => {
  const { host, envName } = options;

  if (UUID_REGEX.test(pagePathOrId)) {
    // Direct page-id path — no workspace walk needed.
    progress(`Reading blocks for page-id ${pagePathOrId}...`);
    const query = readByIdQuery(pagePathOrId);
    const result = await runQuery({ host, envName, command: "read", query });
    const blocks = result[0] || [];
    if (outputResult("read", { "page-id": pagePathOrId, blocks })) return;
    console.log(JSON.stringify(blocks, null, 2));
    return;
  }

  // Existing path-based flow (unchanged).
  // ... (keep existing implementation)
};
```

And add a `readByIdQuery`:

```js
const readByIdQuery = (pageId) => `
(datastore Qo.Public.OqlApi.Page "omega/query-omega/public/oql-api/page")

(Qo.Public.OqlApi.Page/page-by-id "${pageId}" Page)
(get Page "app-id" AppId)
(Qo.Public.OqlApi.Page/page-blocks AppId "${pageId}" Blocks)
(return [Blocks])
`;
```

- [ ] **Step 2: Verify**

Get a page-id from `qo search`, then:

```bash
qo read <page-id>
```

Expect the page's blocks to print.

- [ ] **Step 3: Commit**

```bash
cd /home/ryan/Development/omega/query-omega-cli
git add app/cmd/read.js
git commit -m "feat(cli): qo read accepts a page-id for direct cross-workspace reads"
```

---

## Final integration check

### Task 16: End-to-end verification against two dev workspaces

**Files:** none (verification only)

- [ ] **Step 1: From workspace A, publish a test page**

```bash
cd <workspace-A-project-dir>
qo set-perm "TestPublic/HelloWorld" rwx---r--
qo perm "TestPublic/HelloWorld"
# expect rwx---r--
```

- [ ] **Step 2: From workspace B, search and read**

```bash
cd <workspace-B-project-dir>
qo search hello
# expect a result for the page in workspace A

qo read <page-id-from-search>
# expect the block content, despite being in a different workspace
```

- [ ] **Step 3: From workspace B, attempt to read an unpublished page in workspace A**

Get the page-id of a private page in workspace A; from B:

```bash
qo read <private-page-id>
# expect: page-not-found error (Plan 9 invisibility, not 403)
```

- [ ] **Step 4: Open the AI panel and run the flagship scenario**

Ask "what is the researcher's brief for MAB?" (assuming the Brief page is published). The agent should:
- use `qo search brief` to find the target
- use `qo read <page-id>` to fetch its blocks
- render the response as markdown with proper headings and structure
- answer in under 3 tool calls

- [ ] **Step 5: No commit — this task is manual verification**

If any step fails, open an issue rather than continuing.

---

## Out of scope / follow-on plans

- **Namespace RBAC** — see sibling spec [2026-04-22-namespace-rbac-design.md](../specs/2026-04-22-namespace-rbac-design.md). Separate plan when that work is scheduled.
- **UI surfaces in query-omega** (permission controls on page settings, search bar in AI panel) — follow-on plan after this ships.
- **Symbolic mode ops in `qo set-perm`** (`o+r`, `755`) — if the 9-char absolute proves awkward in practice.
- **Cross-workspace `qo ls`** — needs a cross-workspace path walker; design separately.
