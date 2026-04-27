[< Home](../README.md) | [Gotchas](README.md)

# `page-by-name` Path Lookups Force a Full Scan

## Symptom

A scratch query (or runtime impl) tries to resolve a page by its human-readable path:

```oql
(Qo.Public.OqlApi.Page/page-by-name "Component Installs/MAB Agent System/Brief" BriefPage)
```

and crashes with a 500:

```
{"type":"omega/query/full-scan",
 "data":{"msg":"Query routed to a Mango _find scan, which is disabled.
                Restructure the call site so its bound args match an existing
                primary-key or write-index..."}}
Stack trace:
failed at:
((Qo.Public.OqlApi.Page/page-by-name "Component Installs/MAB Agent System/Brief" BriefPage))
```

The error message links to [`query-omega-oql/docs/how-to/triage-an-oql-full-scan.md`](https://github.com/The-Lambda-Group/query-omega-oql/blob/master/docs/how-to/triage-an-oql-full-scan.md), but the triage doc is a general full-scan playbook — it does not list `page-by-name` as a known trigger, so the agent burns minutes diffing pkeys before realising the API itself was the problem.

Observed during MAB Strategist V1 build (2026-04-28) in a scratch query under `omega-cli run-query`. Full-scans are blocked at the engine level by policy (see [`no-mango-by-default`](https://github.com/The-Lambda-Group/omega-db/blob/master/docs/explanation/no-mango-by-default.md)), so the throw is intentional.

## Why

There is **no `Qo.Public.OqlApi.Page/page-by-name`** in the public API. The closest real APIs in [`query-omega-api/oql/app/public/oql-api/page.oql`](https://github.com/The-Lambda-Group/query-omega-api/blob/master/oql/app/public/oql-api/page.oql) are:

| Clause | Indexed? | What it takes |
|--------|----------|---------------|
| `Qo.Public.OqlApi.Page/page-by-id` | Yes (pkey) | `PageId` (UUID string) |
| `Qo.Public.OqlApi.Page/pages-by-folder` | Yes (write-index) | `FolderId` (UUID string) |
| `Qo.Public.OqlApi.Page/parent-page` | Yes | `Page` object → `ParentPage` |
| `Qo.Public.OqlApi.Page/child-page-by-name` | Yes | `ParentPage`, `Name` → `ChildPage` |
| `Qo.Public.OqlApi.Page/child-by-name` | Yes | `ParentPageId`, `Name` → `ChildPage` |
| `Qo.Public.OqlApi.Page/page-search` | **No — Mangoes** | search-string `Q` |
| `Qo.Public.OqlApi.Page/page-query` | **No — Mangoes** | query map `Q` |

A path-style lookup (`"A/B/C"` → page) doesn't match any indexed primary key or write-index — there is no `path` field stored on the page row. The engine has no choice but to fall back to a Mango `_find` selector, which the engine policy refuses.

The same shape applies to `child-page-path` — it doesn't exist either, for the same reason. See [`child-page-path-does-not-exist`](https://github.com/The-Lambda-Group/query-omega-api/blob/master/docs/reference/child-page-path-does-not-exist.md) for the canonical write-up of the "path-style multi-level navigation does not exist" rule.

`page-by-name` is a natural-feeling API name that an agent will reach for first: "give me the page at this path." The fact that no such clause exists, and that even if you wrote one it would always full-scan, is non-obvious from the API name. Treat it as a class of failure, not a one-off bug.

## Fix — three workarounds

Pick the one that matches your call site's context.

### 1. Parent-walk + `child-page-by-name` (preferred at runtime)

If you are inside a runtime impl (e.g. an `OnEvent` dispatch where you already have `Exec.method.page-id` → `SelfPage`), walk the page tree with indexed calls only. Both `parent-page` and `child-page-by-name` hit the pkey path.

```oql
;; Start from a known page (e.g. SelfPage from Exec.method.page-id), walk to a
;; known root, then descend by name. Every step is index-served.
(Qo.Public.OqlApi.Page/parent-page SelfPage P1)
(Qo.Public.OqlApi.Page/parent-page P1 Root)
(Qo.Public.OqlApi.Page/child-page-by-name Root "Component Installs" InstallsPage)
(Qo.Public.OqlApi.Page/child-page-by-name InstallsPage "MAB Agent System" SystemPage)
(Qo.Public.OqlApi.Page/child-page-by-name SystemPage "Brief" BriefPage)
```

This is the default for runtime queries. Chain one `child-page-by-name` per level — there is no multi-level helper (see [`child-page-path-does-not-exist`](https://github.com/The-Lambda-Group/query-omega-api/blob/master/docs/reference/child-page-path-does-not-exist.md)).

### 2. Hardcode known folder-ids in scratch queries

For a scratch query that is just probing data, paste the known folder-id directly. Folder-ids are stable across re-runs (until a `qo delete-page` + reinstall regenerates them), so this works for short-lived diagnostic queries.

```oql
;; Scratch query — hardcode the folder-id rather than resolving via path.
(= StrategiesDbId "4daf5e24-e906-4ccc-ac99-d62cf04a0f25")
(Qo.Public.OqlApi.Page/page-by-id StrategiesDbId StrategiesPage)
```

`qo describe <page>` prints the folder-id under the "Folder ID" label (which is actually the page's own `page-id` — see [`page-id-vs-folder-id`](https://github.com/The-Lambda-Group/query-omega-api/blob/master/docs/reference/page-id-vs-folder-id.md)). Copy that string. Do **not** ship hardcoded ids in stored impls — they are environment-specific.

### 3. Resolve once at install time, stash the page-id

Install impls run with full scan enabled, so they may use `page-search`, `page-query`, or any other Mango-routed API to resolve a page by name during boot. Stash the resolved page-id as component-install metadata; subsequent runtime queries look it up by id (path 1 or path 2 above).

```oql
;; Inside an install impl — full-scan is permitted here.
;; Resolve once, stash the id, then read back by id at runtime.
(Qo.Public.OqlApi.Page/page-search {"app-id" AppId
                                    "search-string" "Brief"
                                    "limit" 1} BriefPage)
(get BriefPage "page-id" BriefPageId)
;; Persist BriefPageId on the component-install record for later runtime lookup.
```

This is the right pattern when the path is fixed at install time (e.g. a component declares "I always live under `Component Installs/<MyName>/Brief`") and runtime queries should never re-resolve.

## Diagnostic signs you are about to hit this

- Your query string includes `"/"` separators inside a single page-resolution call.
- The clause name in your draft contains `by-name` and takes more than one bound argument before the `Page` output.
- You're tempted to write a helper that "just looks up the page at this path" — there is no indexed shape that does this.
- The error type is `omega/query/full-scan` and the failed term is one of `page-by-name`, `page-search`, `page-query`, or any custom helper you wrote whose body wraps one of those.

In all cases: switch to one of the three fixes above before reaching for `(enable-full-scan)`. A bookend masks the symptom and leaks through scope (see [`triage-an-oql-full-scan` § Anti-pattern](https://github.com/The-Lambda-Group/query-omega-oql/blob/master/docs/how-to/triage-an-oql-full-scan.md#anti-pattern----bookending-a-fixable-miss)).

## Related

- [`triage-an-oql-full-scan`](https://github.com/The-Lambda-Group/query-omega-oql/blob/master/docs/how-to/triage-an-oql-full-scan.md) — General playbook for diagnosing `omega/query/full-scan` throws. This doc is the `page-by-name`-specific specialisation: same engine policy, but the fix is "use a different API," not "restructure the bound-arg set."
- [`child-page-path-does-not-exist`](https://github.com/The-Lambda-Group/query-omega-api/blob/master/docs/reference/child-page-path-does-not-exist.md) — The sister gotcha: `child-page-path` is also not a real clause; chain `child-page-by-name` per level. Same root cause (no path-style indexed shape exists), same fix shape.
- [`page-id-vs-folder-id`](https://github.com/The-Lambda-Group/query-omega-api/blob/master/docs/reference/page-id-vs-folder-id.md) — Disambiguation for the two ID fields on a Page object. Relevant for workaround 2 (hardcoding ids) and for understanding `qo describe` output.
- [`no-mango-by-default`](https://github.com/The-Lambda-Group/omega-db/blob/master/docs/explanation/no-mango-by-default.md) — Why `omega/query/full-scan` exists at all and why install-time queries can opt in.
- [datastores.md](../reference/datastores.md) — Datastore, functor, and defterm fundamentals; the index-matching rule that this gotcha is a special case of.
