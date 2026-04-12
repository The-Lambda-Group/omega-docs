# HDC Enrichment Step Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Build the enrichment pipeline step that PATCHes GHL contacts with social media links and sales performance data as custom fields, completing the HDC customer delivery.

**Architecture:** A new Event Handler page (`Enrich GHL Contacts`) under the HDC Connector's Events/ folder. Reads the GHL Contacts ledger for contact IDs, joins raw data (Homes.com Data) and scored data (Scored Agents) per agent, builds a `customFields` array with only non-empty values, PUTs to GHL. Tracks progress via an `enriched_at` column on the existing ledger. Coordinator gets a new `"enrich"` step for automatic pipeline continuation after the batch (sync) step completes.

**Tech Stack:** OQL, `qo` CLI, GoHighLevel v2 API (`PUT /contacts/{id}`), OmegaDB

**Design spec:** [`projects/homes-dot-com/explanation/enrich-ghl-contacts-design.md`](../../../../omega-knowledge-base/projects/homes-dot-com/explanation/enrich-ghl-contacts-design.md) in the knowledge base, and "Pipeline → 4. Enrich GHL Contacts" in the [HDC repo README](../../../../../wttc/homes-dot-com/README.md).

**Spec for custom-field queries (dependency):** [2026-04-11-ghl-custom-field-queries-design.md](../specs/2026-04-11-ghl-custom-field-queries-design.md) — already implemented, all three queries deployed and working.

---

## Constants

- **HDC project cwd:** `/home/ryan/Development/wttc/homes-dot-com`
- **GHL connector cwd:** `/home/ryan/Development/wttc/gohighlevel`
- **Event Handler protocol ID:** `8795affb-6274-41ea-879d-d31b362c8695`
- **Query protocol ID:** `6701cfb9-247a-471e-a038-cd6eb2db0cf0`
- **GHL base URL:** `https://services.leadconnectorhq.com`
- **GHL API version header:** `2021-07-28`
- **Connector install path for `qo run`:** `Component Installs/Homes.Com Connector/Events/<name>`
- **GHL Contacts ledger:** `Component Installs/Homes.Com Connector/Data/GHL Contacts` (pkey: `agent_profile_link`)
- **Push connector name on HDC install page:** `GoHighLevel Connector`

## OQL patterns — mandatory reading

Read these before writing any OQL in this plan. All are in the knowledge base at `~/Development/omega/omega-knowledge-base/`:

- `projects/query-omega-oql/gotchas/no-not-operator.md` — OQL has no `not`. Use else-branch.
- `projects/query-omega-oql/gotchas/defined-is-universal.md` — never `(defined X)` in conditions
- `projects/query-omega-oql/reference/with-table-if-body-scope.md` — schema is the interface, body is its own scope
- `projects/query-omega-oql/gotchas/four-twenty-two-falls-through-dispatch.md` — handle all HTTP statuses
- `projects/query-omega-oql/how-to/debug-by-returning-early.md` — comment out + bind Result to tuple

The canonical reference implementation is the fixed `ghl-sync` at:
`/home/ryan/Development/wttc/homes-dot-com/impl/18f2e4bb-homes-com-component-library/a31e2741-events-library/8e5821d0-ghl-sync/bfe95f5a-implementation.oql`

## The 7 custom fields

| Custom field name | Source table | Source column | GHL `field_key` |
|---|---|---|---|
| `linkedin_url` | Homes.com Data | `linked_in` | `contact.linkedin_url` |
| `facebook_url` | Homes.com Data | `facebook` | `contact.facebook_url` |
| `twitter_url` | Homes.com Data | `twitter` | `contact.twitter_url` |
| `pinterest_url` | Homes.com Data | `pinterest` | `contact.pinterest_url` |
| `score` | Scored Agents | `score` | `contact.score` |
| `closed_sales` | Homes.com Data | `closed_sales` | `contact.closed_sales` |
| `total_value` | Homes.com Data | `total_value` | `contact.total_value` |

All are `TEXT` type on the `contact` model. The `field_key` is `contact.<field_name>` (GHL auto-generates this from the field name at creation time — confirmed in Task 2's smoke test where the create response included `"fieldKey": "contact.test_delete_me_20260411"`).

---

## Task 1: Create the 7 custom fields in GHL

**No code files.** This task provisions the fields in the live GHL location using the `create-custom-field` query deployed in the previous plan.

- [ ] **Step 1: Create all 7 fields**

Run each command from the GHL connector project directory:

```bash
cd /home/ryan/Development/wttc/gohighlevel

qo run "Component Installs/GoHighLevel Connector/Queries/create-custom-field" -p "Query" -m "run" -a '[{"name": "linkedin_url", "data-type": "TEXT", "model": "contact"}]'

qo run "Component Installs/GoHighLevel Connector/Queries/create-custom-field" -p "Query" -m "run" -a '[{"name": "facebook_url", "data-type": "TEXT", "model": "contact"}]'

qo run "Component Installs/GoHighLevel Connector/Queries/create-custom-field" -p "Query" -m "run" -a '[{"name": "twitter_url", "data-type": "TEXT", "model": "contact"}]'

qo run "Component Installs/GoHighLevel Connector/Queries/create-custom-field" -p "Query" -m "run" -a '[{"name": "pinterest_url", "data-type": "TEXT", "model": "contact"}]'

qo run "Component Installs/GoHighLevel Connector/Queries/create-custom-field" -p "Query" -m "run" -a '[{"name": "score", "data-type": "TEXT", "model": "contact"}]'

qo run "Component Installs/GoHighLevel Connector/Queries/create-custom-field" -p "Query" -m "run" -a '[{"name": "closed_sales", "data-type": "TEXT", "model": "contact"}]'

qo run "Component Installs/GoHighLevel Connector/Queries/create-custom-field" -p "Query" -m "run" -a '[{"name": "total_value", "data-type": "TEXT", "model": "contact"}]'
```

Expected: each returns `[{"status": "ok", "field": {"id": "...", "fieldKey": "contact.<name>", ...}}]`. Capture the `fieldKey` from each response — they should match the table above.

If any returns an error with `code: 400` and "field already exists," the field was created in a prior run or manually. That's fine — skip it and verify via list.

- [ ] **Step 2: Verify all 7 fields exist**

```bash
qo run "Component Installs/GoHighLevel Connector/Queries/list-custom-fields" -p "Query" -m "run" -a '[{}]'
```

Expected: the `fields` array contains 7 entries with names `linkedin_url`, `facebook_url`, `twitter_url`, `pinterest_url`, `score`, `closed_sales`, `total_value`. All `TEXT` type, all `contact` model. If any are missing, re-run the create command for that field.

---

## Task 2: Document custom field creation in KB and README

**Files:**
- Modify: `~/Development/omega/omega-knowledge-base/projects/homes-dot-com/explanation/enrich-ghl-contacts-design.md`
- Modify: `~/Development/wttc/homes-dot-com/README.md`

- [ ] **Step 1: Update the enrichment design doc**

In `enrich-ghl-contacts-design.md`, change the "Custom field dependency" section from:

> GHL custom fields must already exist in the location before this step runs. **They are NOT provisioned by this pipeline's installer.**

To:

> GHL custom fields are provisioned programmatically via the `create-custom-field` query in the GoHighLevel Connector (see Task 1 of the enrichment implementation plan). The 7 fields (`linkedin_url`, `facebook_url`, `twitter_url`, `pinterest_url`, `score`, `closed_sales`, `total_value`) are created as TEXT fields on the contact model. Their `field_key` values (`contact.<name>`) are used at runtime to populate the update payload — no field IDs need to be cached.

- [ ] **Step 2: Update the HDC README**

In the README's "Setup → Prerequisites → GHL custom fields (manual setup, required for enrichment)" section, replace the manual-UI-setup instructions with a note that the fields are now created programmatically:

> #### GHL custom fields (created programmatically)
>
> The enrichment pipeline step needs specific custom fields to exist in the GHL location. These are created via the GoHighLevel Connector's `create-custom-field` query before the enrichment step runs — see the 7 commands in Task 1 of the enrichment implementation plan. Once created, they persist in the GHL location and do not need to be re-created.

- [ ] **Step 3: Commit both changes**

```bash
git -C ~/Development/omega/omega-knowledge-base add projects/homes-dot-com/explanation/enrich-ghl-contacts-design.md
git -C ~/Development/omega/omega-knowledge-base commit -m "docs: custom fields now created programmatically via create-custom-field query"

git -C ~/Development/wttc/homes-dot-com add README.md
git -C ~/Development/wttc/homes-dot-com commit -m "docs: custom fields now created programmatically"
```

---

## Task 3: Scaffold the Enrich GHL Contacts page

**Files:**
- Create (via CLI + pull): `impl/18f2e4bb-homes-com-component-library/a31e2741-events-library/<page-id>-enrich-ghl-contacts/page.json`
- Create (via CLI + pull): `impl/18f2e4bb-homes-com-component-library/a31e2741-events-library/<page-id>-enrich-ghl-contacts/<component-id>-enrich-ghl-contacts.component.json`
- Modify (via CLI): `impl/index.json`
- Create (hand-written in Task 4): `impl/18f2e4bb-homes-com-component-library/a31e2741-events-library/<page-id>-enrich-ghl-contacts/<impl-id>-implementation.oql`

Note: all scaffolding commands run from the **homes-dot-com** project, not gohighlevel. The enrichment step is part of the HDC connector, not the GHL connector.

- [ ] **Step 1: Find the Events folder page.json path**

The Events folder in the HDC connector is `impl/18f2e4bb-homes-com-component-library/a31e2741-events-library/`. Verify it has a `page.json`:

```bash
ls /home/ryan/Development/wttc/homes-dot-com/impl/18f2e4bb-homes-com-component-library/a31e2741-events-library/page.json
```

- [ ] **Step 2: Scaffold the page**

```bash
cd /home/ryan/Development/wttc/homes-dot-com
qo add-page impl/18f2e4bb-homes-com-component-library/a31e2741-events-library/page.json "Enrich GHL Contacts"
qo pull
```

Note the new page directory name from the output or `ls`.

- [ ] **Step 3: Scaffold the component**

```bash
cd /home/ryan/Development/wttc/homes-dot-com
qo add-component impl/18f2e4bb-homes-com-component-library/a31e2741-events-library/<page-dir>/page.json "Enrich GHL Contacts"
qo pull
```

Note the component ID from index.json.

- [ ] **Step 4: Scaffold the implementation binding**

```bash
cd /home/ryan/Development/wttc/homes-dot-com
qo add-implementation impl/18f2e4bb-homes-com-component-library/a31e2741-events-library/<page-dir>/page.json 8795affb-6274-41ea-879d-d31b362c8695 <component-id>
qo pull
```

`8795affb-...` is the **Event Handler** protocol ID (NOT the Query protocol — enrichment is an event handler like GHL Sync, not a query like list-custom-fields).

Read the new implementation entry from `impl/index.json` to get the OQL file path.

- [ ] **Step 5: Register in the install impl**

Read the HDC connector install impl at `impl/18f2e4bb-homes-com-component-library/efbde608-homes-com-data-connector/bdae504d-implementation.oql`. Find the pattern where sibling event pages (Score Agents, Filter Agents, GHL Sync) are registered. Add a matching block for "Enrich GHL Contacts" with the new component ID. Push the install impl, then re-run install:

```bash
cd /home/ryan/Development/wttc/homes-dot-com
qo push impl/18f2e4bb-homes-com-component-library/efbde608-homes-com-data-connector/bdae504d-implementation.oql
qo run "Component Installs/Homes.Com Connector" -p "Installable" -m "install" -a '[]'
```

Verify the new page is reachable:

```bash
qo ls "Component Installs/Homes.Com Connector/Events" -v
```

Expected: "Enrich GHL Contacts" appears in the listing.

---

## Task 4: Write the enrichment OQL

This is the core task. The OQL reads from the GHL Contacts ledger, joins raw + scored data, builds a customFields payload, PUTs to GHL, and writes `enriched_at` back to the ledger.

**File:** `impl/18f2e4bb-homes-com-component-library/a31e2741-events-library/<page-dir>/<impl-id>-implementation.oql`

**Structure overview before the full code:**

```
1. Parse args (skip, limit)
2. Resolve page tree → find Data folders, Event Listener, push connector
3. Bind GHL config (api-key, location-id, auth header)
4. Find print-stream query page for logging
5. Log start
6. Read GHL Contacts ledger (paginated via page-query)
7. Per row:
   a. Read agent_profile_link, ghl_contact_id, enriched_at
   b. DEDUP: skip if enriched_at is already set
   c. Skip if ghl_contact_id = "skipped" (never created in GHL)
   d. Look up raw data from Homes.com Data via row-index
   e. Look up scored data from Scored Agents via row-index
   f. Build customFields array (7 conditional appends — only non-empty values)
   g. PUT to GHL: /contacts/{ghl_contact_id} with {customFields: [...]}
   h. Classify response (2xx = ok, else = error)
   i. Write enriched_at to the GHL Contacts ledger row
8. Aggregate: count enriched, count api-calls, has-more
9. Log done
10. Return batch summary
```

- [ ] **Step 1: Write the OQL implementation**

Create the file at the path from Task 3 Step 4. The full OQL is below.

**IMPORTANT implementation notes BEFORE reading the code:**

- The `customFields` array is built via 7 sequential flat `with-table-if`s (lines marked CF1–CF7 in comments). Each one checks if the source column has a value, and if so appends a `{key, value}` pair to the running array. If the source column is empty/missing, the array passes through unchanged. This is the only way to build a variable-length array in OQL without nesting.
- The GHL update uses `PUT /contacts/{id}` with `{"customFields": [...]}` in the body. The `key` field in each array element is the `field_key` format (`"contact.linkedin_url"`, etc.). **If GHL rejects this format during smoke test, fall back to using the `id` format instead — call `list-custom-fields` to get the field IDs, and replace `"key"` with `"id"` and the field_key with the UUID.**
- The `enriched_at` write uses `write-table` on the GHL Contacts ledger with the full row header including the new `enriched_at` column. OmegaDB adds columns implicitly on first write.
- The `Ctx` map threads per-invocation config into every `with-table-if` schema, following the ghl-sync pattern. This is the reliable way to get outer-scope values into branch bodies — don't rely on lexical capture.
- 422 responses from GHL are dropped via `(= true false)` in their with-table-if then-branch, same as ghl-sync.

```oql
(datastore Qo.Public.OqlApi.Impl "omega/query-omega/public/oql-api/implementation")
(datastore Qo.Public.OqlApi.Page "omega/query-omega/public/oql-api/page")
(datastore Qo.Public.OqlApi.Page.Bl.PushCon "omega/query-omega/public/oql-api/page/block/push-connector")
(datastore Qo.Public.OqlApi.Db.Prop "omega/query-omega/public/oql-api/database/property")
(datastore Qo.Public.OqlApi.Proto "omega/query-omega/public/oql-api/protocol")
(datastore Qo.Public.Api.Run "omega/query-omega/public/api/run")
(datastore Qo.Db.Prop "omega/query-omega/database/property")
(datastore Qo.Page "omega/query-omega/page")

;; Enrich GHL Contacts — backfill custom fields (social links + sales data)
;;
;; Reads the GHL Contacts ledger, joins Homes.com Data + Scored Agents,
;; PUTs custom fields to each GHL contact. Tracks progress via enriched_at.
;;
;; Args:
;;   skip  — offset into GHL Contacts ledger (default 0)
;;   limit — batch size (default 10)
;;
;; Returns:
;;   status, contacts-enriched, api-calls, has-more, next-skip

(:- (OnEvent Exec Result)
    (get-in Exec ["method" "page-id"] PageId)
    (get Exec "args" Args)
    (get Args 0 Data)
    (when-not (get Data "skip" Skip) (= Skip 0))
    (when-not (get Data "limit" Limit) (= Limit 10))

    ;; Resolve page tree: Enrich → Events → Install Page
    (Qo.Public.OqlApi.Page/page-by-id PageId EnrichPage)
    (get EnrichPage "folder-id" EventsFolderId)
    (Qo.Page/page-object _ _ EventsFolderId EventsFolder)
    (get EventsFolder "folder-id" InstallPageId)

    ;; Data folders
    (Qo.Page/page-object _ InstallPageId DataFolderId DataFolder)
    (get-in DataFolder ["page-data" "name"] "Data")
    (Qo.Page/page-object _ DataFolderId GhlContactsFolderId GhlContactsPage)
    (get-in GhlContactsPage ["page-data" "name"] "GHL Contacts")
    (Qo.Page/page-object _ DataFolderId ScoredFolderId ScoredPage)
    (get-in ScoredPage ["page-data" "name"] "Scored Agents")

    ;; Raw data folder (child of install page, not Data/)
    (Qo.Page/page-object _ InstallPageId RawFolderId RawDataPage)
    (get-in RawDataPage ["page-data" "name"] "Homes.com Data")

    ;; Event Listener (for logging)
    (Qo.Page/page-object _ InstallPageId EventListenerPageId EventListenerPageData)
    (get-in EventListenerPageData ["page-data" "name"] "Homes.com Event Listener")

    ;; Push connector → GHL config
    (Qo.Public.OqlApi.Page/page-by-id InstallPageId InstallPage)
    (Qo.Public.OqlApi.Page.Bl.PushCon/get-by-name InstallPage "GoHighLevel Connector" GhlConfig)
    (get GhlConfig "ghl-location-id" GhlLocationId)
    (get GhlConfig "ghl-api-key" GhlApiKey)
    (string-concat "Bearer " GhlApiKey AuthHeader)
    (get GhlConfig "omega-queries-page-id" OmegaQueriesPageId)

    ;; print-stream query page (for logging)
    (Qo.Page/page-object _ OmegaQueriesPageId QueriesFolderId QueriesFolder)
    (get-in QueriesFolder ["page-data" "name"] "Queries")
    (Qo.Page/page-object _ QueriesFolderId PrintStreamPageId PrintStreamPageData)
    (get-in PrintStreamPageData ["page-data" "name"] "print-stream")
    (Qo.Public.OqlApi.Page/page-by-id PrintStreamPageId PrintStreamPage)
    (Qo.Public.OqlApi.Proto/protocol _ "6701cfb9-247a-471e-a038-cd6eb2db0cf0" QueryProto)

    ;; Log start
    (= StartMsg ["enrich:start" "skip" Skip "limit" Limit])
    (= StartArgs [{"page-id" EventListenerPageId "message" StartMsg}])
    (Qo.Public.Api.Run/run-page QueryProto PrintStreamPage "run" StartArgs _)

    ;; Context map — threaded into every with-table-if schema
    (= Ctx {"ghl-contacts-folder-id" GhlContactsFolderId
            "raw-folder-id" RawFolderId
            "scored-folder-id" ScoredFolderId
            "auth-header" AuthHeader})

    ;; Read from GHL Contacts ledger (paginated)
    (= Q {"folder-id" GhlContactsFolderId "skip" Skip "limit" Limit})
    (Qo.Public.OqlApi.Page/page-query Q ContactPage)
    (get ContactPage "page-id" ContactPageId)
    (Qo.Db.Prop/prop-vals _ ContactPageId ContactPropVals)
    (get ContactPropVals "prop-vals" ContactPV)
    (get ContactPV "agent_profile_link" AgentLink)
    (get ContactPV "ghl_contact_id" GhlContactId)
    (get ContactPV "agent_full_name" AgentName)
    (get ContactPV "synced_at" SyncedAt)

    ;; ================================================================
    ;; DEDUP — skip if already enriched.
    ;; Every ledger row has a valid GHL UUID as ghl_contact_id (ghl-sync
    ;; only writes rows that were successfully created or claimed). No
    ;; need to check for ghl_contact_id = "skipped".
    ;; ================================================================
    (with-table-if (get ContactPV "enriched_at" _)
      [AgentLink ContactPV EnrichStatus ApiCallIncrement]
      (and (= EnrichStatus "skipped")
           (= ApiCallIncrement 0))
      (and (= EnrichStatus "enrich")
           (= ApiCallIncrement 1)))

    ;; ================================================================
    ;; DATA LOOKUP — join raw + scored data for "enrich" rows
    ;; ================================================================
    (with-table-if (= EnrichStatus "enrich")
      [AgentLink Ctx EnrichStatus RawPV ScoredPV]
      (and (= RowKey [AgentLink])
           (get Ctx "raw-folder-id" LookupRawFolderId)
           (get Ctx "scored-folder-id" LookupScoredFolderId)
           (Qo.Db.Prop/row-index LookupRawFolderId RowKey RawPageId)
           (Qo.Db.Prop/prop-vals _ RawPageId RawData)
           (get RawData "prop-vals" RawPV)
           (Qo.Db.Prop/row-index LookupScoredFolderId RowKey ScoredPageId)
           (Qo.Db.Prop/prop-vals _ ScoredPageId ScoredData)
           (get ScoredData "prop-vals" ScoredPV))
      (= true true))

    ;; ================================================================
    ;; BUILD customFields ARRAY — 7 conditional appends (CF1–CF7)
    ;; Each checks if the source column exists and is non-empty,
    ;; then appends a {key, value} pair. Non-existent or empty fields
    ;; are skipped (the array passes through unchanged).
    ;; Only runs for "enrich" rows; "skipped" rows pass through all 7.
    ;; ================================================================

    ;; Initialize the custom fields array before the conditional-append chain
    (= Fields0 [])

    ;; CF1: linkedin_url
    (with-table-if (and (= EnrichStatus "enrich")
                        (get RawPV "linked_in" _))
      [AgentLink EnrichStatus RawPV Fields0 Fields1]
      (and (get RawPV "linked_in" V)
           (= P {"key" "contact.linkedin_url" "value" V})
           (fold [] P append Fields1))
      (= Fields1 Fields0))

    ;; CF2: facebook_url
    (with-table-if (and (= EnrichStatus "enrich")
                        (get RawPV "facebook" _))
      [AgentLink EnrichStatus RawPV Fields1 Fields2]
      (and (get RawPV "facebook" V)
           (= P {"key" "contact.facebook_url" "value" V})
           (fold Fields1 P append Fields2))
      (= Fields2 Fields1))

    ;; CF3: twitter_url
    (with-table-if (and (= EnrichStatus "enrich")
                        (get RawPV "twitter" _))
      [AgentLink EnrichStatus RawPV Fields2 Fields3]
      (and (get RawPV "twitter" V)
           (= P {"key" "contact.twitter_url" "value" V})
           (fold Fields2 P append Fields3))
      (= Fields3 Fields2))

    ;; CF4: pinterest_url
    (with-table-if (and (= EnrichStatus "enrich")
                        (get RawPV "pinterest" _))
      [AgentLink EnrichStatus RawPV Fields3 Fields4]
      (and (get RawPV "pinterest" V)
           (= P {"key" "contact.pinterest_url" "value" V})
           (fold Fields3 P append Fields4))
      (= Fields4 Fields3))

    ;; CF5: score (from Scored Agents, not raw data)
    (with-table-if (and (= EnrichStatus "enrich")
                        (get ScoredPV "score" _))
      [AgentLink EnrichStatus ScoredPV Fields4 Fields5]
      (and (get ScoredPV "score" V)
           (= P {"key" "contact.score" "value" V})
           (fold Fields4 P append Fields5))
      (= Fields5 Fields4))

    ;; CF6: closed_sales
    (with-table-if (and (= EnrichStatus "enrich")
                        (get RawPV "closed_sales" _))
      [AgentLink EnrichStatus RawPV Fields5 Fields6]
      (and (get RawPV "closed_sales" V)
           (= P {"key" "contact.closed_sales" "value" V})
           (fold Fields5 P append Fields6))
      (= Fields6 Fields5))

    ;; CF7: total_value
    (with-table-if (and (= EnrichStatus "enrich")
                        (get RawPV "total_value" _))
      [AgentLink EnrichStatus RawPV Fields6 Fields7]
      (and (get RawPV "total_value" V)
           (= P {"key" "contact.total_value" "value" V})
           (fold Fields6 P append Fields7))
      (= Fields7 Fields6))

    ;; ================================================================
    ;; PUT TO GHL — update contact with custom fields
    ;; ================================================================
    (with-table-if (= EnrichStatus "enrich")
      [AgentLink Ctx GhlContactId Fields7 EnrichStatus HttpStatus]
      (and (get Ctx "auth-header" PutAuthHeader)
           (= PutBody {"customFields" Fields7})
           (json-stringify PutBody PutBodyJson)
           (string-concat "https://services.leadconnectorhq.com/contacts/" GhlContactId PutUrl)
           (= PutReq {"method" "PUT"
                       "url" PutUrl
                       "headers" {"Authorization" PutAuthHeader
                                  "Version" "2021-07-28"
                                  "Content-Type" "application/json"}
                       "body" PutBodyJson})
           (http-request PutReq PutRes)
           (get PutRes "status" HttpStatus))
      (= true true))

    ;; 422 — GHL rejected the update (bad data). Drop from solution.
    (with-table-if (and (= EnrichStatus "enrich")
                        (= HttpStatus 422))
      [AgentLink EnrichStatus HttpStatus]
      (= true false)
      (= true true))

    ;; Classify: 2xx = enriched, else = error (but don't halt — just
    ;; record the status). For non-2xx non-422, the contact stays
    ;; un-enriched and will be retried on the next run.
    (with-table-if (and (= EnrichStatus "enrich")
                        (get [200 201] _ HttpStatus))
      [AgentLink EnrichStatus HttpStatus FinalStatus]
      (= FinalStatus "enriched")
      (= FinalStatus EnrichStatus))

    ;; ================================================================
    ;; WRITE enriched_at to the ledger — "enriched" rows only
    ;; ================================================================
    ;; NOTE: write-table replaces the full row, so we must include synced_at
    ;; (read from the ledger at the top of the clause) to preserve it.
    (with-table-if (= FinalStatus "enriched")
      [AgentLink AgentName Ctx GhlContactId SyncedAt FinalStatus]
      (and (get Ctx "ghl-contacts-folder-id" WriteFolderId)
           (time-now EnrichedAt)
           (= Row [AgentLink AgentName GhlContactId SyncedAt EnrichedAt])
           (fold [] Row append WriteRows)
           (= WriteInput {"file-id" WriteFolderId
                          "data" {"header" ["agent_profile_link" "agent_full_name" "ghl_contact_id" "synced_at" "enriched_at"]
                                  "rows" WriteRows}})
           (Qo.Public.OqlApi.Db.Prop/write-table WriteInput _))
      (= true true))

    ;; ================================================================
    ;; AGGREGATE + RETURN
    ;; ================================================================
    (datastore PropValsDs "omega/query-omega/data/database/property/property-values")
    (with-count LedgerTotal (PropValsDs/property-values GhlContactsFolderId _ _))

    (fold [] AgentLink append AllLinks)
    (length AllLinks TotalCount)
    (with-unique-by [AgentLink]
      (fold 0 ApiCallIncrement + ApiCount))

    (+ Skip Limit NextSkip)
    (if (or (> NextSkip LedgerTotal) (= NextSkip LedgerTotal))
      (= HasMore "false")
      (= HasMore "true"))

    ;; Log done
    (= DoneMsg ["enrich:done" TotalCount "contacts" ApiCount "api-calls" "has-more" HasMore])
    (= DoneArgs [{"page-id" EventListenerPageId "message" DoneMsg}])
    (Qo.Public.Api.Run/run-page QueryProto PrintStreamPage "run" DoneArgs _)

    (= NextArgs [{"step" "enrich" "skip" NextSkip "limit" Limit}])
    (= Result {"status" "done"
               "contacts-enriched" TotalCount
               "api-calls" ApiCount
               "has-more" HasMore
               "next-skip" NextSkip
               "next-args" NextArgs}))

(= Clauses {"on-event" {"2" OnEvent}})
(get $$ARG$$ "implementation-id" ImplId)
(Qo.Public.OqlApi.Impl/set-implementation-clauses ImplId Clauses Result)
```

- [ ] **Step 2: Push the implementation**

```bash
cd /home/ryan/Development/wttc/homes-dot-com
qo push impl/18f2e4bb-homes-com-component-library/a31e2741-events-library/<page-dir>/<impl-file>
```

- [ ] **Step 3: Commit**

```bash
cd /home/ryan/Development/wttc/homes-dot-com
git add impl/18f2e4bb-homes-com-component-library/a31e2741-events-library/<page-dir>/ impl/index.json impl/18f2e4bb-homes-com-component-library/efbde608-homes-com-data-connector/bdae504d-implementation.oql
git commit -m "feat(hdc): add Enrich GHL Contacts event handler"
```

---

## Task 5: Smoke test the enrichment step

**No new files.** Live smoke testing against a single batch.

- [ ] **Step 1: Run one batch at skip=0, limit=1**

Start with a single contact to verify the full path works:

```bash
cd /home/ryan/Development/wttc/homes-dot-com
qo run "Component Installs/Homes.Com Connector/Events/Enrich GHL Contacts" -p "Event Handler" -m "on-event" -a '[{"skip": 0, "limit": 1}]'
```

Expected: `[{"status": "done", "contacts-enriched": 1, "api-calls": 1, "has-more": "true", ...}]`.

If the response is an error or empty: use the debug-by-returning-early technique. Comment out the OQL from the PUT step down, bind `Result = [AgentLink GhlContactId EnrichStatus Fields7]`, push, re-run. This shows you:
- Whether the agent was found in the ledger (`AgentLink` bound)
- Whether the GHL contact ID is present (`GhlContactId` not "skipped")
- Whether dedup passed (`EnrichStatus` = "enrich")
- What the custom fields array looks like (`Fields7` — should be a list of `{key, value}` pairs)

If `Fields7` is empty, the raw data lookup failed. Debug further by binding `Result = [AgentLink RawPV ScoredPV]` to see if the join worked.

**If GHL rejects the `key` format in customFields:** The PUT body uses `{"key": "contact.linkedin_url", "value": "..."}`. If GHL returns an error saying the key format is invalid, switch to the `id` format:
1. Run `list-custom-fields` to get the 7 field IDs
2. Replace `"key"` with `"id"` and the field_key string with the UUID in each CF1–CF7 block
3. Store the field IDs in the push connector config for reference

- [ ] **Step 2: Verify the contact in GHL has custom fields**

Use `get-contact` to fetch the enriched contact and check that the custom fields are populated:

```bash
cd /home/ryan/Development/wttc/gohighlevel
qo run "Component Installs/GoHighLevel Connector/Queries/get-contact" -p "Query" -m "run" -a '[{"contact-id": "<ghl-contact-id-from-step-1>"}]'
```

Look for the `customFields` key in the returned contact object. The 7 fields should be present with their values from the raw/scored data.

- [ ] **Step 3: Verify enriched_at was written to the ledger**

```bash
cd /home/ryan/Development/wttc/homes-dot-com
qo query "Component Installs/Homes.Com Connector/Data/GHL Contacts" -l 1
```

The first row should now have an `enriched_at` column with a timestamp.

- [ ] **Step 4: Run a 10-row batch to verify pagination**

```bash
qo run "Component Installs/Homes.Com Connector/Events/Enrich GHL Contacts" -p "Event Handler" -m "on-event" -a '[{"skip": 0, "limit": 10}]'
```

Expected: `contacts-enriched: 10` (or 9-10 due to the already-enriched row from Step 1 being skipped), `has-more: "true"`. The first row should be skipped (already has `enriched_at`), the next 9 should be enriched.

- [ ] **Step 5: Run the 10-row batch again to verify idempotency**

```bash
qo run "Component Installs/Homes.Com Connector/Events/Enrich GHL Contacts" -p "Event Handler" -m "on-event" -a '[{"skip": 0, "limit": 10}]'
```

Expected: `contacts-enriched: 10`, `api-calls: 0` — all 10 rows should be skipped because they all have `enriched_at` set now.

---

## Task 6: Add enrichment to the coordinator

**File:** `impl/18f2e4bb-homes-com-component-library/a31e2741-events-library/c5020b33-coordinator/867913ed-implementation.oql`

- [ ] **Step 1: Read the coordinator impl**

The coordinator is at the path above. Read it and find:

1. **Line 71 — transition map:**
   ```oql
   (= Transitions {"score" "filter" "filter" "batch" "batch" "done" "test" "test-next" "test-next" "done"})
   ```
   Change `"batch" "done"` to `"batch" "enrich" "enrich" "done"`.

2. **Lines 55-59 — step-to-page-name mapping:**
   ```oql
   (when (= Step "score")     (= PageName "Score Agents"))
   (when (= Step "filter")    (= PageName "Filter Agents"))
   (when (= Step "batch")     (= PageName "GHL Sync"))
   ```
   Add: `(when (= Step "enrich")    (= PageName "Enrich GHL Contacts"))`

That's the full change — the dispatch, continuation, and advancement logic is generic and already handles any step in the transition map.

- [ ] **Step 2: Push and test**

```bash
cd /home/ryan/Development/wttc/homes-dot-com
qo push impl/18f2e4bb-homes-com-component-library/a31e2741-events-library/c5020b33-coordinator/867913ed-implementation.oql
```

Test the enrichment step via the coordinator (single batch, no advancement):

```bash
qo run "Component Installs/Homes.Com Connector/Events/Coordinator" -p "Event Handler" -m "on-event" -a '[{"step": "enrich", "skip": 0, "limit": 10, "continue": "false", "advance": "false"}]'
```

Expected: coordinator dispatches to "Enrich GHL Contacts", returns the enrichment result. Check the event listener logs for `coordinator:dispatch enrich "Enrich GHL Contacts"`.

- [ ] **Step 3: Commit**

```bash
cd /home/ryan/Development/wttc/homes-dot-com
git add impl/18f2e4bb-homes-com-component-library/a31e2741-events-library/c5020b33-coordinator/867913ed-implementation.oql
git commit -m "feat(hdc): add enrich step to coordinator transition map"
```

---

## After this plan

Once the plan is complete and the enrichment step passes smoke tests:

1. **Wait for the GHL Sync create pass to finish** (currently running, ~35k rows remaining as of 09:03 UTC 2026-04-12). The enrichment step can run on any contacts that already have `ghl_contact_id` set, but for a clean full run it's better to wait until the sync is done.

2. **Run enrichment to completion** via the coordinator:
   ```bash
   qo run "Component Installs/Homes.Com Connector/Events/Coordinator" \
     -p "Event Handler" -m "on-event" -a '[{"step": "enrich", "limit": 100}]'
   ```
   This will self-continue via the event loop until all ledger rows are enriched or skipped. Monitor via `qo logs`.

3. **Spot-check customer delivery** — pick a few random contacts in GHL and verify they have social links + sales data populated. Check both high-score agents (should have most fields) and low-score agents (may have fewer social links).

4. **Update the HDC README** to mark the enrichment step as COMPLETE and update the "Current state" section.
