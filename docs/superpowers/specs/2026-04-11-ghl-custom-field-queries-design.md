# GHL Custom Field Queries — Design Spec

**Date:** 2026-04-11 (UTC)
**Status:** Design, pre-implementation
**Scope:** Add three new queries to the GoHighLevel Connector for managing GHL location custom fields.

## Purpose

Unblock the HDC enrichment step (see `projects/homes-dot-com/explanation/enrich-ghl-contacts-design.md` in the knowledge base). The enrichment step needs to resolve custom field names to GHL field IDs at batch time, and it needs the seven HDC custom fields (`linkedin_url`, `facebook_url`, `twitter_url`, `pinterest_url`, `score`, `closed_sales`, `total_value`) to exist in the location before the first batch runs.

These queries let HDC — and any future consumer — list, create, and delete custom fields programmatically instead of requiring manual setup in the GHL UI.

## Scope

**In scope (three queries):**

1. `list-custom-fields` — GET all custom fields for the configured location
2. `create-custom-field` — POST a new custom field on the configured location
3. `delete-custom-field` — DELETE a custom field by its GHL ID

**Out of scope:**

- `get-custom-field` (by ID or by name) — redundant with list + in-memory filter for the only known consumer (HDC enrichment, which needs all seven at once)
- `update-custom-field` — no known consumer; admin rename is a rare path
- `ensure-custom-field` — callers can compose from list + create if they want idempotent provisioning
- Refactoring the existing `list-contacts` / `create-contact` / `delete-contacts` queries to share code with the new queries — deferred to a separate session
- Rate-limit (429) handling — no consumer exercises the retry path
- Duplicate-detection (400 with `meta.contactId`) — contact-specific, doesn't apply to custom fields
- Safety gates on delete (dry-run, confirm flag) — callers manage their own discipline

## Design decisions

**These queries are throwaway for now.** HDC is the only known consumer and is a one-shot project. The goal is the thinnest useful wrapper over the GHL API, matching the style of the existing `get-contact` query.

### Response shapes

Each query returns one of three shapes:

```
{"status" "ok" ...data}
{"status" "not-found"}
{"status" "error" "code" <http-status> "body" "<raw body>"}
```

This matches `get-contact`. Callers branch on `status` without seeing HTTP details on the happy paths.

### Arg shapes

**`list-custom-fields`** takes `{}`. No args. Returns `{status: "ok", fields: [...]}` on 2xx.

**`create-custom-field`** takes a hybrid of explicit required fields plus an optional passthrough map:

```
{"name" "<human-readable name>"
 "data-type" "TEXT"
 "model" "contact"
 "extra" {}}            ; optional, camelCase keys merged into the POST body
```

- `name`, `data-type`, `model` are required. The query fails loudly if any is missing.
- `extra` is an optional map of any additional GHL fields (`placeholder`, `position`, `acceptedFormat`, etc.) with camelCase keys. The query merges it into the POST body untouched.
- The query translates `data-type` → `dataType` internally (kebab-case is OQL idiomatic; camelCase is GHL wire format).
- Returns `{status: "ok", field: {...}}` on 2xx.

**`delete-custom-field`** takes `{field-id}`. No safety gate. Returns `{status: "ok"}` on 2xx, `{status: "not-found"}` on 404, `{status: "error", ...}` otherwise.

### No shared `ghl-request` helper

Brainstorm considered building a private `ghl-request` helper impl that all three queries would delegate to for auth header injection, `http-request` invocation, and status-code dispatch. **Dropped** after reading the existing `get-contact` impl, which inlines the full pattern in ~60 lines and reads cleanly.

Reason: OQL has no clean cross-impl-file helper mechanism. The alternatives are:
- A helper page invoked via `run-page` from each of the three queries — adds a cross-page protocol call per invocation, which is real runtime overhead and obscures the code path
- Include-style sharing — not available in OQL

Three queries × ~50 lines of inlined pattern is cleaner than a helper page with the ceremony of protocol dispatch. Copy-paste matches how `get-contact` is already written, so the new code will feel consistent with the existing codebase.

If a second consumer ever needs these queries, or if we start building more GHL query surface area, the refactor can extract `ghl-request` at that time.

### HTTP primitive

All three queries use `http-request` (not the deprecated `http-get-request` / `http-post-request` variants). This is non-negotiable — the deprecated variants don't capture status codes, which is the exact bug pattern that halted the HDC pipeline at skip=94200.

### Cloudflare header gotcha

GETs must use `Accept: application/json` for content negotiation — **not** `Content-Type: application/json`. Cloudflare's WAF (sitting in front of GHL) rejects GETs carrying a `Content-Type` header with a 400 Bad Request HTML page before the request reaches GHL. This is already documented inline in `get-contact.oql` and will be mirrored in `list-custom-fields`.

POST and DELETE can safely set `Content-Type: application/json` because those methods have bodies (or are expected to).

## Architecture

```
GoHighLevel Connector
  Queries/
    get-contact           (existing)
    list-contacts         (existing, not touched)
    create-contact        (existing, not touched)
    delete-contacts       (existing, not touched)
    list-custom-fields    (NEW)
    create-custom-field   (NEW)
    delete-custom-field   (NEW)
```

Each new query is its own page under `Queries/`, with its own component, Query protocol, and implementation file. No shared infrastructure.

### Parent chain walking

Each query walks from its own page up to the GoHighLevel Connector install page to read the push connector config (same pattern as `get-contact`):

```
MyPage → Queries folder → Connector install page → push connector: "GoHighLevel" → {api-key, location-id}
```

This gives each query `ApiKey` and `LocationId` to build auth headers and URLs.

### Endpoints

Endpoints (from the GHL API v2 marketplace docs):

- **List:** `GET https://services.leadconnectorhq.com/locations/{locationId}/customFields`
- **Create:** `POST https://services.leadconnectorhq.com/locations/{locationId}/customFields`
- **Delete:** `DELETE https://services.leadconnectorhq.com/locations/{locationId}/customFields/{fieldId}` *(by convention — verified empirically at implementation time)*

Standard headers on all requests:
- `Authorization: Bearer <api-key>`
- `Version: 2021-07-28`
- `Accept: application/json` (GET)
- `Content-Type: application/json` (POST, DELETE with body)

### Request body translation

`create-custom-field` builds the POST body by:
1. Starting with `{name: Name, dataType: DataType, model: Model}`
2. Merging `extra` on top (callers can override, though they shouldn't)
3. JSON-stringifying and sending

The exact shape of `dataType` values (e.g. `TEXT`, `LARGE_TEXT`, `NUMERICAL`, `DATE`) will be confirmed empirically during the first smoke test. HDC only needs `TEXT`.

### Response parsing

Each query uses flat sequential `with-table-if` dispatch, matching `get-contact`:

1. HTTP status 200/201 → bind a success symbol from the parsed body
2. HTTP status 404 → bind a not-found flag
3. Dispatch on which symbol was bound to set the `Result` map
4. Fallback: if `Result` is still unbound, emit an error shape

**No nested `with-table-if`s.** Flat sequential dispatch is a hard rule from the HDC post-mortem (`projects/homes-dot-com/gotchas/ghl-sync-halt-skip-94200.md`).

## Testing

**Live smoke tests against the configured GHL location.** No mocking, no dry-runs. Each query is tested by running it:

1. **`list-custom-fields`** — run against the current location; verify it returns a list and that the response shape matches the `{status: "ok", fields: [...]}` contract. Also use this to discover the exact field schema GHL returns.
2. **`create-custom-field`** — create a throwaway test field named `test_delete_me_<timestamp>` with `data-type: "TEXT"` and `model: "contact"`. Verify the response is `{status: "ok", field: {...}}` with an ID.
3. **`delete-custom-field`** — delete the throwaway field created in step 2 by its ID. Verify the response is `{status: "ok"}`.
4. **End-to-end sanity** — re-run `list-custom-fields` after deletion; verify the test field no longer appears.

Once those four steps pass, use `create-custom-field` to provision the seven HDC fields (`linkedin_url`, `facebook_url`, `twitter_url`, `pinterest_url`, `score`, `closed_sales`, `total_value`) for the enrichment step. This is the first real consumer and closes out the customer-field prerequisite for HDC.

### Error cases to observe

These aren't gated tests but are worth watching for during the smoke tests so the error branch can be tuned:

- What does GHL return for `create-custom-field` when a field with the same name already exists? (400 with some body shape — our query will surface it as `{status: "error", ...}`, but we should capture the shape in case we later build `ensure-custom-field`.)
- What does GHL return for `delete-custom-field` when the field ID doesn't exist? (404 → should map to `{status: "not-found"}`. If it's instead a 400 or a 200 with an error in the body, the dispatch needs a different branch.)

## Out-of-scope follow-ups

These are explicitly deferred and should NOT be mixed into this work:

- Refactoring the existing contact queries to share auth/header/dispatch code with the new queries
- Migrating `list-contacts` / `create-contact` / `delete-contacts` off the deprecated `http-get-request` / `http-post-request` primitives
- Building `ghl-request` as a real shared helper
- Building `update-custom-field`, `ensure-custom-field`, `get-custom-field`
- Adding rate-limit or duplicate-detection handling anywhere

All of these are listed in the GoHighLevel Connector Query Library Roadmap in the repo README and will be picked up in a future session once HDC is delivered.
