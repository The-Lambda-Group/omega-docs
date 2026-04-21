# OQL Namespace RBAC — Design

> Draft. 2026-04-22.
> Sibling to [2026-04-21-cross-workspace-docs-design.md](2026-04-21-cross-workspace-docs-design.md). Covers the API-surface axis of the same control plane.

## Motivation

OmegaDB's OQL query runner currently executes any query regardless of which datastores it references. A query can name `Qo.Data.Page` directly and bypass the `Qo.Public.Api.Page` boundary. In practice this is fine when the only callers are trusted (us, internal services), but it stops being fine the moment an agent container (or an end-user) can submit arbitrary queries.

This spec adds enforcement: the query runner rejects any query that names a namespace the caller isn't entitled to. Public namespaces (`Qo.Public.*`) are world-accessible by design. Everything else is owner-only.

## Relationship to the page-RBAC spec

These are orthogonal axes of one control plane:

- **Page-RBAC** ([cross-workspace-docs spec](2026-04-21-cross-workspace-docs-design.md)) gates **content** — which pages a caller can read/write.
- **Namespace-RBAC** (this spec) gates **API surface** — which OQL clauses a caller can name.

A cross-workspace read must satisfy both: the caller must be entitled to reference `Qo.Public.Api.Page/page-by-id!` (namespace-RBAC) AND the target page's effective policy must include `other +r` (page-RBAC). Either check alone is insufficient.

## Scope

**In scope:**

- Per-namespace permission model on the OQL datastore tree
- Default-deny for non-`Qo.Public.*` namespaces
- Enforcement point in the query runner: parse query, extract datastore references, check each
- Hardcoded config in v1 (no runtime policy store, no per-caller overrides)

**Out of scope (deferred):**

- Per-workspace namespace overrides (v2+)
- Runtime policy store (v3+)
- Fine-grained per-clause permissions (v3+)
- Rate limits / quotas on public namespaces (separate concern)

## Conceptual model

Same permission tuple shape as the page-RBAC spec — `(owner, group, other) × (r, w, x)` — but attached to namespaces instead of pages. For OQL namespaces the semantics map:

- `r` — **reference**: can name this datastore in a `datastore` declaration and read its clauses.
- `x` — **execute**: can invoke clauses from this datastore. In v1 `r` and `x` are co-granted (always `r-x` together).
- `w` — **extend**: can define new clauses in this namespace. Not load-bearing for query-runtime; relevant at deploy time for source-file publication. v1 stores but does not enforce.

### Inheritance

Permissions inherit down the dotted namespace tree. A grant at `Qo.Public` applies to every descendant `Qo.Public.*` unless explicitly overridden at a sub-namespace.

Walk from the query's named namespace up to the root; use the first explicit permission entry found. Implicit default at the root is `owner-only, rwx------`.

### Enforcement point

The OmegaDB query runner already parses incoming queries to identify datastore references (it has to — they determine which clauses are loaded). Adding RBAC is one extra check at parse time:

```
for each datastore reference D in query:
  policy = effective-namespace-policy(D)
  if caller is not owner AND not (policy's other includes "r"):
    reject query with a clear error
execute query
```

Queries referencing any disallowed namespace are rejected before any clause runs. Callers see an explicit error with the namespace that was denied.

## Data model

### Namespace permission config

A single config file — JSON or EDN — shipped with the OmegaDB service. Keyed by namespace prefix, valued by the mode string.

```
{
  "Qo":                          "rwx------",   ; root default: owner-only
  "Qo.Public":                   "rwx---r-x",   ; world can reference + execute
  "Qo.Public.Api":               "inherit",     ; inherits from Qo.Public
  "Qo.Public.OqlApi":            "inherit"
}
```

v1 only needs two entries: the root (default-deny) and `Qo.Public` (world +rx). Everything else inherits.

The config lives in version control alongside the OQL source. Changes are reviewed and deployed like any other code change.

### Caller identity

The query runner already receives the caller's `AppId` on each request (via auth headers / session). That's the only subject needed in v1. "Owner" for a namespace is interpreted as "the OmegaDB instance operator" — effectively always-allowed internal callers (pipeline workers, the platform itself). External callers (agents, frontends, third parties) are "other."

In v2, per-workspace overrides would allow workspace X to be treated as "owner" for namespaces it publishes. Not in v1.

## Public API

No new Public clauses — this work is inside the OmegaDB query runner itself, below the public API layer. The externally visible change is:

- Queries that reference non-public namespaces now fail with a permission error for external callers, where they previously succeeded.
- A clear error shape: `{error: "namespace-forbidden", namespace: "Qo.Data.Page", caller: "<app-id>"}`. Clients can catch this specifically.

## CLI surface

No CLI changes in v1. Future possibilities:

- `qo namespace-perm <namespace>` — read the effective policy for a namespace. Useful for audit. Deferred.
- `qo chmod-namespace <namespace> <mode>` — edit the config from CLI. Probably never — config should stay under version control, edited via PRs, not CLI toggles.

## Migration and backward compatibility

This is **enforcement where previously there was none**, so it's breaking for any external caller that was using non-public namespaces. In practice:

- The frontend (query-omega) only references `Qo.Public.*` (by design). No changes needed.
- The CLI (qo) only references `Qo.Public.*`. No changes needed.
- The MCP server references `Qo.Public.*`. No changes needed.
- Third-party / developer code paths: any usage of internal namespaces in user code was always implicitly unsafe and should move to public API or be migrated to public API.

**Rollout approach:** ship the enforcement behind a feature flag. Run for a week in monitor-mode (log would-be-denies without rejecting) to catch any unnoticed dependency. Flip to enforce-mode after the log is clean.

## End-to-end flows

### Agent reads a page via qo

1. Agent sends `qo read <page-id>` — the CLI submits a query referencing `Qo.Public.OqlApi.Page`.
2. Query runner parses, sees only public namespaces, passes the namespace check.
3. Query runner executes; `Qo.Public.Api.Page/page-by-id!` runs, which internally touches `Qo.Data.Page` (private).
4. The `Qo.Data.Page` reference is inside a clause's body, not the top-level query. It's allowed because the clause's body was defined by trusted source, not by the caller.
5. Result returns to agent.

**Key distinction:** the namespace check applies to datastore references in the *caller's query*, not to transitively-referenced datastores inside clause bodies. Clauses encapsulate internal access.

### Malicious query attempt

1. Caller submits query `(datastore Qo.Data.Page ...) (Qo.Data.Page/delete ...) (return [])`.
2. Query runner parses, finds `Qo.Data.Page` in the query's datastore declarations.
3. Namespace check: caller is external, `Qo.Data.Page` is not under `Qo.Public.*`, rejected.
4. Error returned: `{error: "namespace-forbidden", namespace: "Qo.Data.Page"}`.
5. No clause runs; no data touched.

## Tradeoffs and open questions

- **"Internal clause transitively touches private namespace" is allowed.** Clauses in `Qo.Public.*` routinely call into `Qo.Data.*` to do their work. That's the whole point of the public/private split. The check is at the *query boundary*, not at every clause invocation. Restating: the caller declares their datastores; the check runs on their declaration; clause bodies are trusted source.

- **Config file vs runtime store.** v1 is hardcoded config. Adds no new data-layer; changes go through code review. Simpler. Downside: changing permissions requires a deploy. Acceptable because changes should be rare.

- **Granularity.** v1 is at namespace level, not clause level. A namespace with any world-readable clause is fully world-accessible. If a namespace needs mixed visibility (some clauses public, some not), split it into two sub-namespaces. Less flexible than per-clause but much simpler to reason about.

- **Error message verbosity.** Current plan: return the denied namespace and the caller's AppId. Could leak information about existence of internal namespaces. Probably fine — namespace names are not secret; they're visible in source. Don't overthink this.

- **Rate limits.** Out of scope. An authenticated caller with legit public-namespace access could still hammer the server. That's a separate throttling concern.

## Phasing

Phase 1 (this spec):

- Namespace-permission config file in the OmegaDB service
- Parse-time namespace check in the query runner
- Feature-flag gated; monitor-mode first, enforce-mode after a week
- Error shape for `namespace-forbidden`

Phase 2:

- Per-workspace overrides (runtime policy store) if use cases arise
- Audit CLI (`qo namespace-perm`)

Phase 3+:

- Per-clause granularity (if needed)
- Unified with per-page RBAC in the north-star substrate (see [architecture-pages-as-datastores.md](../../../omega-knowledge-base/projects/architecture-pages-as-datastores.md))

## Testing

- Query referencing only `Qo.Public.*` from external caller: succeeds.
- Query referencing `Qo.Data.*` from external caller: rejected with clear error.
- Query referencing `Qo.Data.*` from internal caller (owner): succeeds (v1 treats only platform-internal as owner).
- Internal clause body that transitively touches `Qo.Data.*` when invoked via a `Qo.Public.*` entry point: succeeds.
- Feature flag in monitor-mode logs would-be-denials without rejecting; enforce-mode rejects.
- Error shape stable and parseable by consumers.
