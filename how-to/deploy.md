[< Home](../README.md) | [How-to](README.md)

# Deploy an implementation fix

This doc covers how to push an OQL implementation fix and make it live in a running installed component. The two steps are separate: `qo push` registers clauses on the library-side (dev) implementation; deploying to an installed component requires a separate targeted step.

## Background: library-side vs installed-side

Every component exists in two places in the workspace:

- **Library-side (dev) component** — the component in your developer library. `qo push [impl-file]` updates this one. The impl file's embedded `set-implementation-clauses` script calls the engine with `$$ARG$$` as the impl-id; at push time `$$ARG$$` resolves to the library-side impl-id.
- **Installed component** — a separate copy of the component installed into a specific workspace location (e.g. a MAB Agent System page). It has its own component-id and its own impl-id. `qo push` does **not** touch the installed component.

If you push a fix and the running workflow continues to exhibit the old behavior, you have fixed the library side only. The installed component is still running the old clauses.

## How to identify which impl-id the installed component uses

Run a scratch query to look up the installed component's impl-id. You need the page-path where the component is installed:

```
(datastore D "omega://default")
(Qo.Public.OqlApi.Page/page-by-name D "Workflows/Design/Copywriter" P)
(Qo.Public.OqlApi.Page.Comp/page-component D P Pc)
(Qo.Public.OqlApi.Comp/component D (get-in Pc ["page-component-data" "componentId"]) C)
(= ImplId (get-in C ["component-data" "implementationId"]))
(return [ImplId])
```

Replace `"Workflows/Design/Copywriter"` with the actual installed page path.

The returned `ImplId` is the impl-id you must target to deploy the fix to the installed component.

## Deploy a fix to the installed component

Once you have the installed impl-id, run the `set-implementation-clauses` query directly, passing the installed impl-id as the argument. Use `omega-cli run-query`:

```
omega-cli run-query --impl-id <installed-impl-id> \
  --file impl/<component-prefix>/<impl-prefix>-implementation.oql
```

Or via `qo run` if your project is configured for it:

```
qo run "System/Queries/Set Implementation Clauses" \
  -p Query -m run -a '[{"impl-id": "<installed-impl-id>"}]'
```

The exact invocation depends on your project's tooling. The key point: you must explicitly pass the **installed** impl-id, not the library-side impl-id.

## Concrete example (MAB Copywriter triage, 2026-05-06)

- Library-side component-id: `1adb9940`; `qo push` updated this impl.
- Installed Copywriter component-id: `0ee1c2af`; impl-id: `8f9735cf`.
- Fix was not live until a separate `omega-cli run-query` call targeted impl-id `8f9735cf`.

## When to use each approach

| Goal | Tool |
|---|---|
| Register updated clauses on the library-side impl (iterating in dev) | `qo push [impl-file]` |
| Deploy the fix to a running installed component | `omega-cli run-query` targeting the installed impl-id |
| Both at once | Not supported — two separate operations |

## See also

- [find-existing-ids.md](../../query-omega-cli/docs/how-to/find-existing-ids.md) — how to recover UUIDs (component-id, impl-id) from a live page registration.
- [get-component-for-page.md](../../query-omega-cli/docs/how-to/get-component-for-page.md) — scratch-query recipe for finding which component is attached to a page.
