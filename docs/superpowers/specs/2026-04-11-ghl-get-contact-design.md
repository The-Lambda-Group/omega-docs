# GHL `get-contact` query — design

**Status:** design, not yet implemented
**Date:** 2026-04-11
**Scope:** one new query in the GoHighLevel Connector's Query Library

## Why this exists

The HDC GHL Sync halted at ~`skip=94200` on an HTTP 400 "duplicated contacts" response from GoHighLevel. The 400 body named an existing contact id (`slKKNisqFtVzT3Tiwd4x`) that blocked our create. The HDC README's "Known issues" section proposes a small fix ("just tolerate 400s") but that guidance assumes the dupe is benign — a real-world email collision with a foreign contact. **We have not verified that assumption.** The dupe could be one of several scenarios, and each implies a different fix:

1. **Ledger drift from our own earlier run.** GHL has our contact (`source: "homes.com pipeline"`, `homes-dot-com` tag) but our local `Data/GHL Contacts` ledger does not — because a prior run crashed mid-batch between the GHL POST and the ledger write. Fix: tolerate the 400, backfill the existing contact id from `meta.contactId` into the ledger, and consider a broader reconciliation.
2. **Foreign pre-existing contact.** Different `source`, no `homes-dot-com` tag, older `dateAdded`. Came from another WTTC integration, manual import, or external sync. Fix: tolerate-and-log but do NOT claim it in our ledger.
3. **Internal dupe in source data.** Two rows in `Data/Filtered Agents` share an email but have different `agent_profile_link` values. Our row-index dedup (keyed on `agent_profile_link`) lets both through; the second POST hits 400. Fix: tolerate-and-log is insufficient — we need email-based dedup on our side before the GHL call, which is a meaningfully larger fix.
4. **Something else.** TBD based on what we observe.

Without the ability to fetch the existing GHL contact by id, we cannot distinguish these scenarios, and we risk picking the wrong fix. The `get-contact` query is the smallest tool that lets us answer the question "who is `slKKNisqFtVzT3Tiwd4x`" and therefore decide what the 400-dupe fix should actually do.

It is also a P0 primitive on the GoHighLevel Connector's Query Library Roadmap (see `gohighlevel/README.md`). Building it now satisfies both the immediate HDC investigation need and a planned SDK milestone.

## Non-goals

This spec deliberately excludes several adjacent changes. They are valid follow-ups, but bundling them expands scope and delays the investigation this work unblocks.

- **The HDC 400-dupe fix itself.** The fix is a follow-up to this work, after the investigation tells us which scenario we're in.
- **`get-contact-by-email`** (roadmapped as P1). Not needed to identify contact `slKKNisqFtVzT3Tiwd4x`.
- **The `ghl-request` shared helper.** Step 2 of the connector's migration path explicitly sequences the helper extraction after the query library is built out. Deviating from that ordering is scope creep.
- **Migrating `list-contacts` / `create-contact` / `delete-contacts` to the normalized response shape.** The existing three queries return a legacy `{status: "ok", response: Parsed}` shape. They will be migrated when the shared helper is introduced. Interim inconsistency is accepted.
- **Any changes to the `ghl-sync` impl.** This work does not touch the sync.

## Architecture

### File layout

New files (three):

```
wttc/gohighlevel/impl/095bb6c8-gohighlevel-connector/755835ce-queries/
  <new-uuid>-get-contact/
    page.json                                   # page metadata
    <component-uuid>-get-contact.component.json # component definition
    <impl-uuid>-implementation.oql              # the clause
```

Modified files (two):

1. `wttc/gohighlevel/impl/095bb6c8-gohighlevel-connector/429cf5d4-implementation.oql` — the connector's install impl gains a fourth query-provisioning block (copy-paste the existing `add-or-get-sub-page-by-name` → `set-page-component` pattern used for `list-contacts`, `create-contact`, `delete-contacts`).
2. `wttc/gohighlevel/README.md` — adds a new `### get-contact` subsection under "Queries", adds `get-contact` to the "Component IDs" table, and marks the `get-contact` row in the Query Library Roadmap as built.

No other files change.

### Consumer-facing interface

Invoked from the `qo` CLI or from another OQL clause via `run-page`:

```bash
qo run "Component Installs/GoHighLevel Connector/Queries/get-contact" \
  -p "Query" -m "run" -a '[{"contact-id": "slKKNisqFtVzT3Tiwd4x"}]'
```

**Arguments:**

| Key | Required | Description |
|-----|----------|-------------|
| `contact-id` | Yes | GoHighLevel contact id (opaque string). |

**Responses** (one of three shapes, matching the "Cross-cutting concerns" section of the GHL README):

```json
{"status": "ok", "contact": { /* full GHL contact object as returned by GHL */ }}
{"status": "not-found"}
{"status": "error", "code": <http-status>, "body": "<raw response body>"}
```

The `{status: "duplicate", ...}` and `{status: "rate-limited", ...}` shapes from the roadmap do not apply to a GET and are not emitted by this query.

### Clause structure

The impl walks up to the connector page and reads credentials exactly like `create-contact` does (the same connector, the same page-tree walk), but the HTTP call uses `http-request` — the only supported HTTP primitive per `omegadb/language/built-ins.md`. The deprecated `http-get-request` / `http-post-request` forms are strictly forbidden: they do not capture HTTP error codes, which is exactly what this clause needs to branch on.

Per the built-ins reference:
- `http-request` takes a request map `{method, url, headers, query-params?, body?}` where `method` is lowercase (`"get"`, `"post"`, etc.)
- The response map has `status` (HTTP code, as a number), `body` (response body as a string), and `headers` (map)

After the call, the clause uses nested `with-table-if` per-solution branching (documented in `omegadb/language/control-flow.md`) to produce one of the three normalized response shapes. `with-table-if` is binary, so three-way branching requires one nesting level: outer tests for 200, else-branch contains an inner `with-table-if` testing for 404, inner else-branch is the general error path.

**Header-var discipline** (critical for `with-table-if`): each branch is its own lexical scope, and any outer-scope variable that either branch reads or writes must appear in the header. Variables defined *inside* a branch via `(and ...)` are local and do not need to be in the header. Both branches of the outer `with-table-if` reference `HttpStatus`, `ResBody`, and `Result`, so all three must appear in the outer header. The inner `with-table-if` has the same set.

Pseudo-OQL outline (final syntax verified at implementation time, but this structure is expected to land effectively unchanged):

```
(:- (Run Exec Result)
  (get Exec "args" Args)
  (get Args 0 Data)
  (get Data "contact-id" ContactId)

  ;; Walk up to the GHL Connector page (same pattern as create-contact)
  ;; Read api-key from the "GoHighLevel" push connector on the connector page
  ;; -- page-tree walk omitted for brevity; see create-contact/implementation.oql

  ;; Build GET /contacts/{id}
  (string-concat "https://services.leadconnectorhq.com/contacts/" ContactId Url)
  (string-concat "Bearer " ApiKey AuthVal)
  (= Req {"method" "get"
           "url" Url
           "headers" {"Authorization" AuthVal
                       "Version" "2021-07-28"
                       "Content-Type" "application/json"}})
  (http-request Req Res)
  (get Res "status" HttpStatus)
  (get Res "body" ResBody)

  ;; Three-way branch: 200 / 404 / error, via nested with-table-if.
  ;; Header vars: HttpStatus + ResBody (read in else branches), Result (produced).
  (with-table-if (= HttpStatus 200)
    [HttpStatus ResBody Result]
    ;; 200: parse body, extract contact, emit ok shape. Parsed/Contact are
    ;; local to this branch and do not need to be in the header.
    (and (json-stringify Parsed ResBody)
         (get Parsed "contact" Contact)
         (= Result {"status" "ok" "contact" Contact}))
    ;; else: check 404 vs error
    (with-table-if (= HttpStatus 404)
      [HttpStatus ResBody Result]
      (= Result {"status" "not-found"})
      (= Result {"status" "error" "code" HttpStatus "body" ResBody}))))
```

Note on `json-stringify`: despite the name, this OQL built-in is bidirectional — when the output position is a map/array and the input is a string, it parses the string (equivalent to `JSON.parse`). This matches the existing pattern in `create-contact`/`list-contacts`.

One implementation-phase verification: GHL's actual 404 response body shape. If GHL returns a body that still parses to a map with a `"contact"` key (unlikely but worth a one-line test with a garbage contact id), the 200 branch's `(get Parsed "contact" Contact)` would incorrectly succeed on a 404 — but we already branch on `HttpStatus` *before* parsing, so this is not a concern. Noted only for completeness.

### Install impl update

The install impl at `429cf5d4-implementation.oql` currently provisions three query pages. Adding a fourth is mechanical:

```
;; get-contact query
(Qo.Public.OqlApi.Page/add-or-get-sub-page-by-name
  QueriesFolder "get-contact" GetContactPage)
(= GetContactComp {"app-id" "5b4edd7a-f920-450b-8732-399095154c83"
                    "component-id" "<new-component-uuid>"})
(Qo.Public.OqlApi.Page.Comp/set-page-component GetContactPage GetContactComp _)
```

The `installed` result's `queries` array gains `"get-contact"`.

### Data flow

```
qo run (or run-page from another clause)
  → Query/run method on Queries/get-contact page
  → impl walks up to connector page
  → reads api-key + location-id from "GoHighLevel" push connector
  → GET https://services.leadconnectorhq.com/contacts/{contact-id}
  → branches on HTTP status to normalized response shape
  → returns to caller
```

No writes. No mirror table. No ledger interaction. Read-only against GHL.

## Error handling

The normalized response shapes give callers three clean branches:

- `"ok"` — the contact exists, the caller can read `contact.source`, `contact.tags`, `contact.dateAdded`, `contact.customFields`, etc.
- `"not-found"` — GHL returned 404 for this id. The caller can handle this without seeing HTTP details.
- `"error"` — anything else (500, 401, timeout, malformed body). The caller decides whether to retry, log, or surface. The raw HTTP status and body are included so debugging is possible.

The clause itself does not log, retry, or throw. It produces a deterministic response and returns. Any retry / backoff / logging discipline lives in the caller.

One intentional limitation: the clause does not currently distinguish 401 (bad credentials) from 500 (server error) — both are `"error"` with the HTTP code in the payload. If a caller needs finer discrimination, that's a future refinement, not a blocker for this work.

## Testing

One end-to-end smoke test post-deploy, driven manually:

1. Deploy the new impl to the dev DB.
2. Re-run the connector's `install` method to provision the new `get-contact` query page.
3. `qo run` the new query with contact id `slKKNisqFtVzT3Tiwd4x`. Expected: `{status: "ok", contact: {...}}` with a real GHL contact body. **This is the moment the investigation actually happens** — the body tells us which dupe scenario we're in.
4. `qo run` with a garbage contact id (e.g. `"nonsense-id"`). Expected: `{status: "not-found"}`.
5. (Optional) Temporarily corrupt the push connector's `api-key` to something invalid and verify the query returns `{status: "error", code: 401, body: ...}`. Revert the key afterward.

No automated test suite for OQL components currently exists in this repo; this matches the testing discipline of the existing three queries.

## Follow-up (NOT IN SCOPE here)

Once the investigation from step 3 tells us which dupe scenario we're in, the HDC 400-dupe fix becomes the next piece of work. That work lives in `wttc/homes-dot-com/impl/.../bfe95f5a-implementation.oql`, not in the gohighlevel connector, and is tracked separately. It may or may not benefit from a follow-up SDK primitive depending on which scenario we're in (e.g. scenario 3 — internal email dupes — would probably want `get-contact-by-email` to do pre-flight dedup on our side).

**Latent bug discovered while writing this spec (separate work item):** the existing three queries in this SDK (`list-contacts`, `create-contact`, `delete-contacts`) all use `http-get-request` / `http-post-request`, which are deprecated and do not capture HTTP error codes. They will silently fail to branch on non-2xx responses — the same class of bug that caused the HDC GHL Sync halt. Migrating them to `http-request` is not part of this spec's scope, but it should be done as its own focused piece of work before the connector is exercised in any new production path. Listing here so it does not get lost.

## Open questions

None. Every design-level question resolved to a doc-aligned default: the `gohighlevel/README.md` Query Library Roadmap prescribes the query name, scope, and response shape; the `homes-dot-com/README.md` "Known issues" section prescribes the motivation; `omegadb/language/built-ins.md` fully specifies `http-request`'s request and response maps (so there is no unknown about the `Res` shape); and `omegadb/language/control-flow.md` specifies how to use `with-table-if` with the correct header-var discipline for nested branching. The pseudo-OQL in this spec is close enough to drop-in ready that implementation is mostly mechanical.
