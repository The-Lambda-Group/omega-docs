# Omega Knowledge Base — Design

**Date:** 2026-04-12
**Status:** Approved for implementation planning
**Author:** Ryan + Claude (brainstorm session)

## Problem

The current contributor-facing documentation system is fragmented and unreliable for agent use:

- **Memory is bloated.** 34 files, 2,262 lines, ~200KB. ~40% is strategic reports and design docs that don't belong in session-start memory. Feedback rules compete for attention with reference material.
- **Session-start reliability is low.** Rules like "read the project README before writing code" exist in memory but get forgotten after ~20 turns of other work. They're behavioral, not mechanical, so they drift under pressure.
- **Doc coverage is patchy and unpredictable.** `omega-docs` covers some OQL concepts but not others. Each project has its own README with no common shape — some have deploy instructions, some don't, some document known gotchas, most don't.
- **Agents burn tokens re-reading code** to answer questions that should live in docs. Every code-read for "how does this work" or "what's the deploy process" is a missed doc entry.
- **No mechanism captures what's missing.** When an agent can't find something in the docs, the gap is silently absorbed and the docs never improve.

The current session burned several hours on OQL gotchas (silent row drops from `with-table-if` schema collapse, `(defined X)` being effectively universal, mixed-literal symbol resolution failing in `throw` calls) that could have been one-shot lookups if the docs had been structured to be findable.

## Goal

A **contributor knowledge base** that lets an agent answer most questions without reading code, with a structure so regular the agent can predict the path to the answer before reading anything.

Concretely:

- **Predictable path grammar** — the agent can translate a question into a predicted path using a mechanical rule, without search.
- **Descend-from-root discipline** — the agent always walks the parent chain so the answer is carried with its context.
- **Gap capture and async filling** — when the prediction fails, the gap is recorded automatically, and a fill-gap agent writes the missing doc in the background.
- **Self-similar tree** — every node in the tree has the same shape, so the descent algorithm is the same at every depth.
- **Three-skill discipline** — orientation, gap marking, and gap filling are distinct skills with clear responsibilities.
- **Mechanical session-start enforcement** — a SessionStart hook ensures the navigator skill fires on every Omega-context session.

## Non-goals

- **No tutorials bucket.** Agents don't need onboarding walkthroughs; `how-to/` covers task recipes.
- **No rebalance log.** Fill-gap agents judge rebalancing at write time and handle it inline.
- **No GitHub issue integration.** Gap capture is entirely automatic and internal to the knowledge base.
- **No CLAUDE.md per project.** If the skills and hook work correctly, project-level CLAUDE.md files become redundant. We delete them after the navigator skill is in place.
- **No migration of `omega-docs` in this project.** The `omega-docs` Diátaxis restructure is a follow-on spec. This project ships `omega-knowledge-base` with a pointer to `omega-docs` as it currently is.

## Architecture overview

The knowledge base is a tree whose root is a new private repo (`omega-knowledge-base`) and whose leaves span multiple repos.

Three kinds of nodes:

1. **Root nodes** — live in `omega-knowledge-base`. Top-level index, contributor workflow docs, gap queue, private content that can't live in public `omega-docs`.
2. **Project-root nodes** — live in each project repo's own `/docs/` folder. Every project (public or private) has one. All follow the same template.
3. **Public concept nodes** — live in `omega-docs`. OQL language, execution model, public API reference. Unchanged in this project (migration is follow-on).

The root repo stitches these into a single logical tree by linking out to each project's `/docs/README.md` and to `omega-docs`. An agent starting at the root can descend into any of them by following links.

**Why three physical locations:**

- `omega-docs` has to stay public-clean. It can't reference private work like HDC or GHL connectors.
- Project-local docs live with the code they document, so they move together.
- `omega-knowledge-base` is the private index that binds them — it knows about every project and has its own private-only content.

The agent doesn't need to know which physical repo a path is in — it navigates by following links from parents to children.

## Taxonomy — Diátaxis + gotchas

Every directory in the tree has the same four-bucket children:

```
<any-node>/
  README.md        ← index listing every child with a one-line description
  how-to/          ← task recipes, workflows, deploy steps (merges tutorials)
  reference/       ← API, schemas, IDs, protocol signatures, config
  explanation/     ← concepts, architecture, design rationale, the "why"
  gotchas/         ← failure modes, footguns, debugging patterns
```

This is **Diátaxis** (diataxis.fr) with two modifications:

1. **`tutorials/` merged into `how-to/`.** Agents don't need guided onboarding walkthroughs — they need task recipes. Humans who want onboarding can read the project README.
2. **`gotchas/` added as a fourth bucket.** Failure-mode knowledge is distinct from `explanation/` (which is about design) and distinct from `reference/` (which is factual lookup). Gotchas are diagnostic and narrative — "if you hit X, the cause is usually Y."

Why these four specifically:

- **Concepts vs guides is not optional.** People ask both "what is this" and "how do I" and they're different reading modes.
- **Reference vs concepts is not optional.** You want to look up a protocol ID without reading an essay about the protocol system.
- **Gotchas vs explanation is not optional.** A debugging session reads gotchas very differently from how it reads explanation.

### Why Diátaxis specifically

- Most widely-adopted technical doc framework in the open-source world.
- Used by Django, NumPy, Cloudflare, Divio, GitLab.
- Industry vocabulary — "look it up in the reference docs" has a universal meaning.
- Clean underlying theory: two axes (theoretical vs practical, acquisition vs application).

### Rules for self-similarity

1. **Every directory has the four buckets as its children.** Plus possibly top-level container directories (`projects/`, `contributor/`) at the root.
2. **Every directory has a `README.md`** that indexes its children with one-line descriptions sufficient for a descend-or-skip decision.
3. **A file lives at the deepest level where it still fits a single axis.** If a file is both concept and guide, split it.
4. **Recursion happens inside any bucket when it grows.** So `guides/` can have flat files or a nested tree of topic-guides each with their own four sub-buckets.
5. **Cross-references by relative path.** Never absolute URLs. Navigable offline, resilient to repo moves.
6. **Lowercase kebab-case paths.** Regex-friendly, glob-friendly.

## Rebalancing

**Any leaf can become a subtree with the same four buckets inside it.** This is the rebalancing primitive.

### When to promote (split out)

- A bucket has **more than ~7 entries** (Miller's number — the signal to consider promotion).
- A single file has **grown beyond a screen or two** and would benefit from axis separation.
- A topic has **high gotcha surface** — when you find yourself writing the third gotcha about a topic, promote so gotchas get their own bucket inside the new subtree.

### When to consolidate (roll up)

- A subtree's `how-to/` has 1-2 entries and no other bucket is populated — the subtree was over-promoted, pull it back.
- Multiple sibling files share 80%+ of their content with only small variations — combine into one file with per-variant sections.
- A topic turned out to have less gotcha surface than expected and its `gotchas/` bucket is empty — consider flattening.

### Promotion example

Before (flat):

```
omega-db/docs/how-to/deploy.md      ← growing past one screen
```

After (promoted):

```
omega-db/docs/how-to/deploy/
  README.md
  how-to/
    deploy-to-staging.md
    deploy-to-production.md
    rollback-from-staging.md
    rollback-from-production.md
  reference/
    ci-env-vars.md
  explanation/
    deploy-pipeline-shape.md
  gotchas/
    deploy-during-sync-kills-pipeline.md
```

### Rebalance operation

1. Detect imbalance.
2. Propose in structured form: `{action, source_path, target_path, files_affected, justification}`.
3. Execute: move/rewrite files, update parent READMEs, **walk every `.md` in the knowledge base looking for stale links to the old path and rewrite them**, remove any resolved gap entries.
4. Commit as one atomic commit: `rebalance: promote how-to/deploy to subtree (N files)`.

### Rebalance safety rails

- **Promote freely on agent judgment** when a bucket exceeds 7 entries and one file is clearly the biggest. Low risk.
- **Consolidate cautiously** — ask user to confirm any consolidation that removes a file unless the files are clearly redundant.
- **Never consolidate across different axis buckets.** Merging a `how-to/` file with an `explanation/` file is a taxonomy violation, not a consolidation.

Fill-gap agents are responsible for judging rebalance triggers at write time and handling them inline. There is **no rebalance log** — rebalancing is just another kind of doc work, committed atomically like any other.

## Path prediction algorithm

**The mechanical rule:** translate a question into a predicted path using a fixed algorithm.

```
Given a question Q:
  1. Identify the SCOPE — which project is this about?
     (If unclear, start at knowledge-base root.)
  2. Identify the AXIS — is this how-to / reference / explanation / gotcha?
  3. Identify the TOPIC — what noun or verb phrase is Q about?
  4. Construct the predicted path: <scope>/<axis>/<topic>.md
  5. Walk from knowledge-base root down to the prediction, reading each
     README.md on the way so the context of the answer is carried down.
  6. If the path resolves → read the target file. Done.
  7. If the path is missing → mark a gap and either:
     (a) read the sibling README to see if the answer is under a
         different topic name
     (b) fall back to grep / code as a last resort (and also mark a gap)
```

### Axis decision rule

The agent asks one question about Q:

| Question-word in Q | Axis |
|---|---|
| "How do I…" / "How should I…" | `how-to/` |
| "What is the…" (definition) / "Where is the X ID" / "What's the schema for…" | `reference/` |
| "Why does…" / "How does … work" / "What's the design of…" | `explanation/` |
| "Why did that fail" / "Why won't X" / "What happened when I…" | `gotchas/` |

Two edge cases:

- **"How does X work"** is `explanation/`, not `how-to/`. If the answer is "here's the mechanism," it's explanation. If the answer involves running commands, it's how-to.
- **"Why does this bug happen"** is `gotchas/`, not `explanation/`. Gotchas are "things that bit us." Explanation is "here's the design."

### Descend-from-root — non-negotiable

The agent **never** jumps directly to a leaf. It always starts at `omega-knowledge-base/README.md`, reads the project listing, picks the project, reads the project's `README.md`, picks the axis, reads the axis `README.md`, picks the topic, then reads the topic file.

Why:

- **Parent READMEs carry context.** "This project uses the old protocol" in the project README changes how a leaf file should be read. Skip parents and you misread leaves.
- **Gap detection depends on it.** If the agent can grep around the tree, it'll accidentally find the answer even when the path was wrong, and the gap never gets flagged.
- **It's auditable.** Anyone can check "did the agent read each parent README before reading the child?"

### When prediction is wrong

Two failure modes:

1. **Path doesn't exist.** Mark a gap with the predicted path, question, and parent chain walked. Fall back to sibling README or grep.
2. **Path exists but content is missing/stub/obsolete.** Mark a gap for content, not path. Fall back to code.

Fallback to code is **always the last resort** and **always accompanied by a gap marker**. Every code-read for a question that should be answered by docs is a signal that the docs have a gap.

## The three skills

All three live in an `omega:` plugin/namespace.

### `omega:omega-docs-navigator`

**Purpose:** Orient the agent in the knowledge base tree at session start or on demand.

**What it does when invoked:**

1. **Reads `omega-knowledge-base/README.md`** — the tree root. Populates context with project list, taxonomy, link to contributor rules.
2. **Presents the navigation rules** — the descend-from-root algorithm above, verbatim. The agent reads and commits to it for the session.
3. **Announces availability of the other two skills** so the agent knows to invoke them.

**What it does NOT do:**

- Doesn't read every project's README. That's what descent is for.
- Doesn't preload axis descriptions beyond what's in the root README.
- Doesn't fetch any specific doc — it's orientation, not lookup.

**Invocation:**

- Auto-invoked by SessionStart hook when the session is in an Omega-ecosystem context.
- Manually invoked when user says "navigate the docs" or wants to reset agent context.

### `omega:omega-docs-mark-gap`

**Purpose:** Record that the agent tried to find something in the docs and failed. Spawn fill-gap agents.

**Inputs (batch — 1 to N gaps in a single invocation):**

Each gap entry has:

- `question` — the original question, paraphrased
- `predicted_path` — the path the agent tried
- `parent_chain` — list of parent READMEs walked (required, validated)
- `fallback_source` — what the agent read instead

**What it does:**

1. For each gap in the batch, generates a unique key: `hash(predicted_path + question)[:8]`. Stable — duplicate gap attempts collide to the same key and don't accumulate.
2. **Refuses to accept a gap without a full parent chain.** If the parent_chain is shorter than the predicted_path's depth, mark-gap returns an error asking the agent to complete the descent first. This is the consistency enforcement mechanism.
3. **Appends each accepted gap to `omega-knowledge-base/gaps.md`** in the structured format specified below.
4. **Spawns fill-gap agents based on batch size:**
   - 1 gap → 1 agent
   - 2-3 gaps → 1-2 agents (sequential usually fine)
   - 4-5 gaps → 2-3 agents in parallel
   - 6+ gaps → cap at 3 parallel agents; remaining gaps processed as earlier ones finish
5. **Each agent is assigned specific gap keys**, not "pick from the queue" — prevents races.
6. Agents run in the background via the Agent tool's `run_in_background` parameter.

### `omega:omega-docs-fill-gap`

**Purpose:** Read from `gaps.md`, pick a gap, research the answer, write the doc, clean up the entry. Runs as a background agent.

**What it does:**

1. **Reads `omega-knowledge-base/gaps.md`** — sees open gaps.
2. **Picks a gap** — either one specified by caller via `gap_key` arg, or the oldest open one.
3. **Marks it `claimed`** — atomic commit to prevent races with other agents.
4. **Does the research:**
   - Reads the parent chain that mark-gap recorded (to rebuild the context at failure point)
   - Reads the fallback sources (code, commits, etc.)
   - Reads sibling docs in the predicted path's parent directory for style consistency
5. **Writes the missing doc** at the predicted path — or at a path that better fits the taxonomy if the prediction was structurally wrong. If the latter, also updates the parent README so the new doc is indexed.
6. **Checks for rebalancing triggers.** If adding this doc pushes a bucket over ~7 entries, the agent auto-rebalances (if confident) or records it as a follow-up gap.
7. **Deletes the gap entry from `gaps.md`** — resolved gaps are not marked, they are removed. History lives in git.
8. **Commits atomically:** new doc + README update + gap cleanup in one commit with message `docs: fill gap <hash> — <topic>`.

**Safety rails:**

- Never writes outside the knowledge base or referenced project `/docs/` trees. No code changes.
- Never removes a gap without writing the corresponding doc — partial fills disallowed.
- Always commits atomically.
- Bails with an error if it can't find the answer in the fallback source. Better to leave the gap open than guess.

### Concurrency

- Multiple fill-gap agents may run in parallel with their own assigned keys.
- Each agent marks its gap `claimed` via an atomic commit before work begins.
- If two agents race on the same key, the second agent's claim commit fails on the already-claimed status check and it bails out as a no-op.
- Worst case: N agents all working the same gap produces one good doc and N-1 no-op exits. Acceptable.

## gaps.md format

One file. Markdown. Entries separated by horizontal rules. Each entry has a unique key.

### Entry format

```markdown
## gap-<8-char-hash>

- **question**: <paraphrased question that triggered the lookup>
- **predicted-path**: <path from knowledge-base root, no leading slash>
- **parent-chain**:
  - <path-1>
  - <path-2>
  - <path-3>
- **fallback-source**: <what was read instead — file path, command, or "none">
- **recorded-at**: <ISO 8601 UTC timestamp>
- **status**: open | claimed | resolved
- **claimed-by**: <agent id or empty>
- **notes**: <optional free text>

---
```

### Status transitions

- `open` — recorded, not being worked on
- `claimed` — a fill-gap agent has picked it up (agent writes `claimed-by`)
- `resolved` — deleted from gaps.md (not marked)

**Why delete on resolution:** gaps.md is a work queue, not a history. Once a gap is filled, its existence is implied by the doc that now exists. Full history is in `git log -- gaps.md`.

### Example

```markdown
# Documentation Gaps

Entries are work items for `omega:omega-docs-fill-gap`. Each gap has a
stable hash key. Resolved gaps are deleted, not marked.

---

## gap-a4f3c2d1

- **question**: how do I redeploy omega-db after the JVM subscription died
- **predicted-path**: projects/omega-db/how-to/redeploy.md
- **parent-chain**:
  - README.md
  - projects/omega-db/README.md
  - projects/omega-db/how-to/README.md
- **fallback-source**: omega-db/docker/omegadb.Dockerfile, git log --grep=deploy
- **recorded-at**: 2026-04-12T03:14:00Z
- **status**: open
- **claimed-by**:
- **notes**:

---
```

## File locations

### `omega-knowledge-base` repo (new, private)

Initial structure:

```
omega-knowledge-base/
  README.md                          ← root index; loaded by navigator skill
  gaps.md                            ← rolling gap queue (git-tracked)
  contributor/
    README.md
    how-to/
      README.md
      navigate-the-docs.md           ← the descend-from-root discipline
      mark-a-gap.md                  ← how to invoke the mark-gap skill
      rebalance-a-subtree.md         ← how fill-gap agents rebalance
    reference/
      README.md
      diataxis.md                    ← the 4-axis taxonomy explained
      path-prediction-rules.md       ← the algorithm
    explanation/
      README.md
      why-diataxis-plus-gotchas.md
      why-descend-from-root.md
    gotchas/
      README.md
  projects/
    README.md                        ← lists every project with a description
    omega-docs.md                    ← each project is a .md pointer entry
    omega-db.md
    query-omega-oql.md
    query-omega-cli.md
    query-omega-api.md
    query-omega-mcp.md
    omega-cli.md
    omega-ai-components.md
    homes-dot-com.md
    gohighlevel.md
```

**Note:** `contributor/` is a top-level bucket (alongside `projects/`), not a project entry. Contributor content is meta — it describes how the knowledge base itself works — and shouldn't fit the "project = external repo" shape.

Each `projects/<name>.md` file contains:

- Short description of the project
- Link to the project repo
- Link to the project's `/docs/README.md` if it has one
- Any private notes that can't live in public `omega-docs`

### Per-project docs (inside each project repo)

Every project grows a `/docs/` tree that follows the same Diátaxis+gotchas taxonomy:

```
<project>/docs/
  README.md
  how-to/
    README.md
    ...
  reference/
    README.md
    ...
  explanation/
    README.md
    ...
  gotchas/
    README.md
    ...
```

### The three skills

```
~/.claude/skills/omega/
  omega-docs-navigator/
    SKILL.md
  omega-docs-mark-gap/
    SKILL.md
  omega-docs-fill-gap/
    SKILL.md
```

### SessionStart hook

Added to `~/.claude/settings.json`:

```jsonc
{
  "hooks": {
    "SessionStart": [
      {
        "matcher": ".*",
        "hooks": [
          {
            "type": "command",
            "command": "~/.claude/hooks/omega-session-start.sh"
          }
        ]
      }
    ]
  }
}
```

The `omega-session-start.sh` script checks if the session is in an Omega-ecosystem context (cwd matches `~/Development/omega/` or `~/Development/wttc/`, or the recent prompt mentions Omega-ecosystem keywords) and, if so, emits a SessionStart additionalContext reminder to invoke the navigator skill.

```bash
#!/usr/bin/env bash
cwd="$(pwd)"
if [[ "$cwd" == *"/Development/omega/"* ]] || [[ "$cwd" == *"/Development/wttc/"* ]]; then
  cat <<'EOF'
{
  "hookSpecificOutput": {
    "hookEventName": "SessionStart",
    "additionalContext": "Before answering any question about Omega, OQL, or any project in the Omega ecosystem, invoke the omega:omega-docs-navigator skill. It will orient you in the knowledge base tree at ~/Development/omega/omega-knowledge-base and establish the descend-from-root navigation rules for this session."
  }
}
EOF
fi
```

The hook output appears as the first thing in the agent's context so the navigator skill fires before any other work.

## Memory pruning

The memory pruning is a side effect of this project, not a separate project. Once the knowledge base is in place:

1. Delete every memory file whose content has a home in the knowledge base.
2. Keep only memory files that are genuinely session-state: user preferences, in-flight work, process rules that can't live as docs.
3. MEMORY.md shrinks from ~95 lines to ~25 — just the hard rules and a single pointer: "Everything else is in `omega-knowledge-base`. Invoke `omega:omega-docs-navigator` to orient."

**What gets deleted from memory:**

- All 4 `report_*.md` files (strategic analysis, not rules)
- `design_copy_generation_system.md` (strategic analysis)
- All pointer-only `project_*.md` files (`project_homes_dot_com.md`, `project_component_patterns.md`, `project_oql_conventions.md`)
- Feedback files whose content is really doc-shaped (`feedback_http_request_only`, `feedback_cd_before_qo`, `feedback_nvmrc_standard`, `feedback_deploy_readme`) — migrated to `omega-knowledge-base/contributor/reference/` or the appropriate project docs

**What stays in memory:**

- Hard rules section at the top of MEMORY.md
- Process/behavioral feedback: `feedback_no_coauthored_by`, `feedback_git_no_cd`, `feedback_design_violations`, `feedback_copy_reference`, `feedback_one_change_at_a_time`, `feedback_docs_taxonomy`, `feedback_debug_state_breadcrumbs`, `feedback_assumptions`, `feedback_no_delete_data`, `feedback_documentation`, `feedback_readme_agent`, `feedback_memory_agent`
- Genuinely memory-shaped project state: `project_current_work`, `project_wttc_contract`, `project_orphaned_data`, `project_property_rename_gap`, `project_omega`
- Reference pointers: `reference_omega_db_internals`, `reference_ui_frontend`

## Scope — everything at once

This project ships the full knowledge base system in one go:

1. **Create `omega-knowledge-base` repo** with the initial structure above.
2. **Migrate `omega-docs` into Diátaxis** — restructure the existing `omega-docs` repo to use the four-bucket taxonomy at its root. Existing content in `omegadb/`, `omega-ai/`, `reference/` gets refactored into `how-to/`, `reference/`, `explanation/`, `gotchas/`.
3. **Create per-project `/docs/` trees** for every project listed in `projects/` — seed with the standard structure and migrate existing README content into it.
4. **Build the three skills** — navigator, mark-gap, fill-gap.
5. **Add the SessionStart hook** and test in an Omega-context session.
6. **Prune memory** per the pruning section above.
7. **Seed the knowledge base with this session's OQL gotchas:**
   - `with-table-if` schema collapse (silent row drops)
   - `(defined X)` being effectively universal
   - Mixed-literal map/list values in `throw` / `run-page` primitives (symbol binding doesn't resolve)
   - Universal quantifier fold behavior (`fold` folds over unique values, not rows — need `with-unique-by`)
   - Batch-size-dependent bugs (things that work at N=1 but break at N=100)
   - `page-query` + CouchDB Mango `skip` being O(skip) on `include_docs=true`
   - The `(= true false)` row-drop idiom

Doing everything at once is justified because the pieces are tightly coupled: the skills depend on the repo structure, the hook depends on the skills, the memory prune depends on the knowledge base existing to receive the content.

## Open questions for implementation

1. **Exact matching rules for the SessionStart hook's "is this an Omega session" check.** Current proposal is `cwd contains /Development/omega/ OR /Development/wttc/`. This may need refinement as edge cases come up (e.g., agents invoked outside a cwd context, sessions that start in `~` then cd).
2. **Where do skills physically live and how are they distributed?** The spec says `~/.claude/skills/omega/` but that's a local path. For a multi-machine contributor workflow, the skills may need to live in a shared plugin repo that gets installed.
3. **Gap key collision handling.** 8-char hash gives ~4B keys. Collisions are astronomically unlikely but not impossible. Decide whether to extend to 12 chars or accept collision risk.
4. **First-time setup.** The SessionStart hook assumes `omega-knowledge-base` exists at a known path. What happens on first run before the repo is cloned? Probably the hook should silently no-op when the path is missing.

## Not in scope

- GitHub issue integration for gaps (automatic queue is enough)
- Rebalance log (handled inline by fill-gap)
- CLAUDE.md files (deleted after skills prove out)
- Tutorials bucket (merged into how-to)
- Migration of contributor content *out* of memory until the knowledge base exists to receive it (chicken-and-egg; memory-prune happens last)
- Cross-agent learning (the gap → fill → learn loop exists, but we're not building dashboards or analytics on top of it)
