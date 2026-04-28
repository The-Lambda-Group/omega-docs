[< Home](../README.md) | [Gotchas](README.md)

# `omega-cli` Defaults to `localhost:3001` While `qo` Reads `OMEGA_URL` from `.env`

## Symptom

You run a scratch query against what you assume is the production database:

```bash
omega-cli run-query path/to/scratch.oql
```

and the query returns no results, or no-return on a `prop-vals-by-folder` / `pages-by-folder` / `row-index` lookup that you know has data in production. No error about hosts, no connection failure — just a clean zero-solutions envelope.

The same OQL hand-pasted into `qo run` (or `qo query`) against the same workspace returns the expected rows.

Observed during MAB Copywriter V1 implementation (2026-04-28) on a Phase 3 smoke test of a Strategies-table read. The query was correct; the data was in production; the local DB on `http://localhost:3001` was empty, so the query truthfully returned no rows.

## Why

`omega-cli` and `qo` are two different CLIs with different host-resolution defaults:

| Tool | Default host | Override |
|---|---|---|
| `omega-cli run-query` | `http://localhost:3001` (compiled-in default) | `-h <url>` flag, or `OMEGA_URL` env var exported in the shell that launches `omega-cli` |
| `qo` (commands like `run`, `query`, `push`, `ls`) | Reads `OMEGA_URL` from the project's `.env` file | `.env` value, or shell-exported `OMEGA_URL` (the latter wins — see [env-loading](https://github.com/The-Lambda-Group/query-omega-cli/blob/master/docs/reference/env-loading.md)) |

The mismatch is silent. `omega-cli run-query` happily POSTs the query to `http://localhost:3001/query_clj`. If a local OmegaDB is running there (developer machine), it answers — but with whatever data is in the **local** DB, which is typically empty or has stale fixtures. The query envelope is identical to a successful production query, just with zero solutions because no rows match.

The agent then triages "no-return" looking for a body-level zero-solutions cause (per [triage-no-return § Body-level](../how-to/triage-no-return.md#body-level-no-result-bubbling-to-the-dispatch-boundary)), bisects the body, and finds nothing wrong with the query — because nothing **is** wrong with the query. The query is asking the wrong DB.

`qo` does not have this footgun on the same workstation because the project's `.env` pins `OMEGA_URL=https://db.queryomega.com`. `omega-cli` ignores the project `.env` — it has no dotenv loading layer of its own.

## Fix — three ways, pick one

### 1. Pass `--host` (or `-h`) explicitly on each invocation (preferred for ad-hoc queries)

```bash
omega-cli run-query --host https://db.queryomega.com path/to/scratch.oql
omega-cli run-query -h https://db.queryomega.com path/to/scratch.oql
```

This is what the canonical OQL development docs already show — see [develop-oql.md § The push-run cycle](../how-to/develop-oql.md#the-push-run-cycle), which uses `omega-cli run-query -h $OMEGA_URL ...`. The `$OMEGA_URL` there expands from the shell, not from `.env`.

### 2. Export `OMEGA_URL` in the shell that runs `omega-cli`

```bash
export OMEGA_URL=https://db.queryomega.com
omega-cli run-query path/to/scratch.oql
```

Persist this in your shell rc if every session targets the same host. If you frequently switch between hosts (local dev DB vs production), prefer option 1 to make the choice explicit per command.

### 3. Source the project `.env` before invoking `omega-cli`

```bash
set -a; source .env; set +a
omega-cli run-query path/to/scratch.oql
```

This makes `omega-cli` honour the same `OMEGA_URL` the project's `qo` uses. Convenient when you're already in a project directory and want both CLIs pointed at the same host.

## Diagnostic signs you are about to hit this

- A scratch query returns no rows from a public-API read (`prop-vals-by-folder`, `pages-by-folder`, `row-index`, `page-by-id`) when you expect rows.
- The same OQL run via `qo run` / `qo query` (against a stored impl, or as a `qo query` of the same scratch body) returns rows.
- `time omega-cli run-query ...` reports tens of milliseconds — too fast for a network round-trip to a remote host. (A real production round-trip is ~100ms+.)
- The agent is mid-triage on `omega/query/no-return` with `:failed-at nil` and the four dispatch-layer checks all pass cleanly — drop into body bisection (per [triage-no-return](../how-to/triage-no-return.md)), but **also confirm the host first** before bisecting.

## Why this hasn't been "fixed" upstream

`omega-cli` is the engine-side CLI tool (lower-level than `qo`); its `localhost:3001` default exists because its primary use is local engine development. The `qo` CLI is the workspace-aware tool that knows about projects and `.env` files. They serve different audiences and the divergence is intentional. The reader-side fix is to know about the divergence — which is what this gotcha exists to surface.

## Related

- [develop-oql.md § The push-run cycle](../how-to/develop-oql.md#the-push-run-cycle) — the canonical OQL authoring loop, which already uses `omega-cli run-query -h $OMEGA_URL ...`. This gotcha explains *why* the `-h` flag is non-optional.
- [develop-oql-implementations.md](../how-to/develop-oql-implementations.md) — the simpler push-run summary; the host caveat applies there too.
- [triage-no-return.md](../how-to/triage-no-return.md) — the no-return triage flow. Body-level zero-solutions bisection (check 5) will not find a bug if the host is wrong; verify host before bisecting.
- [`qo` env vars](https://github.com/The-Lambda-Group/query-omega-cli/blob/master/docs/reference/env-vars.md) — `OMEGA_URL`, `QO_APP_ID`, `QO_WORKSPACE_ID` and how `qo` reads them from `.env`.
- [`qo` env loading](https://github.com/The-Lambda-Group/query-omega-cli/blob/master/docs/reference/env-loading.md) — precedence rules between shell-exported env and project `.env` for `qo`. The shell-export-wins rule that affects `qo` also covers option 2 above for `omega-cli`.
