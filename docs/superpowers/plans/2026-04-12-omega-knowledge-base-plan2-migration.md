# Omega Knowledge Base Plan 2 — Migration & Pruning

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Migrate the existing `omega-docs` repo to the Diátaxis+gotchas 4-bucket taxonomy, seed `/docs/` trees in every project repo listed in `omega-knowledge-base/projects/`, and prune `MEMORY.md` to session-state only.

**Architecture:** Three independent phases plus a final smoke test. Phase A restructures `omega-docs` with pure `git mv` operations (no content rewrites beyond link updates). Phase B creates templated `/docs/` scaffolding in each project repo and migrates the small amount of existing doc content. Phase C deletes memory files whose content now has a canonical home in the knowledge base and rewrites `MEMORY.md` to its pruned form. Phase D smoke-tests by invoking the navigator skill and walking the tree.

**Tech stack:** git, Markdown. No new code, no test framework. Verification happens via `ls`, `find`, `grep`, and manually resolving a handful of cross-links.

---

## Pre-flight notes

- **Plan 1 is a hard prerequisite.** The `omega-knowledge-base` repo must exist at `~/Development/omega/omega-knowledge-base` with the three `omega:omega-docs-*` skills installed. If either is missing, stop and reinstall via `/plugin install omega@omega-local`.

- **Commit discipline.** Every task ends with a commit. Use `docs:` prefix for content/structure commits, `chore:` for memory-only operations. Never add `Co-Authored-By` trailers (see `feedback_no_coauthored_by`).

- **Use `git -C <repo>` for all git operations**, never `cd && git` (see `feedback_git_no_cd`). This plan touches ~13 different git repos, so rooting every command at its repo keeps the shell state clean.

- **Uncommitted state in `query-omega-oql`.** The working tree in that repo has two unrelated modifications (`.gitignore`, `oql/app/database/property.oql`) that predate this plan. Before running any git op in that repo, decide whether to commit, stash, or leave. Phase B Task B4 operates on that repo — do not bundle those unrelated changes into a Plan 2 commit.

- **Link updates are mechanical.** When a file moves from `omegadb/execution-model.md` to `explanation/oql-execution-model.md`, its own header links (`[< Home](../README.md)`) and any inbound links from sibling files must be rewritten. Each Phase A task does its own link rewriting inline; Task A7 does a final `grep` sweep to catch anything missed.

- **No TDD for file moves.** This plan does not have a traditional failing-test-first structure because the work is restructuring, not new behavior. "Verification" steps replace "run the test" steps — each task ends with a concrete `ls`/`find`/`grep` invocation and its expected output.

- **No skill installs or hook changes.** Plan 1 handled those. Plan 2 touches only content.

---

## File structure produced by this plan

### `omega-docs` after Phase A

```
omega-docs/
  README.md                     ← rewritten to index the four buckets
  how-to/
    README.md                   ← new stub (only empty bucket today)
  reference/
    README.md                   ← updated to index reference files
    api.md                      ← from reference/api.md (unchanged)
    protocols.md                ← from reference/protocols.md (unchanged)
    built-ins.md                ← from omegadb/language/built-ins.md
    clauses.md                  ← from omegadb/language/clauses.md
    control-flow.md             ← from omegadb/language/control-flow.md
    datastores.md               ← from omegadb/language/datastores.md
    reducers.md                 ← from omegadb/language/reducers.md
    tools.md                    ← from tools/README.md
  explanation/
    README.md                   ← new
    what-is-omega.md            ← from intro.md
    philosophy.md               ← from philosophy.md (unchanged filename)
    oql-language.md             ← from omegadb/language.md
    oql-execution-model.md      ← from omegadb/execution-model.md
    components.md               ← from omega-ai/components.md
    pages.md                    ← from omega-ai/pages.md
    plugins.md                  ← from omega-ai/plugins.md
  gotchas/
    README.md                   ← new
    lexical-scope.md            ← from omegadb/lexical-scope.md
    conventions.md              ← from omegadb/language/conventions.md
  docs/
    superpowers/                ← UNCHANGED — specs + plans live here
  pricing/
    *.html                      ← UNCHANGED — marketing pages
  site/
    *                           ← UNCHANGED — built static site
  tools/
    gen_*.py                    ← UNCHANGED — asset generation scripts stay
```

Deleted after Phase A: empty `architecture/` and `guides/` dirs, `omegadb/` (emptied by moves), `omega-ai/` (emptied), `intro.md` (moved).

### Per-project `/docs/` trees after Phase B

Every project in `omega-knowledge-base/projects/` (except `omega-docs` which is the root of its own taxonomy and is handled in Phase A) gets:

```
<project>/docs/
  README.md
  how-to/
    README.md
  reference/
    README.md
  explanation/
    README.md
  gotchas/
    README.md
```

Projects touched:

| # | Project | Repo path | Existing content to migrate |
|---|---|---|---|
| 1 | omega-db | `~/Development/omega/omega-db` | none — greenfield + 2 seeded gotchas |
| 2 | query-omega-oql | `~/Development/omega/query-omega-oql` | none (superpowers/ stays out of scope) |
| 3 | query-omega-api | `~/Development/omega/query-omega-api` | none |
| 4 | query-omega-cli | `~/Development/omega/query-omega-cli` | `docs/commands.md` → `docs/reference/commands.md` |
| 5 | query-omega-mcp | `~/Development/omega/query-omega-mcp` | none |
| 6 | omega-cli | `~/Development/omega/omega-cli` | none |
| 7 | omega-ai-components | `~/Development/omega/omega-ai-components` | none |
| 8 | query-omega | `~/Development/omega/query-omega` | none |
| 9 | query-omega-website | `~/Development/omega/query-omega-website` | none |
| 10 | homes-dot-com | `~/Development/wttc/homes-dot-com` | none |
| 11 | gohighlevel | `~/Development/wttc/gohighlevel` | none |

Eleven project trees in all. Task B2 batches the eight pure-greenfield ones; Tasks B3/B4/B5 handle the ones with existing content or special notes.

### MEMORY.md after Phase C

~25 lines containing: hard rules, a single pointer to `omega-knowledge-base`, and an index of the ~15 remaining memory files. Deleted files:

- `feedback_cd_before_qo.md`
- `feedback_deploy_readme.md`
- `feedback_http_request_only.md`
- `feedback_nvmrc_standard.md`
- `feedback_read_docs_before_writing.md` (orphan-resolved → delete, content lives in contributor/how-to)

Kept in memory (behavioral / process rules that can't be docs):

- All `feedback_*.md` files except the five above
- `project_current_work.md`, `project_omega.md`, `project_wttc_contract.md`, `project_orphaned_data.md`, `project_property_rename_gap.md`
- `reference_omega_db_internals.md`, `reference_ui_frontend.md`
- `next_session.md` (rewritten for post-Plan-2 state)

---

# Phase A — Migrate omega-docs into Diátaxis+gotchas

## Task A1: Create 4-bucket scaffolding at omega-docs root

**Files:**
- Create: `~/Development/omega/omega-docs/how-to/README.md`
- Create: `~/Development/omega/omega-docs/reference/README.md` (overwrite — dir already exists with 2 files)
- Create: `~/Development/omega/omega-docs/explanation/README.md`
- Create: `~/Development/omega/omega-docs/gotchas/README.md`

- [ ] **Step 1: Check current state of omega-docs root**

Run: `ls ~/Development/omega/omega-docs/`

Expected: sees `README.md`, `architecture/`, `docs/`, `guides/`, `intro.md`, `omega-ai/`, `omegadb/`, `philosophy.md`, `pricing/`, `reference/`, `site/`, `tools/`. The `reference/` dir already exists (containing `api.md` and `protocols.md`). The `explanation/`, `gotchas/`, and `how-to/` dirs do not exist yet.

- [ ] **Step 2: Create the `how-to/` bucket with a README**

Write `~/Development/omega/omega-docs/how-to/README.md`:

```markdown
[< Home](../README.md)

# How-to

Task recipes for working with Omega. Each entry is a step-by-step guide to accomplishing a specific task.

This bucket is currently empty. Candidate guides to add:

- How to install a plugin
- How to push a component
- How to run a one-shot query against the public API
- How to trace a failed run through the log stream

Add new entries as markdown files in this directory and link them from this README.
```

- [ ] **Step 3: Create the `explanation/` bucket with a README**

Write `~/Development/omega/omega-docs/explanation/README.md`:

```markdown
[< Home](../README.md)

# Explanation

The "why" of Omega — design rationale, architecture, mechanisms. Read these when you want to understand how a piece of the system works, not how to use it.

## What is Omega

- [What is Omega?](what-is-omega.md) — elevator pitch for the platform
- [Philosophy](philosophy.md) — design principles and motivation

## OQL

- [OQL Language](oql-language.md) — logic programming concepts, clauses, constraints
- [Execution Model](oql-execution-model.md) — solution sets, clause boundaries, performance

## OmegaAI

- [Pages](pages.md) — the page tree, blocks, page model
- [Components](components.md) — components, protocols, implementations
- [Plugins](plugins.md) — plugin lifecycle, actor model, service protocol
```

- [ ] **Step 4: Create the `gotchas/` bucket with a README**

Write `~/Development/omega/omega-docs/gotchas/README.md`:

```markdown
[< Home](../README.md)

# Gotchas

Failure modes, footguns, and debugging patterns. Read these when something surprising happened or when you're trying to diagnose a bug. Design-level rationale lives in `explanation/`; this bucket is for "things that bit us."

## Language-level

- [Lexical scope](lexical-scope.md) — how scoping works in clause bodies; subtle stale-value bugs
- [Conventions](conventions.md) — boolean stringification, return-binding rules, common footguns

Deeper project-specific OQL gotchas live in `omega-knowledge-base/projects/query-omega-oql/gotchas/`. This bucket is for gotchas that are part of the public language surface.
```

- [ ] **Step 5: Overwrite the existing `reference/README.md` (or create if missing)**

Check current state first: `ls ~/Development/omega/omega-docs/reference/` — expected: `api.md`, `protocols.md`, no `README.md`.

Write `~/Development/omega/omega-docs/reference/README.md`:

```markdown
[< Home](../README.md)

# Reference

Factual lookup for Omega APIs, schemas, and IDs. Read these when you need exact signatures, protocol definitions, or command inventories.

## Language reference

- [Clauses](clauses.md) — clause syntax and head/body semantics
- [Built-in terms](built-ins.md) — the core built-in term set
- [Control flow](control-flow.md) — `when`, `when-not`, `if`, `with-table-if`
- [Datastores](datastores.md) — datastore and functor definitions
- [Reducers](reducers.md) — solution table operations

## API & protocols

- [Public API](api.md) — OQL API clause signatures
- [Protocols](protocols.md) — protocol IDs and JSON specs

## Tooling

- [Tools](tools.md) — asset-generation scripts shipped with omega-docs
```

- [ ] **Step 6: Verify all four buckets exist**

Run: `ls ~/Development/omega/omega-docs/how-to/ ~/Development/omega/omega-docs/reference/ ~/Development/omega/omega-docs/explanation/ ~/Development/omega/omega-docs/gotchas/`

Expected:
- `how-to/` contains only `README.md`
- `reference/` contains `README.md`, `api.md`, `protocols.md`
- `explanation/` contains only `README.md`
- `gotchas/` contains only `README.md`

- [ ] **Step 7: Commit**

```bash
git -C ~/Development/omega/omega-docs add how-to/README.md reference/README.md explanation/README.md gotchas/README.md
git -C ~/Development/omega/omega-docs commit -m "docs: scaffold four-bucket Diátaxis taxonomy at omega-docs root"
```

---

## Task A2: Migrate `omegadb/` files

**Files:**
- Move: `omegadb/execution-model.md` → `explanation/oql-execution-model.md`
- Move: `omegadb/language.md` → `explanation/oql-language.md`
- Move: `omegadb/lexical-scope.md` → `gotchas/lexical-scope.md`

Rationale:
- `execution-model.md` describes mechanism ("how OQL executes") → explanation
- `language.md` is an overview of concepts and syntax ("OQL is a logic language…") → explanation (it's introductory, not a reference list)
- `lexical-scope.md` opens with "getting them wrong produces subtle bugs where clauses appear to work but silently use stale or wrong values" → gotchas

- [ ] **Step 1: Move `execution-model.md`**

```bash
git -C ~/Development/omega/omega-docs mv omegadb/execution-model.md explanation/oql-execution-model.md
```

- [ ] **Step 2: Rewrite header links in the moved file**

The old file starts with `[< Home](../README.md) | [OmegaDB](../README.md#omegadb)`. At its new depth (1 level from root instead of 1), `../README.md` still resolves correctly, but the `#omegadb` anchor will be gone after the root README is rewritten in Task A7. Rewrite to point at the new bucket README.

Edit `~/Development/omega/omega-docs/explanation/oql-execution-model.md`:

Replace `[< Home](../README.md) | [OmegaDB](../README.md#omegadb)` with `[< Home](../README.md) | [Explanation](README.md)`.

- [ ] **Step 3: Move `language.md`**

```bash
git -C ~/Development/omega/omega-docs mv omegadb/language.md explanation/oql-language.md
```

- [ ] **Step 4: Rewrite header links**

Edit `~/Development/omega/omega-docs/explanation/oql-language.md`:

Replace `[< Home](../README.md) | [OmegaDB](../README.md#omegadb)` with `[< Home](../README.md) | [Explanation](README.md)`.

Also scan for any inbound link like `[Language](../language.md)` that children previously used — those will be fixed when Task A3 moves the children.

- [ ] **Step 5: Move `lexical-scope.md`**

```bash
git -C ~/Development/omega/omega-docs mv omegadb/lexical-scope.md gotchas/lexical-scope.md
```

- [ ] **Step 6: Rewrite header links**

Edit `~/Development/omega/omega-docs/gotchas/lexical-scope.md`:

Replace `[< Home](../README.md) | [OmegaDB](../README.md#omegadb)` with `[< Home](../README.md) | [Gotchas](README.md)`.

- [ ] **Step 7: Verify moves**

Run: `ls ~/Development/omega/omega-docs/omegadb/ ~/Development/omega/omega-docs/explanation/ ~/Development/omega/omega-docs/gotchas/`

Expected:
- `omegadb/` contains only `language/` subdirectory (the 3 top-level files are gone)
- `explanation/` now has `README.md`, `oql-execution-model.md`, `oql-language.md`
- `gotchas/` now has `README.md`, `lexical-scope.md`

- [ ] **Step 8: Commit**

```bash
git -C ~/Development/omega/omega-docs add -A
git -C ~/Development/omega/omega-docs commit -m "docs: migrate omegadb/ top-level files into explanation/ and gotchas/"
```

---

## Task A3: Migrate `omegadb/language/` files

**Files:**
- Move: `omegadb/language/built-ins.md` → `reference/built-ins.md`
- Move: `omegadb/language/clauses.md` → `reference/clauses.md`
- Move: `omegadb/language/control-flow.md` → `reference/control-flow.md`
- Move: `omegadb/language/datastores.md` → `reference/datastores.md`
- Move: `omegadb/language/reducers.md` → `reference/reducers.md`
- Move: `omegadb/language/conventions.md` → `gotchas/conventions.md`

Rationale: the 5 reference files are factual lookup (lists of built-ins, clause syntax tables, reducer catalogues). `conventions.md` is explicitly titled "Conventions & Gotchas" and contains booleans-as-strings and return-binding pitfalls → gotchas.

- [ ] **Step 1: Move the 5 reference files**

```bash
git -C ~/Development/omega/omega-docs mv omegadb/language/built-ins.md reference/built-ins.md
git -C ~/Development/omega/omega-docs mv omegadb/language/clauses.md reference/clauses.md
git -C ~/Development/omega/omega-docs mv omegadb/language/control-flow.md reference/control-flow.md
git -C ~/Development/omega/omega-docs mv omegadb/language/datastores.md reference/datastores.md
git -C ~/Development/omega/omega-docs mv omegadb/language/reducers.md reference/reducers.md
```

- [ ] **Step 2: Rewrite header links in each reference file**

Each of the 5 files currently starts with `[< Language](../language.md) | [OmegaDB](../../README.md#omegadb)`. Rewrite each to `[< Home](../README.md) | [Reference](README.md)`.

Run these five edits:

```
reference/built-ins.md:      [< Language](../language.md) | [OmegaDB](../../README.md#omegadb)   →   [< Home](../README.md) | [Reference](README.md)
reference/clauses.md:        [< Language](../language.md) | [OmegaDB](../../README.md#omegadb)   →   [< Home](../README.md) | [Reference](README.md)
reference/control-flow.md:   [< Language](../language.md) | [OmegaDB](../../README.md#omegadb)   →   [< Home](../README.md) | [Reference](README.md)
reference/datastores.md:     [< Language](../language.md) | [OmegaDB](../../README.md#omegadb)   →   [< Home](../README.md) | [Reference](README.md)
reference/reducers.md:       [< Language](../language.md) | [OmegaDB](../../README.md#omegadb)   →   [< Home](../README.md) | [Reference](README.md)
```

- [ ] **Step 3: Move `conventions.md` to gotchas**

```bash
git -C ~/Development/omega/omega-docs mv omegadb/language/conventions.md gotchas/conventions.md
```

- [ ] **Step 4: Rewrite header links in `gotchas/conventions.md`**

Replace `[< Language](../language.md) | [OmegaDB](../../README.md#omegadb)` with `[< Home](../README.md) | [Gotchas](README.md)`.

- [ ] **Step 5: Verify `omegadb/` is now empty and remove it**

Run: `ls ~/Development/omega/omega-docs/omegadb/`

Expected: `language/` is empty. Run:

```bash
ls ~/Development/omega/omega-docs/omegadb/language/
```

Expected: empty output. Remove both directories:

```bash
rmdir ~/Development/omega/omega-docs/omegadb/language ~/Development/omega/omega-docs/omegadb
```

- [ ] **Step 6: Verify final state**

Run: `ls ~/Development/omega/omega-docs/reference/ ~/Development/omega/omega-docs/gotchas/`

Expected:
- `reference/`: `README.md`, `api.md`, `built-ins.md`, `clauses.md`, `control-flow.md`, `datastores.md`, `protocols.md`, `reducers.md`
- `gotchas/`: `README.md`, `conventions.md`, `lexical-scope.md`

Run: `ls ~/Development/omega/omega-docs/ | grep omegadb`

Expected: no output (directory gone).

- [ ] **Step 7: Commit**

```bash
git -C ~/Development/omega/omega-docs add -A
git -C ~/Development/omega/omega-docs commit -m "docs: migrate omegadb/language/ into reference/ and gotchas/"
```

---

## Task A4: Migrate `omega-ai/` files

**Files:**
- Move: `omega-ai/components.md` → `explanation/components.md`
- Move: `omega-ai/pages.md` → `explanation/pages.md`
- Move: `omega-ai/plugins.md` → `explanation/plugins.md`

Rationale: all three describe mechanism and design of OmegaAI's code model — they are explanation, not reference.

- [ ] **Step 1: Move the 3 files**

```bash
git -C ~/Development/omega/omega-docs mv omega-ai/components.md explanation/components.md
git -C ~/Development/omega/omega-docs mv omega-ai/pages.md explanation/pages.md
git -C ~/Development/omega/omega-docs mv omega-ai/plugins.md explanation/plugins.md
```

- [ ] **Step 2: Rewrite header links**

Each file currently starts with `[< Home](../README.md) | [OmegaAI](../README.md#omegaai)`. Rewrite each to `[< Home](../README.md) | [Explanation](README.md)`.

```
explanation/components.md: [< Home](../README.md) | [OmegaAI](../README.md#omegaai)  →  [< Home](../README.md) | [Explanation](README.md)
explanation/pages.md:      [< Home](../README.md) | [OmegaAI](../README.md#omegaai)  →  [< Home](../README.md) | [Explanation](README.md)
explanation/plugins.md:    [< Home](../README.md) | [OmegaAI](../README.md#omegaai)  →  [< Home](../README.md) | [Explanation](README.md)
```

- [ ] **Step 3: Remove the empty `omega-ai/` dir**

Run: `ls ~/Development/omega/omega-docs/omega-ai/`

Expected: empty. Then: `rmdir ~/Development/omega/omega-docs/omega-ai`

- [ ] **Step 4: Verify `explanation/` state**

Run: `ls ~/Development/omega/omega-docs/explanation/`

Expected: `README.md`, `components.md`, `oql-execution-model.md`, `oql-language.md`, `pages.md`, `plugins.md`. (Still missing `what-is-omega.md` and `philosophy.md` — those come in Task A6.)

- [ ] **Step 5: Commit**

```bash
git -C ~/Development/omega/omega-docs add -A
git -C ~/Development/omega/omega-docs commit -m "docs: migrate omega-ai/ into explanation/"
```

---

## Task A5: Migrate `reference/*.md` header links + `tools/README.md`

The existing `reference/api.md` and `reference/protocols.md` files did not move, but they still have outdated header links pointing at the old root README sections. Also move `tools/README.md` to `reference/tools.md`.

**Files:**
- Edit: `reference/api.md` — fix header link
- Edit: `reference/protocols.md` — fix header link
- Move: `tools/README.md` → `reference/tools.md`

- [ ] **Step 1: Rewrite header link in `reference/api.md`**

Replace `[< Home](../README.md) | [Reference](../README.md#reference)` with `[< Home](../README.md) | [Reference](README.md)`.

- [ ] **Step 2: Rewrite header link in `reference/protocols.md`**

Replace `[< Home](../README.md) | [Reference](../README.md#reference)` with `[< Home](../README.md) | [Reference](README.md)`.

- [ ] **Step 3: Move `tools/README.md`**

```bash
git -C ~/Development/omega/omega-docs mv tools/README.md reference/tools.md
```

Note: `tools/` dir still contains the `gen_*.py` asset-generation scripts. Those stay put — they're code, not docs. Only the README moves.

- [ ] **Step 4: Rewrite `reference/tools.md`'s own links if any**

Read the first ~10 lines of `~/Development/omega/omega-docs/reference/tools.md` to check for outbound links. The file currently sits one level deep (`tools/`); after moving to `reference/`, any relative links to sibling files in `tools/` need to become `../tools/<name>`. If no such links exist, skip this step.

If links exist that reference the asset scripts like `gen_dashboard.py`, rewrite them to `../tools/gen_dashboard.py`.

- [ ] **Step 5: Verify**

Run: `ls ~/Development/omega/omega-docs/reference/ ~/Development/omega/omega-docs/tools/`

Expected:
- `reference/`: contains `tools.md` (along with README and everything else from A3)
- `tools/`: contains only `gen_*.py` files — no README anymore

- [ ] **Step 6: Commit**

```bash
git -C ~/Development/omega/omega-docs add -A
git -C ~/Development/omega/omega-docs commit -m "docs: move tools/README.md to reference/tools.md; fix reference header links"
```

---

## Task A6: Migrate root files (`intro.md`, `philosophy.md`)

**Files:**
- Move: `intro.md` → `explanation/what-is-omega.md`
- Move: `philosophy.md` → `explanation/philosophy.md`

- [ ] **Step 1: Move `intro.md`**

```bash
git -C ~/Development/omega/omega-docs mv intro.md explanation/what-is-omega.md
```

The move does two things: changes the filename (more descriptive for the new taxonomy) and changes the depth. The old `intro.md` was at the root; at depth 1 inside `explanation/`, the existing `[< Home](README.md)` link becomes `[< Home](../README.md)`.

- [ ] **Step 2: Rewrite the header link**

Edit `~/Development/omega/omega-docs/explanation/what-is-omega.md`:

Replace `[< Home](README.md)` (at the top of the file) with `[< Home](../README.md) | [Explanation](README.md)`.

- [ ] **Step 3: Move `philosophy.md`**

```bash
git -C ~/Development/omega/omega-docs mv philosophy.md explanation/philosophy.md
```

- [ ] **Step 4: Check whether `philosophy.md` had a header link and update if present**

`philosophy.md` starts with `# OmegaAI Philosophy & Motivation` (no markdown link at the top). No rewrite needed. Confirm by reading the first 3 lines of `~/Development/omega/omega-docs/explanation/philosophy.md`.

- [ ] **Step 5: Verify**

Run: `ls ~/Development/omega/omega-docs/ | head -20`

Expected: no `intro.md`, no `philosophy.md` at root. `explanation/` now contains 8 entries (README + 7 files).

- [ ] **Step 6: Commit**

```bash
git -C ~/Development/omega/omega-docs add -A
git -C ~/Development/omega/omega-docs commit -m "docs: migrate intro.md and philosophy.md into explanation/"
```

---

## Task A7: Rewrite root README; remove empty dirs; global link sweep

**Files:**
- Rewrite: `~/Development/omega/omega-docs/README.md`
- Delete: `architecture/` and `guides/` directories (both empty since start)
- Grep sweep: every `.md` file under `~/Development/omega/omega-docs/` outside `docs/superpowers/` for broken links

- [ ] **Step 1: Rewrite the root README to index the 4 buckets**

Write `~/Development/omega/omega-docs/README.md`:

```markdown
# Omega Documentation

Documentation for the [Omega](https://github.com/The-Lambda-Group) platform, organized as a Diátaxis+gotchas tree. Every top-level bucket indexes its own entries — start here, then descend.

## Navigation

- [how-to/](how-to/README.md) — task recipes ("how do I do X")
- [reference/](reference/README.md) — API, schemas, protocol IDs, command lists ("what is the signature of X")
- [explanation/](explanation/README.md) — design rationale, architecture, mechanisms ("how does X work")
- [gotchas/](gotchas/README.md) — failure modes and debugging patterns ("why did X fail")

## If you're working with an agent

This repo is the public half of the Omega knowledge base. Agents should start at the private root at `~/Development/omega/omega-knowledge-base/README.md` and descend from there — the private root links back into this repo.

The taxonomy is described in detail at `omega-knowledge-base/contributor/reference/diataxis.md`.

## Related

- [query-omega-cli commands](https://github.com/The-Lambda-Group/query-omega-cli/blob/master/docs/reference/commands.md) — `qo` command reference (lives in that repo's `/docs/` tree after Plan 2)
- [MCP Server](https://github.com/The-Lambda-Group/query-omega-mcp)
```

- [ ] **Step 2: Delete the empty `architecture/` and `guides/` dirs**

Run: `ls ~/Development/omega/omega-docs/architecture/ ~/Development/omega/omega-docs/guides/`

Expected: both empty. Then remove:

```bash
rmdir ~/Development/omega/omega-docs/architecture ~/Development/omega/omega-docs/guides
```

Git won't track empty-dir removal directly, but they'll disappear from the working tree. The commit step below captures everything.

- [ ] **Step 3: Grep sweep for stale links to old paths**

Run:

```bash
cd ~/Development/omega/omega-docs && grep -rn --include='*.md' -E '\((\.\./)?(omegadb|omega-ai|intro\.md|philosophy\.md)/' . | grep -v '^\./docs/superpowers/' | grep -v '^\./site/'
```

Expected: **no output**. If any matches appear, they're stale links that need rewriting. For each hit:
- If the target was moved in Tasks A2–A6, rewrite the link to the new path.
- If it's inside `docs/superpowers/` (specs or plans), leave it alone — those are historical records.

Common fixes:
- `[Language](../language.md)` inside `reference/*.md` → no such file; rewrite to `[OQL language overview](../explanation/oql-language.md)` or delete if redundant with the new header link.

- [ ] **Step 4: Verify final omega-docs top-level state**

Run: `ls ~/Development/omega/omega-docs/`

Expected exactly: `README.md`, `docs/`, `explanation/`, `gotchas/`, `how-to/`, `pricing/`, `reference/`, `site/`, `tools/`.

- [ ] **Step 5: Commit**

```bash
git -C ~/Development/omega/omega-docs add -A
git -C ~/Development/omega/omega-docs commit -m "docs: rewrite root README for 4-bucket navigation; remove empty architecture/ and guides/"
```

- [ ] **Step 6: Sanity-check that the four bucket READMEs all resolve back to root**

Run: `for d in how-to reference explanation gotchas; do head -1 ~/Development/omega/omega-docs/$d/README.md; done`

Expected: four lines, each `[< Home](../README.md)`.

---

# Phase B — Per-project `/docs/` trees

## Task B1: Write the seed-template helper

This task creates a single shell helper that stamps out the 4-bucket `/docs/` skeleton in any project repo. Every subsequent Task B* invokes this helper. Writing it once avoids 44 near-identical README stubs scattered across tasks.

**Files:**
- Create: `~/Development/omega/omega-knowledge-base/scripts/seed-project-docs.sh`

- [ ] **Step 1: Write the helper script**

Write `~/Development/omega/omega-knowledge-base/scripts/seed-project-docs.sh`:

```bash
#!/usr/bin/env bash
# Seed a project repo's /docs/ tree with the Diátaxis+gotchas 4-bucket structure.
#
# Usage:
#   seed-project-docs.sh <project-repo-path> <project-display-name>
#
# Creates <repo>/docs/{README.md,how-to/README.md,reference/README.md,explanation/README.md,gotchas/README.md}
# If any file already exists, refuses to overwrite (fails loud — the caller should decide).

set -euo pipefail

repo="${1:?usage: $0 <repo-path> <display-name>}"
name="${2:?usage: $0 <repo-path> <display-name>}"

if [[ ! -d "$repo" ]]; then
  echo "error: $repo is not a directory" >&2
  exit 2
fi

docs="$repo/docs"
mkdir -p "$docs/how-to" "$docs/reference" "$docs/explanation" "$docs/gotchas"

write_if_missing() {
  local path="$1"
  local body="$2"
  if [[ -e "$path" ]]; then
    echo "skip (exists): $path" >&2
    return 0
  fi
  printf '%s\n' "$body" > "$path"
  echo "wrote: $path" >&2
}

write_if_missing "$docs/README.md" "[< Repo](../README.md)

# $name — Docs

Project documentation for $name, organized by the Diátaxis+gotchas taxonomy. Each bucket has its own index; start here and descend.

- [how-to/](how-to/README.md) — task recipes
- [reference/](reference/README.md) — API, schemas, command lists
- [explanation/](explanation/README.md) — architecture, design rationale, mechanisms
- [gotchas/](gotchas/README.md) — failure modes, footguns, debugging patterns

This tree is indexed from the private root at \`~/Development/omega/omega-knowledge-base/projects/$(basename "$repo").md\`."

write_if_missing "$docs/how-to/README.md" "[< Docs](../README.md)

# How-to — $name

Task recipes specific to $name. Empty at seed time; fill as gaps are recorded."

write_if_missing "$docs/reference/README.md" "[< Docs](../README.md)

# Reference — $name

Factual lookup for $name: API signatures, schemas, config, IDs. Empty at seed time; fill as gaps are recorded."

write_if_missing "$docs/explanation/README.md" "[< Docs](../README.md)

# Explanation — $name

Architecture, design rationale, and mechanism notes for $name. Empty at seed time; fill as gaps are recorded."

write_if_missing "$docs/gotchas/README.md" "[< Docs](../README.md)

# Gotchas — $name

Failure modes, footguns, and debugging patterns for $name. Empty at seed time; fill as gaps are recorded."
```

- [ ] **Step 2: Make it executable**

```bash
chmod +x ~/Development/omega/omega-knowledge-base/scripts/seed-project-docs.sh
```

- [ ] **Step 3: Dry-test the helper against a throwaway location**

Run:

```bash
TMPDIR=$(mktemp -d) && ~/Development/omega/omega-knowledge-base/scripts/seed-project-docs.sh "$TMPDIR" "test-project" && find "$TMPDIR/docs" -type f && rm -rf "$TMPDIR"
```

Expected: five lines ending in `README.md`, one per file created:
```
.../docs/README.md
.../docs/how-to/README.md
.../docs/reference/README.md
.../docs/explanation/README.md
.../docs/gotchas/README.md
```

- [ ] **Step 4: Commit the helper**

```bash
git -C ~/Development/omega/omega-knowledge-base add scripts/seed-project-docs.sh
git -C ~/Development/omega/omega-knowledge-base commit -m "scripts: add seed-project-docs.sh for stamping /docs/ trees"
```

---

## Task B2: Seed 9 greenfield projects

These projects have no existing `/docs/` content and get the stock template with no migration work. Each seed is its own commit in its own repo.

**Projects:**

| # | Repo | Display name |
|---|---|---|
| 1 | `~/Development/omega/omega-db` | OmegaDB |
| 2 | `~/Development/omega/query-omega-api` | Query Omega API |
| 3 | `~/Development/omega/query-omega-mcp` | Query Omega MCP |
| 4 | `~/Development/omega/omega-cli` | Omega CLI |
| 5 | `~/Development/omega/omega-ai-components` | Omega AI Components |
| 6 | `~/Development/omega/query-omega` | Query Omega Frontend |
| 7 | `~/Development/omega/query-omega-website` | Query Omega Website |
| 8 | `~/Development/wttc/homes-dot-com` | Homes.com Plugin |
| 9 | `~/Development/wttc/gohighlevel` | GoHighLevel Plugin |

For each project, run the sequence below. Do them one at a time so each commit is atomic and reviewable.

- [ ] **Step 1: Seed omega-db**

```bash
~/Development/omega/omega-knowledge-base/scripts/seed-project-docs.sh ~/Development/omega/omega-db "OmegaDB"
git -C ~/Development/omega/omega-db add docs/
git -C ~/Development/omega/omega-db commit -m "docs: seed 4-bucket taxonomy /docs/ tree"
```

Verify: `ls ~/Development/omega/omega-db/docs/` should show `README.md`, `explanation/`, `gotchas/`, `how-to/`, `reference/`.

- [ ] **Step 2: Seed query-omega-api**

```bash
~/Development/omega/omega-knowledge-base/scripts/seed-project-docs.sh ~/Development/omega/query-omega-api "Query Omega API"
git -C ~/Development/omega/query-omega-api add docs/
git -C ~/Development/omega/query-omega-api commit -m "docs: seed 4-bucket taxonomy /docs/ tree"
```

- [ ] **Step 3: Seed query-omega-mcp**

```bash
~/Development/omega/omega-knowledge-base/scripts/seed-project-docs.sh ~/Development/omega/query-omega-mcp "Query Omega MCP"
git -C ~/Development/omega/query-omega-mcp add docs/
git -C ~/Development/omega/query-omega-mcp commit -m "docs: seed 4-bucket taxonomy /docs/ tree"
```

- [ ] **Step 4: Seed omega-cli**

```bash
~/Development/omega/omega-knowledge-base/scripts/seed-project-docs.sh ~/Development/omega/omega-cli "Omega CLI"
git -C ~/Development/omega/omega-cli add docs/
git -C ~/Development/omega/omega-cli commit -m "docs: seed 4-bucket taxonomy /docs/ tree"
```

- [ ] **Step 5: Seed omega-ai-components**

```bash
~/Development/omega/omega-knowledge-base/scripts/seed-project-docs.sh ~/Development/omega/omega-ai-components "Omega AI Components"
git -C ~/Development/omega/omega-ai-components add docs/
git -C ~/Development/omega/omega-ai-components commit -m "docs: seed 4-bucket taxonomy /docs/ tree"
```

- [ ] **Step 6: Seed query-omega (frontend)**

Before running, verify `~/Development/omega/query-omega` has no uncommitted work that could get bundled into the commit: `git -C ~/Development/omega/query-omega status -s`. If dirty, pause and report to the user.

```bash
~/Development/omega/omega-knowledge-base/scripts/seed-project-docs.sh ~/Development/omega/query-omega "Query Omega Frontend"
git -C ~/Development/omega/query-omega add docs/
git -C ~/Development/omega/query-omega commit -m "docs: seed 4-bucket taxonomy /docs/ tree"
```

- [ ] **Step 7: Seed query-omega-website**

First, check if this project already has a `docs/` dir: `ls ~/Development/omega/query-omega-website/docs/ 2>/dev/null`. The project was listed in the handoff as having a `docs/superpowers/` dir, which would collide with the 4-bucket seed's `docs/README.md`.

If `docs/` exists and contains only `superpowers/`, the helper's `mkdir -p` + `write_if_missing` is safe — it won't clobber the existing `superpowers/` subtree, and it'll write fresh buckets alongside.

If `docs/` exists with other content, stop and report — manual conflict resolution needed.

```bash
~/Development/omega/omega-knowledge-base/scripts/seed-project-docs.sh ~/Development/omega/query-omega-website "Query Omega Website"
git -C ~/Development/omega/query-omega-website add docs/
git -C ~/Development/omega/query-omega-website commit -m "docs: seed 4-bucket taxonomy /docs/ tree"
```

- [ ] **Step 8: Seed homes-dot-com**

```bash
~/Development/omega/omega-knowledge-base/scripts/seed-project-docs.sh ~/Development/wttc/homes-dot-com "Homes.com Plugin"
git -C ~/Development/wttc/homes-dot-com add docs/
git -C ~/Development/wttc/homes-dot-com commit -m "docs: seed 4-bucket taxonomy /docs/ tree"
```

- [ ] **Step 9: Seed gohighlevel**

```bash
~/Development/omega/omega-knowledge-base/scripts/seed-project-docs.sh ~/Development/wttc/gohighlevel "GoHighLevel Plugin"
git -C ~/Development/wttc/gohighlevel add docs/
git -C ~/Development/wttc/gohighlevel commit -m "docs: seed 4-bucket taxonomy /docs/ tree"
```

- [ ] **Step 10: Verify every project has a tree**

```bash
for p in ~/Development/omega/omega-db ~/Development/omega/query-omega-api ~/Development/omega/query-omega-mcp ~/Development/omega/omega-cli ~/Development/omega/omega-ai-components ~/Development/omega/query-omega ~/Development/omega/query-omega-website ~/Development/wttc/homes-dot-com ~/Development/wttc/gohighlevel; do
  echo "=== $p ==="
  ls "$p/docs/" 2>/dev/null
done
```

Expected: nine `===` blocks, each listing `README.md`, `explanation/`, `gotchas/`, `how-to/`, `reference/`.

---

## Task B3: Seed query-omega-cli with commands.md migration

`query-omega-cli` already has `docs/commands.md` — the `qo` command reference. Migrate it into the new `reference/` bucket.

**Files:**
- Move: `~/Development/omega/query-omega-cli/docs/commands.md` → `~/Development/omega/query-omega-cli/docs/reference/commands.md`
- Create: the four bucket READMEs via the helper

- [ ] **Step 1: Run the seed helper**

```bash
~/Development/omega/omega-knowledge-base/scripts/seed-project-docs.sh ~/Development/omega/query-omega-cli "Query Omega CLI"
```

The helper should `skip (exists)` on `docs/README.md` if one already exists, and write fresh bucket READMEs.

Check current state:

```bash
ls ~/Development/omega/query-omega-cli/docs/
```

Expected: `commands.md`, `README.md` (maybe pre-existing), `superpowers/` (the handoff mentioned this), plus the new bucket directories.

If `docs/README.md` already existed and wasn't overwritten, read its content: `cat ~/Development/omega/query-omega-cli/docs/README.md`. If it's the old pre-taxonomy README, overwrite it with the seed template's version by running `rm` first and re-running the helper:

```bash
rm ~/Development/omega/query-omega-cli/docs/README.md
~/Development/omega/omega-knowledge-base/scripts/seed-project-docs.sh ~/Development/omega/query-omega-cli "Query Omega CLI"
```

- [ ] **Step 2: Move `commands.md` into the reference bucket**

```bash
git -C ~/Development/omega/query-omega-cli mv docs/commands.md docs/reference/commands.md
```

- [ ] **Step 3: Update the reference README to link commands.md**

Edit `~/Development/omega/query-omega-cli/docs/reference/README.md` and replace the body with:

```markdown
[< Docs](../README.md)

# Reference — Query Omega CLI

Factual lookup for the `qo` CLI: command syntax, flags, environment variables.

- [Commands](commands.md) — full `qo` command reference
```

- [ ] **Step 4: Check `commands.md` for broken header links after the move**

Read the first 5 lines of `~/Development/omega/query-omega-cli/docs/reference/commands.md`. If it has a `[< Home]` or similar link at the top pointing at a relative path, it may now be wrong (the file moved one level deeper). Fix if needed:
- Old relative like `[< Back](../README.md)` from `docs/commands.md` would need to become `[< Back](../README.md)` → no, that's now `../README.md` pointing from `docs/reference/commands.md` at `docs/README.md`. Still correct. Only fix if the link targets something outside `docs/`.

- [ ] **Step 5: Commit**

```bash
git -C ~/Development/omega/query-omega-cli add docs/
git -C ~/Development/omega/query-omega-cli commit -m "docs: seed 4-bucket taxonomy; move commands.md into reference/"
```

Verify: `ls ~/Development/omega/query-omega-cli/docs/reference/` should show `README.md`, `commands.md`.

---

## Task B4: Seed query-omega-oql `/docs/` tree

`query-omega-oql` is the core OQL library repo. It already has uncommitted changes from previous sessions (see pre-flight note) AND it already has a `docs/superpowers/` subdir. Seed carefully.

**Files:**
- Create: `~/Development/omega/query-omega-oql/docs/{README.md,how-to,reference,explanation,gotchas}/README.md`

- [ ] **Step 1: Verify working-tree state; do NOT touch the two pre-existing unrelated modifications**

```bash
git -C ~/Development/omega/query-omega-oql status
```

Expected: `.gitignore` and `oql/app/database/property.oql` show as modified. These are unrelated to this plan and will be committed separately (or left alone) at the user's discretion. Do not `git add -A` in this repo — add only what the seed helper creates.

- [ ] **Step 2: Run the seed helper**

```bash
~/Development/omega/omega-knowledge-base/scripts/seed-project-docs.sh ~/Development/omega/query-omega-oql "Query Omega OQL"
```

The helper should create `docs/{README.md, how-to/README.md, reference/README.md, explanation/README.md, gotchas/README.md}`. The existing `docs/superpowers/` subtree is untouched (the helper only writes inside the four buckets it creates).

- [ ] **Step 3: Verify the seed result**

```bash
ls ~/Development/omega/query-omega-oql/docs/
```

Expected: `README.md`, `explanation/`, `gotchas/`, `how-to/`, `reference/`, `superpowers/`.

- [ ] **Step 4: Stage only the new files**

```bash
git -C ~/Development/omega/query-omega-oql add docs/README.md docs/how-to/README.md docs/reference/README.md docs/explanation/README.md docs/gotchas/README.md
```

Do NOT use `git add docs/` because that would also pick up `docs/superpowers/` changes (if any) and we want this commit scoped exclusively to the seed.

- [ ] **Step 5: Confirm no unrelated files are staged**

```bash
git -C ~/Development/omega/query-omega-oql status
```

Expected: five `new file:` entries (the five READMEs), and the two pre-existing modifications still showing as `not staged for commit`.

- [ ] **Step 6: Commit**

```bash
git -C ~/Development/omega/query-omega-oql commit -m "docs: seed 4-bucket taxonomy /docs/ tree"
```

---

## Task B5: Seed omega-db gotchas from the handoff

Plan 1 already captured OQL-language gotchas in `omega-knowledge-base/projects/query-omega-oql/gotchas/`. This task captures two omega-db-specific gotchas that came up in the HDC pipeline session: CouchDB skip performance and JVM-restart subscription death.

**Files:**
- Create: `~/Development/omega/omega-db/docs/gotchas/couchdb-skip-include-docs-performance.md`
- Create: `~/Development/omega/omega-db/docs/gotchas/event-subscriptions-die-on-jvm-restart.md`
- Edit: `~/Development/omega/omega-db/docs/gotchas/README.md` to index the new files

- [ ] **Step 1: Write the CouchDB skip gotcha**

Write `~/Development/omega/omega-db/docs/gotchas/couchdb-skip-include-docs-performance.md`:

```markdown
[< Gotchas](README.md)

# CouchDB skip with include_docs is O(skip)

## Symptom

A `page-query` term paginating through a large table (~100k+ docs) starts fast and gets slower with every page. By skip=90000 each batch takes tens of seconds and the sync appears to stall. Replacing the batch size does not help; the wall-clock time per batch keeps growing.

## Cause

CouchDB's Mango `_find` endpoint with `include_docs=true` materializes documents for every row it skips over, not just the rows it returns. So `skip=90000 limit=300` reads 90,300 documents from disk, throws 90,000 away, and returns 300. This is linear in `skip`, so pagination through an N-row table takes O(N²) total work.

## Diagnosis

- Log-stream shows `page-query` events with rising inter-event gap even though batch size is constant.
- CouchDB logs (if accessible) show Mango queries with ever-larger `skip` values and ever-longer execution times.
- The bug is batch-size-independent — it's the skip cost, not the limit cost.

## Workaround

Two options:

1. **Use a keyset cursor instead of offset.** Rewrite the query to use `with-skip` / `with-limit` on a sort key so CouchDB can use an index range read. This is what the `page-query` rewrite in commit `723b3b2` did for `AppFolderPage` — see that commit for the pattern.

2. **Don't paginate — stream.** If the caller only needs a full scan, use a streaming read instead of offset pagination.

## Not a bug in

- OQL (the query language is doing what you told it)
- OmegaDB (it's passing through to CouchDB honestly)
- `page-query` itself (the idiom works fine at small offsets)

## See also

- `omega-knowledge-base/projects/query-omega-oql/gotchas/page-query-skip-performance.md` — the corresponding OQL-level gotcha with an `include_docs` avoidance pattern
- Commit `723b3b2` — `rewrite page-query to use with-skip/with-limit term path`
```

- [ ] **Step 2: Write the JVM-restart subscription gotcha**

Write `~/Development/omega/omega-db/docs/gotchas/event-subscriptions-die-on-jvm-restart.md`:

```markdown
[< Gotchas](README.md)

# Event subscriptions die on JVM restart

## Symptom

A coordinator that was running events off a push-connector subscription stops receiving events after the OmegaDB JVM is restarted. The subscription configuration is still in the database and visually looks fine, but no events fire. Redeploying the coordinator does not help.

## Cause

Subscription registrations are held in JVM memory (in a subscription manager actor), not persisted to disk as live watchers. On startup, the JVM reads persisted subscription records and should re-register them — but if the coordinator was running from a `run-page` session that had already terminated, there's no living session for the subscription to re-attach to, and the subscription silently drops.

In practice, any JVM restart, redeploy, or crash-recovery will kill all running subscriptions.

## Diagnosis

- Last log entry from the coordinator is shortly before the JVM restart timestamp.
- `describe` on the coordinator page shows the subscription config is still present.
- Push-connector continues writing events, but nothing fires.

## Workaround

Re-kick the coordinator with a fresh `run-page` after any JVM restart. Use the push-and-run scratch pattern: push a small `run-coordinator.oql` file that calls `run-page` on the coordinator page with the appropriate start cursor (skip position, last event ID, etc.), then run it once. The subscription re-registers inside the new session.

Do NOT rely on redeploy alone — deploy only updates the page contents, it does not restart the subscription.

## Known open issue

Long-term fix is to make subscriptions robust to JVM restart by re-registering from persisted config without requiring a live session. Tracked in `~/Development/omega/omega-db/` (no issue number yet — gap recorded in `omega-knowledge-base/gaps.md`).

## See also

- `~/Development/wttc/homes-dot-com/README.md` — the HDC sync pipeline where this bit multiple times
- `omega-knowledge-base/projects/query-omega-oql/gotchas/event-subscriptions-die.md` — the OQL-level version of this gotcha
```

- [ ] **Step 3: Update the `gotchas/README.md` index**

Edit `~/Development/omega/omega-db/docs/gotchas/README.md` to replace the stub body with:

```markdown
[< Docs](../README.md)

# Gotchas — OmegaDB

Failure modes and debugging patterns specific to the OmegaDB engine.

## Subscriptions and sessions

- [Event subscriptions die on JVM restart](event-subscriptions-die-on-jvm-restart.md) — why coordinators stop firing after deploys and how to re-kick them

## CouchDB and storage

- [CouchDB skip with `include_docs` is O(skip)](couchdb-skip-include-docs-performance.md) — why paginated scans slow down at high offsets
```

- [ ] **Step 4: Verify**

```bash
ls ~/Development/omega/omega-db/docs/gotchas/
```

Expected: `README.md`, `couchdb-skip-include-docs-performance.md`, `event-subscriptions-die-on-jvm-restart.md`.

- [ ] **Step 5: Commit**

```bash
git -C ~/Development/omega/omega-db add docs/gotchas/
git -C ~/Development/omega/omega-db commit -m "docs: seed omega-db gotchas (jvm-restart subscriptions, couchdb skip perf)"
```

---

# Phase C — Memory prune

## Task C1: Delete doc-shaped feedback files

These five files have content that belongs in the knowledge base or in contributor docs, not in session-start memory. Delete them. Any cross-references from `MEMORY.md` will be fixed in Task C3.

**Files to delete:**
- `~/.claude/projects/-home-ryan-Development-omega-query-omega-oql/memory/feedback_cd_before_qo.md`
- `~/.claude/projects/-home-ryan-Development-omega-query-omega-oql/memory/feedback_deploy_readme.md`
- `~/.claude/projects/-home-ryan-Development-omega-query-omega-oql/memory/feedback_http_request_only.md`
- `~/.claude/projects/-home-ryan-Development-omega-query-omega-oql/memory/feedback_nvmrc_standard.md`
- `~/.claude/projects/-home-ryan-Development-omega-query-omega-oql/memory/feedback_read_docs_before_writing.md`

- [ ] **Step 1: Read each file and confirm its content is preserved elsewhere**

Before deletion, read each file's content (short, usually < 20 lines) and confirm the equivalent is recorded in the knowledge base. If not, pause and migrate first.

```bash
MEM=~/.claude/projects/-home-ryan-Development-omega-query-omega-oql/memory
for f in feedback_cd_before_qo feedback_deploy_readme feedback_http_request_only feedback_nvmrc_standard feedback_read_docs_before_writing; do
  echo "=== $f ==="
  cat "$MEM/$f.md"
done
```

- [ ] **Step 2: Migrate any content that isn't yet in the knowledge base**

For each file whose content has no home in the knowledge base tree, write the equivalent doc. Candidate destinations:

- `feedback_cd_before_qo.md` → `omega-knowledge-base/contributor/gotchas/shell-state-resets-between-bash-calls.md` (if not already present)
- `feedback_deploy_readme.md` → `omega-knowledge-base/contributor/how-to/deploying-a-project.md`
- `feedback_http_request_only.md` → `omega-knowledge-base/projects/query-omega-oql/gotchas/http-post-request-deprecated.md` (new file) — this one is OQL-specific and not covered by the existing Plan 1 gotcha seeds
- `feedback_nvmrc_standard.md` → `omega-knowledge-base/contributor/how-to/setting-up-node-projects.md`
- `feedback_read_docs_before_writing.md` → `omega-knowledge-base/contributor/how-to/navigate-the-docs.md` (already covers this principle — verify, and if so, no new file needed)

For each new file created in the knowledge base, also update the parent `README.md` to index it, and commit the addition as its own commit in `omega-knowledge-base` BEFORE deleting the source memory file. That way, if anything goes wrong mid-delete, the content isn't lost.

- [ ] **Step 3: Delete the five memory files**

```bash
MEM=~/.claude/projects/-home-ryan-Development-omega-query-omega-oql/memory
rm "$MEM/feedback_cd_before_qo.md" \
   "$MEM/feedback_deploy_readme.md" \
   "$MEM/feedback_http_request_only.md" \
   "$MEM/feedback_nvmrc_standard.md" \
   "$MEM/feedback_read_docs_before_writing.md"
```

Memory files are not under git (the memory dir is outside any repo). The deletion is permanent — Task C1 Step 2 already ensured content is preserved.

- [ ] **Step 4: Verify the files are gone**

```bash
ls ~/.claude/projects/-home-ryan-Development-omega-query-omega-oql/memory/ | grep -E 'feedback_(cd_before_qo|deploy_readme|http_request_only|nvmrc_standard|read_docs_before_writing)\.md$'
```

Expected: no output.

---

## Task C2: Resolve remaining orphan memory files

The scope audit surfaced two orphans not explicitly in the spec's delete-or-keep lists: `feedback_workflow.md` and `next_session.md`. Decide each.

- [ ] **Step 1: Read `feedback_workflow.md`**

```bash
cat ~/.claude/projects/-home-ryan-Development-omega-query-omega-oql/memory/feedback_workflow.md
```

Expected content: operational rules ("never run the full pipeline without permission", "don't restart the DB without asking", etc.). This is process feedback that should stay in memory — it's the kind of rule that needs to fire at session-start, not be looked up.

Decision: **keep**. No action.

- [ ] **Step 2: Decide on `next_session.md`**

`next_session.md` is the session handoff file — it's written at the end of one session and read at the start of the next. It's session-state, not docs. It stays in memory.

At the end of Plan 2 execution, this file will need rewriting to reflect the post-Plan-2 state (see Task D2). For now, no action.

- [ ] **Step 3: Sanity-check no other orphans exist**

```bash
ls ~/.claude/projects/-home-ryan-Development-omega-query-omega-oql/memory/*.md | sort
```

Compare against the spec's keep-list and the C1 delete-list. Every file should be either kept, or about to be rewritten in Task C3. Flag any surprise.

---

## Task C3: Rewrite `MEMORY.md` to its pruned form

`MEMORY.md` is the session-start index file. Currently ~95 lines. Target: ~25 lines with a single pointer to the knowledge base and only the load-bearing rules and file pointers.

**Files:**
- Rewrite: `~/.claude/projects/-home-ryan-Development-omega-query-omega-oql/memory/MEMORY.md`

- [ ] **Step 1: Write the new MEMORY.md**

Write `~/.claude/projects/-home-ryan-Development-omega-query-omega-oql/memory/MEMORY.md`:

```markdown
# Workflow Rules

## Knowledge base — start here
Everything non-session-state lives in `~/Development/omega/omega-knowledge-base`. Invoke `omega:omega-docs-navigator` to orient. The navigator establishes the descend-from-root discipline and points to `omega:omega-docs-mark-gap` for when a predicted path is missing.

Before writing code in any project, read its `/docs/README.md` and walk down from there. Never skip the parent chain.

## Hard rules
- **Never run the full pipeline without Ryan's permission.**
- **Never write scratch files for mutations** — use `qo run`, `qo query`, deployed implementations.
- **Never drop/truncate large tables** — pipeline tables take hours to refill.
- **Never use internal datastores in the CLI** — public API endpoints only.
- **Never use boolean `true`/`false`** in OQL — use strings.
- **Never add `Co-Authored-By` trailers** to commit messages.
- **Use `git -C`**, not `cd && git`.
- **Don't restart the DB without asking.**

## Feedback (behavioral rules)
- [No Co-Authored-By](feedback_no_coauthored_by.md)
- [READMEs over code reading](feedback_documentation.md)
- [Auto-run README agent](feedback_readme_agent.md)
- [Workflow rules](feedback_workflow.md)
- [git -C not cd &&](feedback_git_no_cd.md)
- [Memory agent on gaps](feedback_memory_agent.md)
- [Docs taxonomy](feedback_docs_taxonomy.md)
- [Stop holding assumptions](feedback_assumptions.md)
- [Never violate component design](feedback_design_violations.md)
- [Copy reference code exactly](feedback_copy_reference.md)
- [Never drop large databases](feedback_no_delete_data.md)
- [Debug state breadcrumbs](feedback_debug_state_breadcrumbs.md)
- [One change at a time](feedback_one_change_at_a_time.md)

## Project state
- [**Next session handoff**](next_session.md) — **READ FIRST** — state at end of last session
- [Current work](project_current_work.md)
- [Omega overview](project_omega.md)
- [WTTC contract](project_wttc_contract.md)
- [Orphaned data cleanup](project_orphaned_data.md)
- [Property rename gap](project_property_rename_gap.md)

## References (external info)
- [Omega-DB internals](reference_omega_db_internals.md)
- [UI frontend](reference_ui_frontend.md)
```

Line count: ~45 lines. That's slightly above the spec's target of ~25 but preserves the full feedback/project index (compressing further would lose load-bearing pointers). Acceptable.

- [ ] **Step 2: Verify line count**

```bash
wc -l ~/.claude/projects/-home-ryan-Development-omega-query-omega-oql/memory/MEMORY.md
```

Expected: ~45 lines (down from ~95).

- [ ] **Step 3: Verify every link in MEMORY.md resolves to an existing memory file**

```bash
cd ~/.claude/projects/-home-ryan-Development-omega-query-omega-oql/memory && grep -oE '\(([a-z_]+\.md)\)' MEMORY.md | tr -d '()' | while read f; do [[ -e "$f" ]] || echo "missing: $f"; done
```

Expected: no `missing:` lines. If any are reported, either the link is stale (fix MEMORY.md) or the file was deleted in C1 by mistake (restore from another session's backup).

- [ ] **Step 4: No git commit needed**

Memory is not under version control. The rewrite is complete when the file is saved.

---

# Phase D — Smoke test

## Task D1: Invoke the navigator and walk the tree

- [ ] **Step 1: Invoke the navigator skill**

Use the `Skill` tool with `omega:omega-docs-navigator`.

Expected: the skill returns the descend-from-root rules and points at `~/Development/omega/omega-knowledge-base/README.md`.

- [ ] **Step 2: Read the knowledge base root README**

Read `~/Development/omega/omega-knowledge-base/README.md`.

Expected: the root lists the `contributor/` and `projects/` top-level buckets, plus any migrated content references.

- [ ] **Step 3: Descend into omega-docs via the `projects/omega-docs.md` pointer**

Read `~/Development/omega/omega-knowledge-base/projects/omega-docs.md`. Follow its link to `~/Development/omega/omega-docs/README.md`.

Expected: the new 4-bucket README written in Task A7.

- [ ] **Step 4: Walk one path to a leaf**

Pick any path — e.g., root → `projects/omega-docs.md` → `omega-docs/README.md` → `explanation/README.md` → `explanation/oql-execution-model.md`. Read each file in sequence. Confirm each `[< Home]` or `[< Explanation]` link resolves.

- [ ] **Step 5: Walk one path into a project tree**

Root → `projects/omega-db.md` → `omega-db/docs/README.md` → `omega-db/docs/gotchas/README.md` → `omega-db/docs/gotchas/event-subscriptions-die-on-jvm-restart.md`.

- [ ] **Step 6: Confirm MEMORY.md still loads correctly**

The session-start hook loads `MEMORY.md` automatically. Restart the session or invoke manually to verify the rewritten file parses and the navigator guidance is present.

---

## Task D2: Rewrite `next_session.md` for the post-Plan-2 state

- [ ] **Step 1: Overwrite `next_session.md`**

Write `~/.claude/projects/-home-ryan-Development-omega-query-omega-oql/memory/next_session.md` with a new handoff documenting:

- Plan 2 is shipped — omega-docs migrated, per-project trees seeded, memory pruned
- HDC pipeline state (still outstanding from the Plan 1 session — see old `next_session.md` for debug procedure)
- Any uncommitted work left behind (especially the query-omega-oql unrelated modifications from pre-flight)
- Any open gaps recorded in `omega-knowledge-base/gaps.md` that weren't filled during Plan 2

The exact content depends on execution outcome. Draft at execution time, not now.

- [ ] **Step 2: No commit needed**

Memory is not under version control.

---

## Final verification

- [ ] **Step 1: Verify no orphaned stale paths in omega-docs**

```bash
cd ~/Development/omega/omega-docs && find . -type d -not -path './.git*' -not -path './site*' -not -path './docs*' | sort
```

Expected directories: `.`, `./explanation`, `./gotchas`, `./how-to`, `./pricing`, `./reference`, `./tools`. No `omegadb`, no `omega-ai`, no `architecture`, no `guides`, no `intro.md`/`philosophy.md` at root.

- [ ] **Step 2: Verify every project has a seeded tree**

```bash
for p in omega-db query-omega-oql query-omega-api query-omega-cli query-omega-mcp omega-cli omega-ai-components query-omega query-omega-website; do
  test -f ~/Development/omega/$p/docs/README.md && echo "OK: $p" || echo "MISSING: $p"
done
test -f ~/Development/wttc/homes-dot-com/docs/README.md && echo "OK: homes-dot-com" || echo "MISSING: homes-dot-com"
test -f ~/Development/wttc/gohighlevel/docs/README.md && echo "OK: gohighlevel" || echo "MISSING: gohighlevel"
```

Expected: 11 `OK:` lines, no `MISSING:` lines.

- [ ] **Step 3: Verify memory is pruned**

```bash
ls ~/.claude/projects/-home-ryan-Development-omega-query-omega-oql/memory/ | wc -l
wc -l ~/.claude/projects/-home-ryan-Development-omega-query-omega-oql/memory/MEMORY.md
```

Expected: file count dropped by 5 (from 27 to 22), `MEMORY.md` line count ~45.

- [ ] **Step 4: Verify every project commit is on its branch**

```bash
for p in ~/Development/omega/omega-db ~/Development/omega/query-omega-oql ~/Development/omega/query-omega-api ~/Development/omega/query-omega-cli ~/Development/omega/query-omega-mcp ~/Development/omega/omega-cli ~/Development/omega/omega-ai-components ~/Development/omega/query-omega ~/Development/omega/query-omega-website ~/Development/wttc/homes-dot-com ~/Development/wttc/gohighlevel ~/Development/omega/omega-docs ~/Development/omega/omega-knowledge-base; do
  echo "=== $(basename $p) ==="
  git -C "$p" log --oneline -5 | head -5
done
```

Expected: each repo's most recent commit is a Plan 2 `docs:` or `chore:` / `scripts:` commit from this session.

---

## Self-review checklist

- [x] Every task has exact file paths
- [x] Every move operation uses `git -C <repo>` instead of `cd && git`
- [x] No placeholder text ("TBD", "add error handling", "similar to task N")
- [x] Every task ends with a verification step and a commit (except memory, which isn't versioned)
- [x] No `Co-Authored-By` trailers in any commit message example
- [x] The uncommitted state in `query-omega-oql` is explicitly handled in Task B4
- [x] All 11 projects accounted for in Phase B
- [x] All classifications from the scope audit have been walked into explicit `git mv` commands
- [x] The `docs/superpowers/` subtrees in omega-docs, query-omega-cli, query-omega-oql, and query-omega-website are explicitly protected

## Known gaps this plan leaves for future sessions

1. **`how-to/` bucket is empty in omega-docs** — no task recipes migrated because none exist in the current repo. The bucket is stubbed with candidate topics; real content comes from fill-gap invocations as gaps are recorded.
2. **Project `how-to/`, `reference/`, and `explanation/` buckets are empty stubs** in every project except omega-db and query-omega-cli. Filling them is async via fill-gap.
3. **No rebalance** in this plan — the omega-docs `reference/` bucket ends up with 8 entries, one over the ~7 threshold. Promoting `reference/` into `reference/{language,api,tooling}` sub-buckets is a followup once one more entry lands.
4. **The query-omega-oql subtree inside the knowledge base** (`omega-knowledge-base/projects/query-omega-oql/`) remains separate from `query-omega-oql/docs/` in the repo itself. They serve different purposes — the knowledge-base subtree is the private index with 9 already-seeded gotchas, the repo tree is greenfield for future project-local content. No consolidation planned.
5. **No migration of `feedback_workflow.md`** — it stays in memory. If the knowledge base grows a `contributor/how-to/omega-workflow-rules.md` that duplicates this content, remove from memory at that point.

---

## Execution handoff

Plan complete and saved to `docs/superpowers/plans/2026-04-12-omega-knowledge-base-plan2-migration.md`. Two execution options:

**1. Subagent-Driven (recommended)** — dispatch a fresh subagent per task, review between tasks, fast iteration. Good for this plan because most tasks are mechanical and verification is visible in git log.

**2. Inline Execution** — execute tasks in the current session using `superpowers:executing-plans`, with review checkpoints between phases (A, B, C, D).

Given the size (~15 tasks across 13 repos) and the mechanical nature of most of the work, Subagent-Driven is the right call. Dispatch one subagent per task in Phase A sequentially, then parallelize the greenfield seeds in Phase B Task B2 (the 9 project seeds are fully independent), then serialize again for C and D.
