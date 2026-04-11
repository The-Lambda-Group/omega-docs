# HDC Docs Hygiene — Design Spec

**Date:** 2026-04-10
**Status:** Execute
**Scope:** Documentation only — no runtime or OQL logic changes

## Problem

Multiple documentation files drifted during a debug session that left uncommitted work scattered across three repos. The work is:

- Uncommitted "Known issues" section in `wttc/homes-dot-com/README.md` that mixes HDC-private details with generic GHL-dedup explanations.
- Uncommitted Query Library and App Connector Roadmap sections in `wttc/gohighlevel/README.md` that had HDC-tied priority rationale written into a doc that should read like a public SDK.
- Platform docs in `omega-docs/omega-ai/plugins.md` and `omega-docs/philosophy.md` that name "Homes.com Connector" as the canonical plugin example, even though HDC is a one-shot throwaway internal pipeline that will likely never run again after this project.
- Debug-state impl file with a thin `DEBUG:` breadcrumb comment that failed to survive a cold-start session (triggered a full re-investigation of an already-understood bug).
- Memory (`project_current_work.md`) is stale — still describes the marketing site rebuild as current focus with a half-finished note about a GHL sync that was "running at ~130/min."

The root issue: content is in the wrong layer. Platform-level docs reference a throwaway project. A potentially-public SDK doc encodes one specific consumer's needs. A private project doc re-explains generic CRM API semantics that belong upstream. Debug-state breadcrumbs don't carry enough context to survive a fresh session.

## Non-goals

Explicit list of things this spec does NOT address:

- Fixing the GHL Sync failure (deferred — probably just "tolerate 400s," but that's implementation, not docs)
- Building any query on the GHL connector
- Building the HDC enrichment pipeline step (roadmap only)
- Designing the replication/App-Connector migration (roadmap only, already sketched in the GHL README)
- Committing anything — every edit stays uncommitted for Ryan to review
- Reverting the impl file to a pristine state
- Mirroring customer-requirement data into a team-visible doc outside this machine

## Three-layer taxonomy

After this spec lands, content lives in layers as follows:

**1. Platform docs (`omega-docs`)** — abstract. Uses hypothetical / generic plugin examples, not real project names. Holds canonical definitions of: plugins, components, push connectors, protocols, lifecycle, event handling, App Connector pattern. No mention of HDC, the homes.com project, or any throwaway internal work. Examples in plugins.md and philosophy.md will use a "Payments Connector" example instead of naming HDC.

**2. GHL connector (`wttc/gohighlevel/README.md`)** — SDK. Reads like a third-party library for GoHighLevel. Completely caller-agnostic. Eventually public. No references to HDC, homes.com, WTTC, customer requirements, or any specific consumer. Query Library Roadmap describes what a production GHL SDK generically needs, organized by priority based on "what CRM integrations generally require," not "what this one consumer needs right now." Priorities reset: `get-contact` and `update-contact` stay P0 (core read + write), `upsert-contact` drops to P1 (convenience over core), `ensure-custom-field` is P2 (rare admin task). App Connector Roadmap stays as-is structurally but any HDC residue in rationale gets scrubbed.

**3. HDC project (`wttc/homes-dot-com/README.md`)** — a project journal for one specific throwaway pipeline. Holds the top-10% framing, customer requirements, sales data universe context (300k / 1.5M / 3.2M), the enrichment pipeline roadmap, the manual GHL custom field setup as a documented prereq, the current sync halt state with a lightweight "probably just tolerate 400s" fix note, and the debug state pointer. Nothing in this doc is reusable platform content. Nothing in here needs to live anywhere else.

The impl file gets a much more thorough DEBUG breadcrumb comment so a cold-start session has full context without re-investigation. Per `feedback_debug_state_breadcrumbs.md`.

Memory (`project_current_work.md`) gets an updated section reflecting actual current focus.

## HDC project-level framing (new content going into homes-dot-com README)

Brand new content for the HDC README, front-loaded so readers see *why* before *what*. This is the content the customer cares about; the existing operational "Pipeline/Schema/Setup/Running" sections stay put below it.

- **Goal:** Identify the top 10% of real estate agents in the United States.
- **Data source:** Homes.com is the only source we can access without pulling in a third-party broker. Our scraped dataset is ~300k agents.
- **Target universe:** There are an estimated ~1.5M full-time real estate agents in the US (and ~3.2M total licensed). Our 300k sample is a subset — not a uniform random sample, probably biased toward the more active agents (they have more listings on homes.com to scrape). Realistically our "top 150k" from this data is probably top 10–50% in the real market, not top 10% of the full 1.5M universe.
- **Scope:** One-shot internal task. Once the data is delivered to the customer, this pipeline will likely never run again.
- **Customer requirements:**
  - Synced contacts must carry social media links (LinkedIn, Facebook, Twitter, Pinterest — the four fields present in the Homes.com Data schema).
  - Synced contacts must carry sales performance data (score, closed sales, total value) so the customer can filter/qualify agents in GHL themselves.
- **Current state:** Initial create-pass of the pipeline ran through ~94k of the 150k target before halting on a 400-dupe from GHL. The halt is documented in "Known issues" with a lightweight fix note. A planned enrichment pipeline step will backfill social + sales fields onto the already-created contacts via GHL custom fields.

## Enrichment pipeline roadmap (new roadmap subsection in HDC README)

This is documented as a planned pipeline step, not implemented. Lives in the HDC README under the Pipeline section.

- **Purpose:** Populate GHL contacts with social media links and sales performance data as custom fields. Runs after the initial create-pass (GHL Sync) completes or is declared done.
- **Step name:** `Enrich GHL Contacts` (or similar — name TBD at implementation time).
- **Input:** Existing GHL Contacts ledger + Homes.com Data raw table (for social + sales fields).
- **Output:** Updates in GHL API, tracked in a new `Data/Enriched Contacts` ledger table or via updates to the existing `GHL Contacts` table (TBD at implementation time).
- **Dependency:** GHL custom fields must exist in the location before this step runs. Setup section will document that manually creating these fields in the GHL UI is a prerequisite — not a responsibility of this pipeline's installer.
- **Batch strategy:** Same Dagster-style batched event dispatch as the other steps. Reads Filtered Agents in batches, looks up both social and sales fields from Scored Agents / Homes.com Data, builds a PATCH payload, calls the GHL update endpoint, tracks progress in a ledger table.
- **Design decisions deferred to implementation session:** whether to fetch custom field IDs every batch vs cache them (leaning fetch-every-batch for simplicity), how to handle partial failures, whether to use a new GHL connector query or open-code the PATCH call in the HDC impl, what happens if a ledger row exists but the GHL contact was deleted externally.

The GHL custom field setup gets documented in HDC's Setup → Prerequisites as manual work: "Before running the enrichment step, create the following custom fields in the GHL UI: `linkedin_url` (TEXT), `facebook_url` (TEXT), `twitter_url` (TEXT), `pinterest_url` (TEXT), `score` (TEXT or NUMERICAL), `closed_sales` (TEXT or NUMERICAL), `total_value` (TEXT or NUMERICAL)." No query-based provisioning is planned.

## Known issues note (rewrite — lighter than the current version)

The current "Known issues" section is too heavy. It explains GHL's dedup model in detail and cross-references upsert-contact as a P0 unblock — that framing is wrong because (a) upsert-contact is no longer P0, and (b) the real fix is probably much simpler than a new query. Replacement framing:

- The sync halted at a 400-dupe from GHL. Source of the dupe is unknown — might be a pre-existing GHL contact from another system, might be an internal dupe in our source data.
- The fix is probably very simple: tolerate 400 responses, log them as a status, and move on without writing to the ledger. Or optionally, pull the existing GHL contact ID from the 400 response body and write *that* to the ledger. Both are small tweaks to the impl's parse branch.
- Not worth over-designing. We're ~94k of ~150k through and only just hit our first dupe, so this is probably rare.
- Impl is currently in a debug state with stubbed response parse — see the `DEBUG:` comment block in the impl file for details.
- Duplicate risk from the debug session: one row (`skip=94200`) was re-submitted to GHL and rejected both times by server-side dedupe. No new orphan rows were created.

## Debug state breadcrumb (impl file comment expansion)

Per `feedback_debug_state_breadcrumbs.md`: the existing `DEBUG:` comment is too thin. Replace with a full breadcrumb that records who, when, why, what the stubs are, where to read for more context, and the re-enable condition. No OQL logic changes. Only comment lines touched.

## GHL connector edits in detail

All edits to `wttc/gohighlevel/README.md`:

1. **Scrub HDC tag example values.** The doc's usage examples for `list-contacts`, `delete-contacts` use `"tag": "homes-dot-com"` as the example tag. Replace with a generic example tag (e.g., `"tag": "my-plugin"` or `"tag": "example-segment"`).
2. **Priority reset in Contacts roadmap table.**
   - `get-contact` — stays **P0**
   - `get-contact-by-email` — demoted to **P1**
   - `get-contact-by-phone` — stays **P1**
   - `search-contacts` — stays **P1**
   - `upsert-contact` — demoted to **P1**, description rewritten to remove "main usability win" language
   - `update-contact` — stays **P0** (core write)
   - `delete-contact` — stays **P1**
   - `add-tags` / `remove-tags` — stay **P2**
   - `list-contacts-paginated` — stays **P1**
3. **Custom Fields roadmap:** add `ensure-custom-field` at **P2** with generic "admin convenience" framing, not HDC-tied.
4. **Cross-cutting concerns paragraph:** remove the "consumer components can share dedup/error handling" phrasing, replace with SDK-generic language.
5. **App Connector Roadmap — "Full installs scale the way homes.com does"** line (currently in the Lifecycle subsection) — rewrite in generic terms. That's the only HDC name-drop in the App Connector Roadmap section.
6. **Migration path step 6** mentions "any consumer that currently POSTs directly to GHL" — that's acceptable generic language. No change needed.

## Platform docs edits in detail

All edits to `omega-docs/omega-ai/plugins.md`:

1. Replace the example page tree block that names Homes.com Connector with a "Payments Connector" equivalent showing the same plugin shape.
2. Replace the example `qo` commands that use `Component Installs/Homes.com Connector` paths with Payments Connector paths.
3. Replace the Unix philosophy bullet "The Homes.com Connector scrapes agents, scores them, and syncs to a CRM" with an abstract "A payments connector receives webhooks, reconciles balances, and reports deltas."

Edit to `omega-docs/philosophy.md`:

1. Same Unix philosophy bullet (line ~91) gets the same Payments Connector treatment.

## Memory update

`project_current_work.md` — replace the "Background: GHL sync" paragraph with an accurate current-state summary that reflects: the sync is halted at ~94k, HDC is a one-shot throwaway task, the docs hygiene pass was completed, the enrichment pipeline is roadmapped but not built. Keep the marketing site / tools / philosophy sections intact since those are still-valid session state.

## Risk / rollback

Rollback is trivial: `git diff` across affected repos shows every edit; `git checkout` any file returns it to committed state. Memory file is not version-controlled but is a simple text file and can be restored from prior sessions if needed. Nothing touches runtime state, nothing deploys, nothing affects pipelines. The impl file diff should be comment-only, verifiable via `grep '^[+-]' | grep -v '^[+-].*;;'` returning zero lines.

## Files touched

1. `omega/omega-docs/omega-ai/plugins.md`
2. `omega/omega-docs/philosophy.md`
3. `wttc/gohighlevel/README.md`
4. `wttc/homes-dot-com/README.md`
5. `wttc/homes-dot-com/impl/18f2e4bb-homes-com-component-library/a31e2741-events-library/8e5821d0-ghl-sync/bfe95f5a-implementation.oql` (comments only)
6. `~/.claude/projects/-home-ryan-Development-omega-query-omega-oql/memory/project_current_work.md`
7. `omega/omega-docs/docs/superpowers/specs/2026-04-10-hdc-docs-hygiene-design.md` (this file)

Nothing committed. Ryan reviews, then decides what to commit and what to revert (notably: the impl file debug state may get reverted before commit).
