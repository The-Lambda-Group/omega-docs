# GHL Custom Field Queries Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Add three throwaway queries to the GoHighLevel Connector — `list-custom-fields`, `create-custom-field`, `delete-custom-field` — to unblock the HDC enrichment step.

**Architecture:** Each query is a new page under the Connector's `Queries/` folder with a Component, Query protocol binding, and OQL implementation. Each query inlines auth + `http-request` + flat-sequential-`with-table-if` dispatch following the existing `get-contact` pattern. No shared helper. No refactor of existing contact queries. Tests are live smoke runs against the configured GHL location — no mocking.

**Tech Stack:** OQL, `qo` CLI, GoHighLevel v2 API (`https://services.leadconnectorhq.com`), OmegaDB

**Spec:** [2026-04-11-ghl-custom-field-queries-design.md](../specs/2026-04-11-ghl-custom-field-queries-design.md)

---

## Constants referenced throughout this plan

Memorize these; every task uses them.

- **Project cwd for all `qo` commands:** `/home/ryan/Development/wttc/gohighlevel`
- **Parent `page.json` (the Queries folder in the library):** `impl/095bb6c8-gohighlevel-connector/755835ce-queries/page.json`
- **Query protocol ID:** `6701cfb9-247a-471e-a038-cd6eb2db0cf0` (visible in `impl/index.json` on every existing contact query)
- **Connector install path for `qo run`:** `Component Installs/GoHighLevel Connector/Queries/<query-name>`
- **Push connector name:** `GoHighLevel` (read via `Qo.Public.OqlApi.Page.Bl.PushCon/get-by-name`)
- **GHL base URL:** `https://services.leadconnectorhq.com`
- **GHL API version header:** `2021-07-28`

## Reference pattern

The reference implementation for all three queries is [`get-contact.oql`](../../../../../wttc/gohighlevel/impl/095bb6c8-gohighlevel-connector/755835ce-queries/8586ba3b-get-contact/65fd8e7d-implementation.oql). Every new query walks the same parent chain to read the push connector config, builds an `http-request` map, dispatches responses via flat sequential `with-table-if`s, and returns a normalized `{status, ...}` map.

## OQL gotchas that shape the implementation

Every OQL file in this plan bakes in these constraints from `projects/query-omega-oql/gotchas/` in the knowledge base:

1. **Use `http-request` only.** Never `http-get-request` / `http-post-request` — those silently drop status codes. See `http-request-only.md`.
2. **Flat sequential `with-table-if`s, never nested.** Nested branches hide which clause actually crashed in stack traces. See the HDC post-mortem.
3. **Every non-2xx status must be reachable by the dispatch.** The final `(defined Result)` fallback catches any status the explicit branches didn't handle (422, 500, etc.) and binds an error shape. See `four-twenty-two-falls-through-dispatch.md`.
4. **Literal maps/lists inside `throw` / `run-page` don't resolve symbols.** Always bind the map to a symbol first via `(= M {...})`, then pass `M`. Safe inside `(=)` itself — the literal-map gotcha only affects `throw` and `run-page`. See `mixed-literal-throw-run-page.md`.
5. **`with-table-if` schema must include a per-row identity.** For these single-row queries (only one `Exec` processed), the schema columns used here are enough — there's no multi-row collapse risk. But the pattern is kept consistent with multi-row impls so it ports later. See `with-table-if-schema-collapse.md`.
6. **GETs must NOT set `Content-Type`.** Cloudflare's WAF (in front of GHL) 400s GET requests with a Content-Type header. Use `Accept: application/json` on GETs. POST and DELETE may set `Content-Type: application/json`.
7. **Never use boolean `true` / `false` in OQL.** Always `"true"` / `"false"` as strings.

## File structure

Each query adds one directory with three files under the existing `impl/095bb6c8-gohighlevel-connector/755835ce-queries/` tree. `qo add-page` / `add-component` / `add-implementation` scaffold the directory and two of the three files plus the `index.json` entries; the OQL file gets written by hand.

```
impl/095bb6c8-gohighlevel-connector/755835ce-queries/
  <page-id>-list-custom-fields/
    page.json                                      (created by add-page)
    <component-id>-list-custom-fields.component.json  (created by add-component)
    <impl-id>-implementation.oql                   (hand-written after add-implementation)
  <page-id>-create-custom-field/
    (same structure)
  <page-id>-delete-custom-field/
    (same structure)
```

The `impl/index.json` file gets three new top-level entries as each query's page is added and updated twice more as the component and implementation register. Do not hand-edit `index.json` — the CLI commands update it.

---

## Task 1: `list-custom-fields`

**Why this is the first query:** read-only, smallest dispatch (only 2xx vs. error), and the smoke test doubles as an API discovery step — the response body shape from GHL informs how `create-custom-field` parses its response in Task 2.

**Files:**
- Create (via CLI): `impl/095bb6c8-gohighlevel-connector/755835ce-queries/<page-id>-list-custom-fields/page.json`
- Create (via CLI): `impl/095bb6c8-gohighlevel-connector/755835ce-queries/<page-id>-list-custom-fields/<component-id>-list-custom-fields.component.json`
- Modify (via CLI): `impl/index.json` (three updates as each add-* runs)
- Create (hand-written): `impl/095bb6c8-gohighlevel-connector/755835ce-queries/<page-id>-list-custom-fields/<impl-id>-implementation.oql`

- [ ] **Step 1: Scaffold the page**

```bash
cd /home/ryan/Development/wttc/gohighlevel
qo add-page impl/095bb6c8-gohighlevel-connector/755835ce-queries/page.json list-custom-fields
```

Expected: creates a new directory `impl/095bb6c8-gohighlevel-connector/755835ce-queries/<page-id-prefix>-list-custom-fields/` with `page.json`, and adds a new entry to `impl/index.json`. Note the new page's directory path from the command output or by `ls`-ing the queries folder.

- [ ] **Step 2: Scaffold the component**

```bash
qo add-component impl/095bb6c8-gohighlevel-connector/755835ce-queries/<page-dir>/page.json list-custom-fields
```

Replace `<page-dir>` with the actual directory from Step 1. Expected: creates `<component-id-prefix>-list-custom-fields.component.json` inside the query dir and updates `index.json` with the component entry. Note the component ID from the output (or read it from `index.json`).

- [ ] **Step 3: Scaffold the implementation binding**

```bash
qo add-implementation impl/095bb6c8-gohighlevel-connector/755835ce-queries/<page-dir>/page.json 6701cfb9-247a-471e-a038-cd6eb2db0cf0 <component-id>
```

`6701cfb9-...` is the Query protocol ID (hard-coded — it's the same for every query in the connector). `<component-id>` comes from Step 2.

Expected: updates `index.json` with an implementation entry. The entry has `file: "095bb6c8-gohighlevel-connector/755835ce-queries/<page-dir>/<impl-id-prefix>-implementation.oql"`. Read this path from `index.json` — it's where the OQL file goes.

- [ ] **Step 4: Write the OQL implementation**

Create the file at the path from Step 3 with the following content:

```oql
;; list-custom-fields — GoHighLevel Connector query
;; Takes: {}
;; Returns one of:
;;   {"status" "ok" "fields" [...]}
;;   {"status" "error" "code" <http-status> "body" "<raw body>"}
;;
;; GET /locations/{locationId}/customFields
;; Uses http-request (not deprecated variants) so we can branch on HTTP status.
;; Uses Accept: application/json (NOT Content-Type) on GETs per Cloudflare gotcha.

(datastore Qo.Public.OqlApi.Impl "omega/query-omega/public/oql-api/implementation")
(datastore Qo.Public.OqlApi.Page "omega/query-omega/public/oql-api/page")
(datastore Qo.Public.OqlApi.Page.Bl.PushCon "omega/query-omega/public/oql-api/page/block/push-connector")
(datastore Qo.Page "omega/query-omega/page")

(:- (Run Exec Result)
    (get Exec "args" Args)
    (get Args 0 Data)

    ;; Walk up to the GHL Connector page (grandparent: Queries → GHL Connector)
    (get-in Exec ["method" "page-id"] MyPageId)
    (Qo.Public.OqlApi.Page/page-by-id MyPageId MyPage)
    (get MyPage "folder-id" QueriesFolderId)
    (Qo.Page/page-object _ _ QueriesFolderId QueriesPage)
    (get QueriesPage "folder-id" ConnectorPageId)

    ;; Read GHL config from connector's push connector
    (Qo.Public.OqlApi.Page/page-by-id ConnectorPageId ConnectorPage)
    (Qo.Public.OqlApi.Page.Bl.PushCon/get-by-name ConnectorPage "GoHighLevel" GhlConfig)
    (get GhlConfig "api-key" ApiKey)
    (get GhlConfig "location-id" LocationId)

    ;; Build GET /locations/{locationId}/customFields
    ;;
    ;; HEADER NOTE: use Accept, NOT Content-Type, on this GET. Cloudflare
    ;; rejects GETs with a Content-Type header as a 400 Bad Request HTML
    ;; page before the request ever reaches GHL.
    (string-concat "https://services.leadconnectorhq.com/locations/" LocationId UrlA)
    (string-concat UrlA "/customFields" Url)
    (string-concat "Bearer " ApiKey AuthVal)
    (= Req {"method" "GET"
             "url" Url
             "headers" {"Authorization" AuthVal
                         "Version" "2021-07-28"
                         "Accept" "application/json"}})
    (http-request Req Res)
    (get Res "status" HttpStatus)
    (get Res "body" ResBody)

    ;; 2xx → parse body and extract the fields list.
    ;;
    ;; IMPORTANT: the exact key GHL returns the array under is not confirmed
    ;; in advance. First smoke test will print the raw body; adjust the
    ;; (get ParsedRes "customFields" ...) to the actual key if it differs.
    (with-table-if (or (= HttpStatus 200) (= HttpStatus 201))
      [HttpStatus ResBody SuccessFields]
      (and (json-stringify ParsedRes ResBody)
           (get ParsedRes "customFields" SuccessFields))
      (= true true))

    ;; Dispatch success → ok shape
    (with-table-if (defined SuccessFields)
      [SuccessFields Result]
      (= Result {"status" "ok" "fields" SuccessFields})
      (= true true))

    ;; Fallback: if Result is still not bound (any non-2xx, or 2xx with a
    ;; different body shape), emit an error shape. This catches 422, 500,
    ;; and any other status the explicit branches don't match.
    (with-table-if (defined Result)
      [Result HttpStatus ResBody]
      (= true true)
      (and (= ErrResult {"status" "error" "code" HttpStatus "body" ResBody})
           (= Result ErrResult))))

(= Clauses {"run" {"2" Run}})
(get $$ARG$$ "implementation-id" ImplId)
(Qo.Public.OqlApi.Impl/set-implementation-clauses ImplId Clauses Result)
```

- [ ] **Step 5: Push the implementation**

```bash
qo push impl/095bb6c8-gohighlevel-connector/755835ce-queries/<page-dir>/<impl-file>
```

Expected: the CLI reports the push succeeded with no errors. If there's an OQL parse error, the CLI prints a parse-error message pointing to the offending line — fix and re-push.

- [ ] **Step 6: Smoke test the query live**

```bash
qo run "Component Installs/GoHighLevel Connector/Queries/list-custom-fields" -p "Query" -m "run" -a '[{}]'
```

Expected: a JSON result in one of two shapes:

1. **Success (most likely):**
   ```json
   [{"status": "ok", "fields": [ {...}, {...} ]}]
   ```
   The `fields` array contains zero or more GHL custom field objects. Capture the shape of a single field object — it'll inform `create-custom-field`'s response parsing in Task 2.

2. **Error (response shape mismatch):**
   ```json
   [{"status": "error", "code": 200, "body": "{...raw body...}"}]
   ```
   This means the 2xx branch fired but `(get ParsedRes "customFields" ...)` failed. The raw body in the `body` field shows the actual key GHL is using. Read it, then:
   - Edit the OQL: change `"customFields"` to the actual key (could be `"custom_fields"`, `"data"`, etc.)
   - Re-push via `qo push`
   - Re-run the smoke test. Expect the success shape.

   A successful run with an empty array (`"fields": []`) also counts as success.

- [ ] **Step 7: If the Component Installs path is not found**

If `qo run` fails with "page not found" for `Component Installs/GoHighLevel Connector/Queries/list-custom-fields`, the new query page exists in the library but hasn't been linked into the installed tree. Re-run the connector's install (idempotent):

```bash
qo run "Component Installs/GoHighLevel Connector" -p "Installable" -m "install" -a '[]'
```

This should pick up the new query. Then retry Step 6.

If this step is not needed (query is found on the first `qo run`), skip it and move on.

- [ ] **Step 8: Commit**

```bash
cd /home/ryan/Development/wttc/gohighlevel
git add impl/095bb6c8-gohighlevel-connector/755835ce-queries/<page-dir>/ impl/index.json
git commit -m "feat(ghl): add list-custom-fields query"
```

---

## Task 2: `create-custom-field`

**Why this task depends on Task 1:** the response parsing in the OQL below uses `(get ParsedRes "customField" ...)` as a guess. The exact key for the created-field response may differ — we'll know after running `list-custom-fields` in Task 1 and seeing what shape GHL returns. Adjust if needed during Step 6 smoke test.

**Files:**
- Create (via CLI): `impl/095bb6c8-gohighlevel-connector/755835ce-queries/<page-id>-create-custom-field/page.json`
- Create (via CLI): `impl/095bb6c8-gohighlevel-connector/755835ce-queries/<page-id>-create-custom-field/<component-id>-create-custom-field.component.json`
- Modify (via CLI): `impl/index.json`
- Create (hand-written): `impl/095bb6c8-gohighlevel-connector/755835ce-queries/<page-id>-create-custom-field/<impl-id>-implementation.oql`

- [ ] **Step 1: Scaffold the page**

```bash
cd /home/ryan/Development/wttc/gohighlevel
qo add-page impl/095bb6c8-gohighlevel-connector/755835ce-queries/page.json create-custom-field
```

- [ ] **Step 2: Scaffold the component**

```bash
qo add-component impl/095bb6c8-gohighlevel-connector/755835ce-queries/<page-dir>/page.json create-custom-field
```

- [ ] **Step 3: Scaffold the implementation binding**

```bash
qo add-implementation impl/095bb6c8-gohighlevel-connector/755835ce-queries/<page-dir>/page.json 6701cfb9-247a-471e-a038-cd6eb2db0cf0 <component-id>
```

- [ ] **Step 4: Write the OQL implementation**

Create the file at the path from Step 3 with the following content:

```oql
;; create-custom-field — GoHighLevel Connector query
;; Takes: {"name" "...", "data-type" "TEXT", "model" "contact"}
;;        optionally with "extra" {...} — a map of any additional GHL fields in
;;        camelCase (placeholder, position, acceptedFormat, etc.) merged into
;;        the POST body.
;; Returns one of:
;;   {"status" "ok" "field" {...}}
;;   {"status" "error" "code" <http-status> "body" "<raw body>"}
;;
;; POST /locations/{locationId}/customFields

(datastore Qo.Public.OqlApi.Impl "omega/query-omega/public/oql-api/implementation")
(datastore Qo.Public.OqlApi.Page "omega/query-omega/public/oql-api/page")
(datastore Qo.Public.OqlApi.Page.Bl.PushCon "omega/query-omega/public/oql-api/page/block/push-connector")
(datastore Qo.Page "omega/query-omega/page")

(:- (Run Exec Result)
    (get Exec "args" Args)
    (get Args 0 Data)
    (get Data "name" FieldName)
    (get Data "data-type" DataType)
    (get Data "model" Model)

    ;; Walk up to the GHL Connector page
    (get-in Exec ["method" "page-id"] MyPageId)
    (Qo.Public.OqlApi.Page/page-by-id MyPageId MyPage)
    (get MyPage "folder-id" QueriesFolderId)
    (Qo.Page/page-object _ _ QueriesFolderId QueriesPage)
    (get QueriesPage "folder-id" ConnectorPageId)

    ;; Read GHL config from connector's push connector
    (Qo.Public.OqlApi.Page/page-by-id ConnectorPageId ConnectorPage)
    (Qo.Public.OqlApi.Page.Bl.PushCon/get-by-name ConnectorPage "GoHighLevel" GhlConfig)
    (get GhlConfig "api-key" ApiKey)
    (get GhlConfig "location-id" LocationId)

    ;; Build POST body. BaseBody always contains the three required fields
    ;; in camelCase (GHL wire format). If the caller provided "extra", merge
    ;; it on top so any additional GHL fields flow through.
    (= BaseBody {"name" FieldName "dataType" DataType "model" Model})

    (with-table-if (get Data "extra" Extra)
      [BaseBody Extra Body]
      (merge BaseBody Extra Body)
      (= Body BaseBody))

    ;; Serialize map → JSON string (json-stringify is bidirectional: if the
    ;; first arg is bound as a map, second becomes a string; if the second
    ;; is bound as a string, first becomes a parsed object).
    (json-stringify Body BodyJson)

    ;; Build POST /locations/{locationId}/customFields
    (string-concat "https://services.leadconnectorhq.com/locations/" LocationId UrlA)
    (string-concat UrlA "/customFields" Url)
    (string-concat "Bearer " ApiKey AuthVal)
    (= Req {"method" "POST"
             "url" Url
             "headers" {"Authorization" AuthVal
                         "Version" "2021-07-28"
                         "Content-Type" "application/json"}
             "body" BodyJson})
    (http-request Req Res)
    (get Res "status" HttpStatus)
    (get Res "body" ResBody)

    ;; 2xx → parse body and extract the field.
    ;;
    ;; NOTE: the exact key for the created-field response is not confirmed
    ;; in advance. If the smoke test shows the raw body has a different
    ;; key, adjust the (get ParsedRes "customField" ...) call below.
    (with-table-if (or (= HttpStatus 200) (= HttpStatus 201))
      [HttpStatus ResBody SuccessField]
      (and (json-stringify ParsedRes ResBody)
           (get ParsedRes "customField" SuccessField))
      (= true true))

    ;; Dispatch success → ok shape
    (with-table-if (defined SuccessField)
      [SuccessField Result]
      (= Result {"status" "ok" "field" SuccessField})
      (= true true))

    ;; Fallback: error for any non-2xx or 2xx-with-unexpected-shape
    (with-table-if (defined Result)
      [Result HttpStatus ResBody]
      (= true true)
      (and (= ErrResult {"status" "error" "code" HttpStatus "body" ResBody})
           (= Result ErrResult))))

(= Clauses {"run" {"2" Run}})
(get $$ARG$$ "implementation-id" ImplId)
(Qo.Public.OqlApi.Impl/set-implementation-clauses ImplId Clauses Result)
```

- [ ] **Step 5: Push the implementation**

```bash
qo push impl/095bb6c8-gohighlevel-connector/755835ce-queries/<page-dir>/<impl-file>
```

- [ ] **Step 6: Smoke test by creating a throwaway field**

Use a name with a timestamp suffix so reruns don't collide if a prior test field didn't get cleaned up:

```bash
qo run "Component Installs/GoHighLevel Connector/Queries/create-custom-field" -p "Query" -m "run" -a '[{"name": "test_delete_me_20260411", "data-type": "TEXT", "model": "contact"}]'
```

Expected: `[{"status": "ok", "field": {"id": "<ghl-field-id>", ...}}]`.

Capture the `id` — you'll need it for Task 3's delete test. Two common failure modes to watch for:

1. **`{"status": "error", "code": 200, "body": "..."}`** — the 2xx branch fired but `(get ParsedRes "customField" ...)` failed because GHL returned the created field under a different key. Read the raw body in `body`, update the OQL to use the actual key, `qo push` again, re-run.

2. **`{"status": "error", "code": 400, "body": "..."}`** — GHL rejected the create. Read `body` for the reason. Possible causes: field with that name already exists (re-run with a different timestamp suffix), invalid `data-type` value (GHL expects specific enum values like `TEXT`, `LARGE_TEXT`, `NUMERICAL`, etc.), or invalid `model` value. Adjust args and retry.

Also sanity-check that the new field shows up via `list-custom-fields`:

```bash
qo run "Component Installs/GoHighLevel Connector/Queries/list-custom-fields" -p "Query" -m "run" -a '[{}]'
```

Expected: the response's `fields` array includes a field with the name `test_delete_me_20260411`. If it does, the create path works end-to-end.

- [ ] **Step 7: Commit**

```bash
cd /home/ryan/Development/wttc/gohighlevel
git add impl/095bb6c8-gohighlevel-connector/755835ce-queries/<page-dir>/ impl/index.json
git commit -m "feat(ghl): add create-custom-field query"
```

---

## Task 3: `delete-custom-field`

**Files:**
- Create (via CLI): `impl/095bb6c8-gohighlevel-connector/755835ce-queries/<page-id>-delete-custom-field/page.json`
- Create (via CLI): `impl/095bb6c8-gohighlevel-connector/755835ce-queries/<page-id>-delete-custom-field/<component-id>-delete-custom-field.component.json`
- Modify (via CLI): `impl/index.json`
- Create (hand-written): `impl/095bb6c8-gohighlevel-connector/755835ce-queries/<page-id>-delete-custom-field/<impl-id>-implementation.oql`

- [ ] **Step 1: Scaffold the page**

```bash
cd /home/ryan/Development/wttc/gohighlevel
qo add-page impl/095bb6c8-gohighlevel-connector/755835ce-queries/page.json delete-custom-field
```

- [ ] **Step 2: Scaffold the component**

```bash
qo add-component impl/095bb6c8-gohighlevel-connector/755835ce-queries/<page-dir>/page.json delete-custom-field
```

- [ ] **Step 3: Scaffold the implementation binding**

```bash
qo add-implementation impl/095bb6c8-gohighlevel-connector/755835ce-queries/<page-dir>/page.json 6701cfb9-247a-471e-a038-cd6eb2db0cf0 <component-id>
```

- [ ] **Step 4: Write the OQL implementation**

Create the file at the path from Step 3 with the following content:

```oql
;; delete-custom-field — GoHighLevel Connector query
;; Takes: {"field-id" "<ghl-custom-field-id>"}
;; Returns one of:
;;   {"status" "ok"}
;;   {"status" "not-found"}
;;   {"status" "error" "code" <http-status> "body" "<raw body>"}
;;
;; DELETE /locations/{locationId}/customFields/{fieldId}
;;
;; No safety gate — delete is immediate. Callers manage their own discipline.

(datastore Qo.Public.OqlApi.Impl "omega/query-omega/public/oql-api/implementation")
(datastore Qo.Public.OqlApi.Page "omega/query-omega/public/oql-api/page")
(datastore Qo.Public.OqlApi.Page.Bl.PushCon "omega/query-omega/public/oql-api/page/block/push-connector")
(datastore Qo.Page "omega/query-omega/page")

(:- (Run Exec Result)
    (get Exec "args" Args)
    (get Args 0 Data)
    (get Data "field-id" FieldId)

    ;; Walk up to the GHL Connector page
    (get-in Exec ["method" "page-id"] MyPageId)
    (Qo.Public.OqlApi.Page/page-by-id MyPageId MyPage)
    (get MyPage "folder-id" QueriesFolderId)
    (Qo.Page/page-object _ _ QueriesFolderId QueriesPage)
    (get QueriesPage "folder-id" ConnectorPageId)

    ;; Read GHL config from connector's push connector
    (Qo.Public.OqlApi.Page/page-by-id ConnectorPageId ConnectorPage)
    (Qo.Public.OqlApi.Page.Bl.PushCon/get-by-name ConnectorPage "GoHighLevel" GhlConfig)
    (get GhlConfig "api-key" ApiKey)
    (get GhlConfig "location-id" LocationId)

    ;; Build DELETE /locations/{locationId}/customFields/{fieldId}
    (string-concat "https://services.leadconnectorhq.com/locations/" LocationId UrlA)
    (string-concat UrlA "/customFields/" UrlB)
    (string-concat UrlB FieldId Url)
    (string-concat "Bearer " ApiKey AuthVal)
    (= Req {"method" "DELETE"
             "url" Url
             "headers" {"Authorization" AuthVal
                         "Version" "2021-07-28"
                         "Accept" "application/json"}})
    (http-request Req Res)
    (get Res "status" HttpStatus)
    (get Res "body" ResBody)

    ;; 2xx → success (DELETE typically returns 200 with a minimal body or 204;
    ;; we don't care about the body content, only that the status is 2xx)
    (with-table-if (or (= HttpStatus 200) (= HttpStatus 204))
      [HttpStatus Success]
      (= Success "true")
      (= true true))

    ;; 404 → not-found (field ID doesn't exist in the location)
    (with-table-if (= HttpStatus 404)
      [HttpStatus NotFound]
      (= NotFound "true")
      (= true true))

    ;; Dispatch success → ok shape
    (with-table-if (defined Success)
      [Success Result]
      (= Result {"status" "ok"})
      (= true true))

    ;; Dispatch not-found
    (with-table-if (defined NotFound)
      [NotFound Result]
      (= Result {"status" "not-found"})
      (= true true))

    ;; Fallback: error
    (with-table-if (defined Result)
      [Result HttpStatus ResBody]
      (= true true)
      (and (= ErrResult {"status" "error" "code" HttpStatus "body" ResBody})
           (= Result ErrResult))))

(= Clauses {"run" {"2" Run}})
(get $$ARG$$ "implementation-id" ImplId)
(Qo.Public.OqlApi.Impl/set-implementation-clauses ImplId Clauses Result)
```

- [ ] **Step 5: Push the implementation**

```bash
qo push impl/095bb6c8-gohighlevel-connector/755835ce-queries/<page-dir>/<impl-file>
```

- [ ] **Step 6: Smoke test by deleting the throwaway field from Task 2**

Use the `id` captured in Task 2 Step 6:

```bash
qo run "Component Installs/GoHighLevel Connector/Queries/delete-custom-field" -p "Query" -m "run" -a '[{"field-id": "<ghl-field-id-from-task-2>"}]'
```

Expected: `[{"status": "ok"}]`.

Failure modes:

1. **`[{"status": "not-found"}]`** — the field ID doesn't exist. Either the ID was wrong or the field was already deleted.
2. **`[{"status": "error", "code": 2xx, ...}]`** — the 2xx branch didn't cover the status GHL actually returned (maybe it returns 202 or an unusual status on delete). Read the `code` field, add it to the success branch's `(or ...)` list, `qo push` again, re-run.

- [ ] **Step 7: Verify deletion via list-custom-fields**

```bash
qo run "Component Installs/GoHighLevel Connector/Queries/list-custom-fields" -p "Query" -m "run" -a '[{}]'
```

Expected: the `fields` array does NOT contain a field with the name `test_delete_me_20260411`. If it does, the delete didn't actually take effect on GHL's side — check the delete response body for clues.

- [ ] **Step 8: Commit**

```bash
cd /home/ryan/Development/wttc/gohighlevel
git add impl/095bb6c8-gohighlevel-connector/755835ce-queries/<page-dir>/ impl/index.json
git commit -m "feat(ghl): add delete-custom-field query"
```

---

## Task 4: End-to-end cleanup sanity check

**Why this task:** Task 3's smoke test already did a create → delete → verify cycle, but this task exercises a clean create → delete against a brand new field to confirm the full loop works independently of Task 2's test field.

**No new files.** This task is pure live smoke testing of the three queries in sequence.

- [ ] **Step 1: Create a new throwaway test field with a fresh timestamp suffix**

```bash
cd /home/ryan/Development/wttc/gohighlevel
qo run "Component Installs/GoHighLevel Connector/Queries/create-custom-field" -p "Query" -m "run" -a '[{"name": "test_e2e_20260411", "data-type": "TEXT", "model": "contact"}]'
```

Expected: `[{"status": "ok", "field": {"id": "<new-id>", ...}}]`. Capture `<new-id>`.

- [ ] **Step 2: Confirm it appears in list**

```bash
qo run "Component Installs/GoHighLevel Connector/Queries/list-custom-fields" -p "Query" -m "run" -a '[{}]'
```

Expected: the `fields` array contains an entry named `test_e2e_20260411` with the same ID from Step 1.

- [ ] **Step 3: Delete it**

```bash
qo run "Component Installs/GoHighLevel Connector/Queries/delete-custom-field" -p "Query" -m "run" -a '[{"field-id": "<new-id>"}]'
```

Expected: `[{"status": "ok"}]`.

- [ ] **Step 4: Confirm it's gone**

```bash
qo run "Component Installs/GoHighLevel Connector/Queries/list-custom-fields" -p "Query" -m "run" -a '[{}]'
```

Expected: the `fields` array does NOT contain `test_e2e_20260411`.

- [ ] **Step 5: Attempt to delete an already-deleted field (error path check)**

```bash
qo run "Component Installs/GoHighLevel Connector/Queries/delete-custom-field" -p "Query" -m "run" -a '[{"field-id": "<new-id>"}]'
```

Using the same ID from Step 1 — it's now deleted, so GHL should 404. Expected: `[{"status": "not-found"}]`. If instead this returns `{"status": "error", "code": 400, ...}` or similar, GHL's non-existent-ID behavior is different than assumed — note it and adjust the delete query's 404 branch if needed.

- [ ] **Step 6: Commit nothing (this is verification only)**

No code changed. If any step revealed a behavior mismatch and the OQL was edited and pushed, commit those edits with a descriptive message like `fix(ghl): adjust delete-custom-field not-found branch for <status>`.

---

## After this plan

The three queries are throwaway at session-close intent but immediately useful for the HDC enrichment step. Once this plan is complete:

1. **Provision HDC's seven custom fields.** Run `create-custom-field` seven times with `data-type: "TEXT"` and `model: "contact"` for each of: `linkedin_url`, `facebook_url`, `twitter_url`, `pinterest_url`, `score`, `closed_sales`, `total_value`. This is a one-off operator action, not code — just run the commands and verify each one returns `{"status": "ok"}`. Capture the field IDs from the responses.

2. **Update the HDC README** to remove the "manual GHL UI setup required" note for custom fields, replacing it with a reference to these three new queries.

3. **Return to the HDC enrichment step design** and begin planning its implementation. The enrichment step will `list-custom-fields` at the start of each batch to build a name → field-id map, then PATCH each contact with the appropriate field values.

None of the above is part of this plan's scope — this plan ends when Task 4 passes.
