# Omega Knowledge Base — Plan 1 (Core Infrastructure)

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Stand up the `omega-knowledge-base` repo with contributor/projects scaffolding, build the three navigator/mark-gap/fill-gap skills, wire the SessionStart hook, and seed the OQL gotchas captured in the previous debugging session. **This plan does NOT migrate `omega-docs` or create per-project `/docs/` trees** — that is Plan 2.

**Architecture:** The knowledge base is a new private git repo at `~/Development/omega/omega-knowledge-base/`. Every directory has four bucket children (`how-to/`, `reference/`, `explanation/`, `gotchas/`) plus a `README.md` index (Diátaxis + gotchas taxonomy). The three skills live canonically inside the repo at `skills/omega-docs-*/SKILL.md` and are installed into `~/.claude/skills/omega/` by an install script. The SessionStart hook script lives at `scripts/omega-session-start.sh` in the repo and is copied into `~/.claude/hooks/`. OQL gotchas go into a `projects/query-omega-oql/gotchas/` subtree inside the knowledge base for Plan 1; Plan 2 may later migrate them to the actual query-omega-oql repo's `/docs/` tree.

**Tech Stack:** Plain markdown files, bash install scripts, Claude Code skill definitions (markdown with frontmatter), `~/.claude/settings.json` hook registration.

---

## Pre-flight notes

**Uncommitted changes in `query-omega-oql`:** The working tree has prior-session changes to `.gitignore` and `oql/app/database/property.oql` that are **unrelated to this plan**. Do NOT touch query-omega-oql in any task of this plan. Every file this plan creates is either inside the new `omega-knowledge-base/` repo or under `~/.claude/`.

**Plan file itself:** Lives in the `omega-docs` repo at `docs/superpowers/plans/2026-04-12-omega-knowledge-base-plan1-core.md`. Commit the plan file to `omega-docs` on its own before starting Task 0, or leave it uncommitted — either is fine, but do not bundle the plan commit with an omega-knowledge-base commit.

**Privacy:** `omega-knowledge-base` is intended to be private. This plan initializes it as a local git repo with no remote. Setting up a private GitHub remote is not in Plan 1 scope — the user will do that manually if/when desired.

**Hook coexistence:** `~/.claude/settings.json` already has a SessionStart hook that loads `project_current_work.md`. The new Omega navigator hook must be **added alongside** the existing hook, not replace it.

---

## File structure produced by this plan

```
~/Development/omega/omega-knowledge-base/          ← NEW PRIVATE REPO
  .gitignore
  README.md                                         ← root index (navigator reads this)
  gaps.md                                           ← rolling gap queue

  contributor/
    README.md
    how-to/
      README.md
      navigate-the-docs.md
      mark-a-gap.md
      rebalance-a-subtree.md
    reference/
      README.md
      diataxis.md
      path-prediction-rules.md
    explanation/
      README.md
      why-diataxis-plus-gotchas.md
      why-descend-from-root.md
    gotchas/
      README.md                                     ← empty list placeholder

  projects/
    README.md                                       ← lists every project pointer
    omega-docs.md
    omega-db.md
    query-omega-cli.md
    query-omega-api.md
    query-omega-mcp.md
    omega-cli.md
    omega-ai-components.md
    homes-dot-com.md
    gohighlevel.md
    query-omega-oql/                                ← promoted to subtree (has gotchas)
      README.md
      how-to/README.md
      reference/README.md
      explanation/README.md
      gotchas/
        README.md
        with-table-if-schema-collapse.md
        defined-is-universal.md
        mixed-literal-throw-run-page.md
        fold-folds-unique-values.md
        batch-size-dependent-bugs.md
        page-query-skip-on-include-docs.md
        row-drop-idiom.md
        event-subscriptions-die-on-jvm-restart.md
        four-twenty-two-falls-through-dispatch.md

  skills/                                           ← canonical skill sources
    omega-docs-navigator/SKILL.md
    omega-docs-mark-gap/SKILL.md
    omega-docs-fill-gap/SKILL.md

  scripts/
    install-skills.sh                               ← copies skills to ~/.claude/skills/omega/
    install-hook.sh                                 ← copies session-start script to ~/.claude/hooks/
    omega-session-start.sh                          ← the SessionStart hook body

~/.claude/skills/omega/                             ← INSTALLED BY scripts/install-skills.sh
  omega-docs-navigator/SKILL.md
  omega-docs-mark-gap/SKILL.md
  omega-docs-fill-gap/SKILL.md

~/.claude/hooks/                                    ← INSTALLED BY scripts/install-hook.sh
  omega-session-start.sh

~/.claude/settings.json                             ← MODIFIED to register the new hook
```

---

## Task 0: Create the repo

**Files:**
- Create: `~/Development/omega/omega-knowledge-base/` (directory)
- Create: `~/Development/omega/omega-knowledge-base/.gitignore`
- Create: `~/Development/omega/omega-knowledge-base/README.md`
- Create: `~/Development/omega/omega-knowledge-base/gaps.md`

- [ ] **Step 1: Create the repo directory and initialize git**

Run:
```bash
mkdir -p ~/Development/omega/omega-knowledge-base && \
git -C ~/Development/omega/omega-knowledge-base init -b main
```

Expected: `Initialized empty Git repository in /home/ryan/Development/omega/omega-knowledge-base/.git/`

- [ ] **Step 2: Create `.gitignore`**

Write `~/Development/omega/omega-knowledge-base/.gitignore`:
```
.DS_Store
*.swp
*.bak
.claude-local/
```

- [ ] **Step 3: Create the root `README.md`**

Write `~/Development/omega/omega-knowledge-base/README.md`:

````markdown
# Omega Knowledge Base

Contributor knowledge base for the Omega ecosystem. This is the root of a self-similar Diátaxis+gotchas tree that an agent navigates to answer questions about Omega without re-reading code.

## If you are an agent

1. You should have arrived here by invoking the `omega:omega-docs-navigator` skill. If you did not, invoke it now.
2. Commit to the **descend-from-root** navigation rule: never jump directly to a leaf, always walk parent READMEs on the way down.
3. Before falling back to code for any question, invoke `omega:omega-docs-mark-gap` to record the gap.

## Taxonomy

Every directory in this tree has the same four buckets as children, plus a `README.md` index:

| Bucket | What goes here | Question form |
|---|---|---|
| `how-to/` | Task recipes, workflows, deploy steps | "How do I …" |
| `reference/` | APIs, schemas, IDs, config, factual lookup | "What is …" / "Where is …" |
| `explanation/` | Concepts, design rationale, the "why" | "How does X work" / "Why is …" |
| `gotchas/` | Failure modes, footguns, debugging patterns | "Why did X fail" / "What bit us" |

Two top-level containers sit above the taxonomy buckets:

- **`contributor/`** — meta docs about how this knowledge base itself works (navigation, gap marking, rebalancing, taxonomy).
- **`projects/`** — one entry per project in the Omega ecosystem. Most entries are a single pointer file; some (like `query-omega-oql/`) have been promoted to full subtrees.

## Contributor docs

- [contributor/how-to/navigate-the-docs.md](contributor/how-to/navigate-the-docs.md) — the descend-from-root discipline
- [contributor/how-to/mark-a-gap.md](contributor/how-to/mark-a-gap.md) — how to use the mark-gap skill
- [contributor/how-to/rebalance-a-subtree.md](contributor/how-to/rebalance-a-subtree.md) — when and how to split a bucket into a subtree
- [contributor/reference/diataxis.md](contributor/reference/diataxis.md) — the 4-axis taxonomy explained
- [contributor/reference/path-prediction-rules.md](contributor/reference/path-prediction-rules.md) — the question-to-path algorithm
- [contributor/explanation/why-diataxis-plus-gotchas.md](contributor/explanation/why-diataxis-plus-gotchas.md)
- [contributor/explanation/why-descend-from-root.md](contributor/explanation/why-descend-from-root.md)

## Projects

See [projects/README.md](projects/README.md) for the full list.

Highlights:

- [projects/query-omega-oql/](projects/query-omega-oql/README.md) — core OQL library (promoted to subtree; has gotchas)
- [projects/omega-db.md](projects/omega-db.md) — OmegaDB engine
- [projects/omega-docs.md](projects/omega-docs.md) — public docs
- [projects/homes-dot-com.md](projects/homes-dot-com.md) — HDC data pipeline
- [projects/gohighlevel.md](projects/gohighlevel.md) — GHL CRM connector

## Gap queue

Open documentation gaps are tracked in [gaps.md](gaps.md). Fill-gap agents work this queue.
````

- [ ] **Step 4: Create `gaps.md`**

Write `~/Development/omega/omega-knowledge-base/gaps.md`:

```markdown
# Documentation Gaps

Entries are work items for `omega:omega-docs-fill-gap`. Each gap has a stable 8-char hash key. Resolved gaps are **deleted**, not marked — full history lives in `git log -- gaps.md`.

Gap entry format:

```
## gap-<8-char-hash>

- **question**: <paraphrased question that triggered the lookup>
- **predicted-path**: <path from knowledge-base root, no leading slash>
- **parent-chain**:
  - <path-1>
  - <path-2>
- **fallback-source**: <file path, command, or "none">
- **recorded-at**: <ISO 8601 UTC>
- **status**: open | claimed
- **claimed-by**: <agent id or empty>
- **notes**: <optional>

---
```

(No open gaps yet.)
```

- [ ] **Step 5: Verify files and commit**

Run:
```bash
ls -la ~/Development/omega/omega-knowledge-base/ && \
git -C ~/Development/omega/omega-knowledge-base add .gitignore README.md gaps.md && \
git -C ~/Development/omega/omega-knowledge-base commit -m "init: omega-knowledge-base root"
```

Expected: `4 files changed` (including .gitignore), commit created on `main`.

---

## Task 1: Contributor how-to tree

**Files:**
- Create: `~/Development/omega/omega-knowledge-base/contributor/README.md`
- Create: `~/Development/omega/omega-knowledge-base/contributor/how-to/README.md`
- Create: `~/Development/omega/omega-knowledge-base/contributor/how-to/navigate-the-docs.md`
- Create: `~/Development/omega/omega-knowledge-base/contributor/how-to/mark-a-gap.md`
- Create: `~/Development/omega/omega-knowledge-base/contributor/how-to/rebalance-a-subtree.md`

- [ ] **Step 1: Create `contributor/README.md`**

Write `~/Development/omega/omega-knowledge-base/contributor/README.md`:

````markdown
# Contributor

Meta docs about how the Omega knowledge base works. These are the rules the agent must follow when navigating, recording gaps, and rebalancing the tree.

## how-to/

- [navigate-the-docs.md](how-to/navigate-the-docs.md) — the descend-from-root discipline
- [mark-a-gap.md](how-to/mark-a-gap.md) — how to invoke the mark-gap skill
- [rebalance-a-subtree.md](how-to/rebalance-a-subtree.md) — when and how to promote or consolidate

## reference/

- [diataxis.md](reference/diataxis.md) — the 4-axis taxonomy explained
- [path-prediction-rules.md](reference/path-prediction-rules.md) — question-to-path algorithm

## explanation/

- [why-diataxis-plus-gotchas.md](explanation/why-diataxis-plus-gotchas.md) — why this taxonomy
- [why-descend-from-root.md](explanation/why-descend-from-root.md) — why we walk the parent chain

## gotchas/

- (none yet — see [gotchas/README.md](gotchas/README.md))
````

- [ ] **Step 2: Create `contributor/how-to/README.md`**

Write `~/Development/omega/omega-knowledge-base/contributor/how-to/README.md`:

```markdown
# Contributor · how-to

Task recipes for contributors (humans or agents) working on the knowledge base itself.

- [navigate-the-docs.md](navigate-the-docs.md) — how to find an answer using descend-from-root
- [mark-a-gap.md](mark-a-gap.md) — how to record a missing doc
- [rebalance-a-subtree.md](rebalance-a-subtree.md) — how to promote a flat file into a subtree
```

- [ ] **Step 3: Create `contributor/how-to/navigate-the-docs.md`**

Write `~/Development/omega/omega-knowledge-base/contributor/how-to/navigate-the-docs.md`:

````markdown
# How to navigate the Omega docs

**This is non-negotiable.** The descend-from-root rule exists to (a) carry parent context down with every answer, (b) make gap detection reliable, and (c) keep navigation auditable.

## The rule

1. **Start at `omega-knowledge-base/README.md`.** Always. Never jump to a leaf.
2. **Identify scope.** Which project is the question about? Pick a `projects/<name>` (or a `contributor/` topic if the question is about the knowledge base itself).
3. **Identify axis.** Use the axis decision rule (below) to pick `how-to/`, `reference/`, `explanation/`, or `gotchas/`.
4. **Walk the parent chain.** Read each `README.md` on the way from the root to the target — the root, then the scope's `README.md`, then the axis's `README.md`, then the target file. Parent READMEs change how you read leaves.
5. **Construct the predicted path:** `<scope>/<axis>/<topic>.md`.
6. **If the path resolves**, read the target. Done.
7. **If the path does not resolve or the content is insufficient**, invoke `omega:omega-docs-mark-gap` BEFORE falling back to code.

## Axis decision rule

Ask one question about your question Q:

| Phrasing of Q | Axis |
|---|---|
| "How do I …" / "How should I …" | `how-to/` |
| "What is …" / "Where is the X ID" / "What's the schema for …" | `reference/` |
| "Why does … (in general)" / "How does X work" / "What's the design of …" | `explanation/` |
| "Why did X fail" / "Why won't X work" / "What happened when I …" | `gotchas/` |

### Two edge cases

- **"How does X work"** is `explanation/`, not `how-to/`. If the answer is a mechanism, it's explanation. If the answer is commands to run, it's how-to.
- **"Why does this bug happen"** is `gotchas/`, not `explanation/`. Gotchas are "things that bit us." Explanation is "here's the design."

## When prediction is wrong

Two failure modes:

1. **Path doesn't exist.** Mark a gap with predicted path, question, and parent chain. Then try: (a) sibling README for alternate topic name, (b) fall back to code as last resort.
2. **Path exists but content is stub or obsolete.** Mark a gap for content. Fall back to code.

**Fallback to code is always accompanied by a gap marker.** Every code read for a question that docs should answer is a signal the docs have a gap.

## Non-negotiables

- Never jump directly to a leaf.
- Never fall back to grep/code without first marking a gap.
- Never mark a gap without a complete parent chain — mark-gap will refuse.
````

- [ ] **Step 4: Create `contributor/how-to/mark-a-gap.md`**

Write `~/Development/omega/omega-knowledge-base/contributor/how-to/mark-a-gap.md`:

````markdown
# How to mark a gap

When a predicted path doesn't resolve (or resolves to stub content), you invoke `omega:omega-docs-mark-gap` with one or more gap entries.

## Inputs per gap

- `question` — paraphrased question you were trying to answer
- `predicted_path` — the path you tried, relative to `omega-knowledge-base/`
- `parent_chain` — list of READMEs you read during descent (required, validated)
- `fallback_source` — what you read instead (file path, command, or `"none"`)

## Batch, don't stream

If you hit multiple gaps in the same research session, collect them and invoke mark-gap once with the whole batch. Mark-gap will spawn fill-gap agents in parallel when the batch is large.

- 1 gap → 1 fill-gap agent
- 2–3 gaps → 1–2 agents (usually sequential)
- 4–5 gaps → 2–3 agents in parallel
- 6+ gaps → capped at 3 parallel, remainder sequential

## The skill will refuse if

- `parent_chain` is shorter than the predicted path's depth — this is the consistency enforcement mechanism. Complete the descent first, then retry.
- The predicted path is outside `omega-knowledge-base/` or a project's `/docs/` tree — we don't track gaps in arbitrary code.

## After marking

Fall back to reading code or commits to answer the original question. The fill-gap agent will write the missing doc asynchronously — you don't need to block on it.
````

- [ ] **Step 5: Create `contributor/how-to/rebalance-a-subtree.md`**

Write `~/Development/omega/omega-knowledge-base/contributor/how-to/rebalance-a-subtree.md`:

````markdown
# How to rebalance a subtree

Any leaf can become a subtree with the four buckets inside it. This is the rebalancing primitive. Fill-gap agents do this automatically when they detect a trigger.

## Promotion triggers

Promote when any of:

- A bucket has **more than ~7 entries** (Miller's number — the signal to consider promotion).
- A single file has **grown beyond a screen or two** and would benefit from axis separation.
- A topic has **high gotcha surface** — when you're writing the third gotcha about a topic, promote so the new subtree gets its own `gotchas/` bucket.

## Consolidation triggers

Consolidate (cautiously) when:

- A subtree's `how-to/` has 1–2 entries and no other bucket is populated — over-promoted, pull it back.
- Multiple sibling files share 80%+ of their content with only small variations — combine into one file with per-variant sections.
- A subtree's `gotchas/` stayed empty long enough — consider flattening.

**Never consolidate across different axis buckets.** Merging a `how-to/` file with an `explanation/` file is a taxonomy violation, not a consolidation.

## Promotion example

Before:
```
projects/omega-db/how-to/deploy.md      ← growing past one screen
```

After:
```
projects/omega-db/how-to/deploy/
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

## Executing a rebalance

1. **Detect imbalance** (trigger met).
2. **Propose:** `{action, source_path, target_path, files_affected, justification}`. Promote is usually safe on agent judgment; consolidate requires user confirmation unless files are clearly redundant.
3. **Execute:**
   - Move/rewrite files into the new structure.
   - Update all parent READMEs to index the new structure.
   - **Walk every `.md` in the knowledge base looking for stale links to the old path and rewrite them.**
   - Remove any resolved gap entries from `gaps.md` that were about the promoted topic.
4. **Commit atomically:**
   ```
   rebalance: promote <bucket>/<topic> to subtree (N files)
   ```

## Safety rails

- **Promote freely** on agent judgment when a bucket exceeds 7 entries and one file is clearly the biggest.
- **Consolidate cautiously** — confirm with user unless files are redundant.
- **Never cross axis boundaries** in a consolidation.
- **Never leave dangling links** — the cross-ref walk is mandatory.
````

- [ ] **Step 6: Verify and commit**

Run:
```bash
ls ~/Development/omega/omega-knowledge-base/contributor/how-to/ && \
git -C ~/Development/omega/omega-knowledge-base add contributor/ && \
git -C ~/Development/omega/omega-knowledge-base commit -m "docs: contributor how-to (navigate, mark-gap, rebalance)"
```

Expected: 5 files added, 1 commit.

---

## Task 2: Contributor reference, explanation, gotchas

**Files:**
- Create: `~/Development/omega/omega-knowledge-base/contributor/reference/README.md`
- Create: `~/Development/omega/omega-knowledge-base/contributor/reference/diataxis.md`
- Create: `~/Development/omega/omega-knowledge-base/contributor/reference/path-prediction-rules.md`
- Create: `~/Development/omega/omega-knowledge-base/contributor/explanation/README.md`
- Create: `~/Development/omega/omega-knowledge-base/contributor/explanation/why-diataxis-plus-gotchas.md`
- Create: `~/Development/omega/omega-knowledge-base/contributor/explanation/why-descend-from-root.md`
- Create: `~/Development/omega/omega-knowledge-base/contributor/gotchas/README.md`

- [ ] **Step 1: Create `contributor/reference/README.md`**

Write:

```markdown
# Contributor · reference

Factual lookup about the knowledge base itself — the taxonomy and the algorithms that operate on it.

- [diataxis.md](diataxis.md) — the 4-axis taxonomy (how-to, reference, explanation, gotchas) explained
- [path-prediction-rules.md](path-prediction-rules.md) — the mechanical question-to-path algorithm
```

- [ ] **Step 2: Create `contributor/reference/diataxis.md`**

Write `~/Development/omega/omega-knowledge-base/contributor/reference/diataxis.md`:

````markdown
# Diátaxis + gotchas

The taxonomy for every directory in the knowledge base. Based on [Diátaxis](https://diataxis.fr) with two modifications.

## The four buckets

| Bucket | Reading mode | Writing mode |
|---|---|---|
| `how-to/` | "I need to get X done, give me steps." | Task recipe. Commands + expected output. |
| `reference/` | "I need to look up a fact." | Factual. Tables, schemas, IDs, config values. |
| `explanation/` | "I want to understand how this works." | Narrative. Design, rationale, concepts. |
| `gotchas/` | "I hit a bug, help me diagnose." | Diagnostic. "If you see X, the cause is usually Y." |

## Two axes

Diátaxis organizes these along two axes:

- **Theoretical ↔ Practical** — explanation vs reference are theoretical; how-to and gotchas are practical.
- **Acquisition ↔ Application** — reference and how-to serve a working agent; explanation and gotchas serve learning or debugging.

## Modifications from standard Diátaxis

1. **`tutorials/` merged into `how-to/`.** Agents don't need guided onboarding walkthroughs. Humans who want a walkthrough read the project README.
2. **`gotchas/` added as a fourth bucket.** Failure-mode knowledge is distinct from both `explanation/` (design) and `reference/` (facts). A debugging session reads gotchas very differently from how it reads explanation.

## Rules for self-similarity

1. **Every directory has the four buckets as its children.** Plus optional top-level container dirs (`projects/`, `contributor/`) at the root.
2. **Every directory has a `README.md`** that indexes its children with one-line descriptions — sufficient for a descend-or-skip decision.
3. **A file lives at the deepest level where it still fits a single axis.** If a file is both concept and guide, split it.
4. **Recursion happens inside any bucket when it grows.** So `how-to/` can contain flat files OR a nested tree of topic-guides each with their own four sub-buckets.
5. **Cross-references by relative path.** Never absolute URLs. Navigable offline.
6. **Lowercase kebab-case paths.** Regex- and glob-friendly.

## Why Diátaxis specifically

- Most widely-adopted technical doc framework in the open-source world.
- Used by Django, NumPy, Cloudflare, Divio, GitLab.
- Industry vocabulary — "look it up in the reference docs" has a universal meaning.
- Clean underlying theory (see the axes above).
````

- [ ] **Step 3: Create `contributor/reference/path-prediction-rules.md`**

Write `~/Development/omega/omega-knowledge-base/contributor/reference/path-prediction-rules.md`:

````markdown
# Path prediction rules

The mechanical algorithm that translates a question into a predicted path.

## Algorithm

```
Given a question Q:
  1. Identify SCOPE — which project is this about?
     (If unclear, start at the knowledge-base root.)
  2. Identify AXIS — is this how-to / reference / explanation / gotcha?
  3. Identify TOPIC — what noun or verb phrase is Q about?
  4. Construct predicted path: <scope>/<axis>/<topic>.md
  5. Walk from knowledge-base root down to the prediction, reading each
     README.md on the way so context is carried down.
  6. If the path resolves → read the target. Done.
  7. If the path is missing → mark a gap and either:
     (a) read the sibling README to check for a different topic name
     (b) fall back to grep / code as a last resort (also marking a gap)
```

## Axis decision rule

| Question-word in Q | Axis |
|---|---|
| "How do I…" / "How should I…" | `how-to/` |
| "What is the…" (definition) / "Where is the X ID" / "What's the schema for…" | `reference/` |
| "Why does…" / "How does … work" / "What's the design of…" | `explanation/` |
| "Why did that fail" / "Why won't X" / "What happened when I…" | `gotchas/` |

### Edge cases

- **"How does X work"** is `explanation/`, not `how-to/`. Mechanism = explanation. Commands = how-to.
- **"Why does this bug happen"** is `gotchas/`, not `explanation/`. Things that bit us = gotchas. The design itself = explanation.

## Topic extraction

The `<topic>` segment is the kebab-cased noun or verb phrase the question is about:

- "How do I deploy omega-db?" → scope `omega-db`, axis `how-to`, topic `deploy` → `projects/omega-db/how-to/deploy.md`
- "Why did write-table throw in the pipeline?" → scope `query-omega-oql` (or `homes-dot-com`), axis `gotchas`, topic `write-table-throw` or more specifically the root cause → check `projects/query-omega-oql/gotchas/`.
- "What's the schema for Data/Filtered Agents?" → scope `homes-dot-com`, axis `reference`, topic `filtered-agents-schema`.

When in doubt, err toward a more general topic name (so sibling files collect under the same name) and let the fill-gap agent promote it later.

## Descend-from-root — not optional

You **never** jump directly to a leaf. You walk from `omega-knowledge-base/README.md` down to the target file, reading each parent README on the way. See [contributor/how-to/navigate-the-docs.md](../how-to/navigate-the-docs.md) for the procedure and [contributor/explanation/why-descend-from-root.md](../explanation/why-descend-from-root.md) for the rationale.
````

- [ ] **Step 4: Create `contributor/explanation/README.md`**

Write:

```markdown
# Contributor · explanation

The "why" of the knowledge base — the design choices behind the taxonomy and the navigation rules.

- [why-diataxis-plus-gotchas.md](why-diataxis-plus-gotchas.md) — why we chose this taxonomy
- [why-descend-from-root.md](why-descend-from-root.md) — why the agent must walk parent READMEs
```

- [ ] **Step 5: Create `contributor/explanation/why-diataxis-plus-gotchas.md`**

Write `~/Development/omega/omega-knowledge-base/contributor/explanation/why-diataxis-plus-gotchas.md`:

````markdown
# Why Diátaxis + gotchas

The taxonomy is Diátaxis with two modifications. This is the rationale.

## Why a formal taxonomy at all

The previous doc system was "write what you need where it seems to fit." Over time this produced:

- Duplication (the same fact in three files, two of which got stale).
- Uncertainty about where to look (is the deploy process a how-to or a reference?).
- Accidental burial (gotchas hidden inside long explanation essays).

A formal taxonomy makes the path to an answer predictable. The agent can translate a question into a path *without reading anything* — that's the whole point of the path prediction algorithm.

## Why Diátaxis specifically

Diátaxis separates four kinds of documentation along two axes:

|               | Theoretical      | Practical      |
|---------------|------------------|----------------|
| Acquisition   | explanation      | tutorial       |
| Application   | reference        | how-to         |

It's widely adopted (Django, NumPy, Cloudflare, GitLab, Divio) so the vocabulary is stable. The two-axis model has enough theoretical grounding that edge cases resolve cleanly ("is this acquisition or application?" → "am I learning or doing?").

## Why merge `tutorials/` into `how-to/`

Tutorials are linear learning walkthroughs aimed at humans who don't yet know the system. Agents don't need that reading mode — they need task recipes. The agents already have the system's vocabulary from their own training; they just need to know the specific steps for a specific task.

A human contributor who wants to learn the system end-to-end reads the project README, not a "getting started" walkthrough.

Dropping the tutorials bucket removes one reading mode the agent never uses and avoids the "is this a tutorial or a how-to?" edge case.

## Why add `gotchas/`

Failure-mode knowledge doesn't fit cleanly into any standard Diátaxis bucket:

- It's not **reference**, because a gotcha is a *narrative* about a failure — "if you hit X, the cause is usually Y, here's the fix." A table of field names can't carry that.
- It's not **explanation**, because explanation is about *design*. A gotcha is about things that went wrong *despite* the design.
- It's not **how-to**, because a how-to is a forward-looking recipe. A gotcha is backward-looking debugging.

Before we added the bucket, gotchas were buried inside `explanation/` essays and nobody could find them during debugging. Splitting gotchas out gives debuggers a dedicated reading mode that loads differently from either design or reference.

The signal to add a new gotcha is: "I just spent an hour diagnosing something." That hour is the cost of the gap. Capturing the lesson makes the next hit a 30-second lookup.

## Self-similarity — every directory has the same shape

The four-bucket structure applies at every depth. A single project's `how-to/` can itself contain a subtree (e.g., `how-to/deploy/` with its own four sub-buckets) when the topic grows. This is the rebalancing primitive — see [contributor/how-to/rebalance-a-subtree.md](../how-to/rebalance-a-subtree.md).

Self-similarity means the agent's descent algorithm is the *same* at every depth. There is no special case for "top level" vs "nested" — just walk the tree.
````

- [ ] **Step 6: Create `contributor/explanation/why-descend-from-root.md`**

Write `~/Development/omega/omega-knowledge-base/contributor/explanation/why-descend-from-root.md`:

````markdown
# Why descend from root

The agent **never** jumps directly to a leaf. It starts at `omega-knowledge-base/README.md`, walks the parent chain, and reads each `README.md` on the way down.

This rule looks pedantic. It is load-bearing.

## Reason 1: Parent READMEs carry context

A leaf file like `projects/homes-dot-com/gotchas/write-table-throws.md` is written *assuming* the reader knows which project they're in. The leaf doesn't re-state "this project uses OQL, it runs against OmegaDB, the pipeline has N stages." That context lives in the parent README.

If the agent jumps straight to the leaf, the missing context causes misreads. The leaf says "use the coordinator to resume" and the agent doesn't know what the coordinator *is* because it skipped the project README that introduces it.

Walking the chain reconstructs the reading mind the leaf author expected.

## Reason 2: Gap detection depends on it

If the agent can grep across the tree, it will accidentally find answers even when the predicted path was wrong. For example, the agent predicts `projects/omega-db/reference/schema.md`, that file doesn't exist, but grep finds the answer in `projects/homes-dot-com/explanation/pipeline-shape.md`.

Without descent, the agent reads the fallback, declares success, and the gap — `projects/omega-db/reference/schema.md` is missing — is silently absorbed. The next agent hits the same gap and does the same grep dance. The docs never improve.

With descent, the agent notices the path doesn't resolve, marks the gap, *then* falls back. The gap is now a work item and fill-gap will write the missing doc.

## Reason 3: It's auditable

A reader (human or another agent) can check "did the agent read each parent README before the child?" by looking at the tool call history. Either the chain is there or it isn't.

This matters when triaging an agent that gave a wrong answer. The first question is "was the answer in the docs?" The second is "did the agent walk down to it correctly?" Without descent, the second question is unanswerable.

## Cost

Descent is 3–5 extra `Read` calls per query. On the margin this is cheap — README files are small and cache-friendly, and the payoff is that every fall-back is flagged as a gap instead of silently absorbed.

## Non-negotiable

- Never jump to a leaf.
- Never fall back to code without marking a gap first.
- Mark-gap refuses entries without a full parent chain as the consistency enforcement.
````

- [ ] **Step 7: Create `contributor/gotchas/README.md` (empty placeholder)**

Write:

```markdown
# Contributor · gotchas

Failure modes and footguns about the knowledge base itself.

(None yet. When contributor workflow hits a footgun — e.g., a mark-gap invocation that silently failed, a rebalance that broke cross-links — record it here.)
```

- [ ] **Step 8: Verify and commit**

Run:
```bash
find ~/Development/omega/omega-knowledge-base/contributor -type f -name "*.md" | sort && \
git -C ~/Development/omega/omega-knowledge-base add contributor/ && \
git -C ~/Development/omega/omega-knowledge-base commit -m "docs: contributor reference + explanation + gotchas"
```

Expected: 7 new files listed, commit created.

---

## Task 3: Projects pointers (non-OQL projects)

**Files:**
- Create: `~/Development/omega/omega-knowledge-base/projects/README.md`
- Create: `~/Development/omega/omega-knowledge-base/projects/omega-docs.md`
- Create: `~/Development/omega/omega-knowledge-base/projects/omega-db.md`
- Create: `~/Development/omega/omega-knowledge-base/projects/query-omega-cli.md`
- Create: `~/Development/omega/omega-knowledge-base/projects/query-omega-api.md`
- Create: `~/Development/omega/omega-knowledge-base/projects/query-omega-mcp.md`
- Create: `~/Development/omega/omega-knowledge-base/projects/query-omega.md`
- Create: `~/Development/omega/omega-knowledge-base/projects/query-omega-website.md`
- Create: `~/Development/omega/omega-knowledge-base/projects/omega-cli.md`
- Create: `~/Development/omega/omega-knowledge-base/projects/omega-ai-components.md`
- Create: `~/Development/omega/omega-knowledge-base/projects/homes-dot-com.md`
- Create: `~/Development/omega/omega-knowledge-base/projects/gohighlevel.md`

- [ ] **Step 1: Create `projects/README.md`**

Write:

```markdown
# Projects

One entry per project in the Omega ecosystem. Most entries are a single pointer file; `query-omega-oql/` has been promoted to a full subtree because it accumulated gotcha content.

## Platform

- [omega-db.md](omega-db.md) — OmegaDB engine (Clojure runtime, CouchDB storage)
- [query-omega-oql/](query-omega-oql/README.md) — Core OQL library (page, impl, property, sec-index). **Subtree with gotchas.**
- [query-omega-api.md](query-omega-api.md) — Public OQL API layer
- [query-omega-cli.md](query-omega-cli.md) — `qo` CLI
- [query-omega-mcp.md](query-omega-mcp.md) — MCP server wrapping the CLI for AI agents
- [omega-cli.md](omega-cli.md) — Low-level `omega-cli run-query` for raw OQL files

## Frontend

- [query-omega.md](query-omega.md) — React frontend application for Omega
- [query-omega-website.md](query-omega-website.md) — Marketing + blog website

## Docs

- [omega-docs.md](omega-docs.md) — Public concept docs (OQL language, execution model, components, plugins)

## Component libraries

- [omega-ai-components.md](omega-ai-components.md) — Shared components: event handler, queries, protocols

## Plugins / customer projects

- [homes-dot-com.md](homes-dot-com.md) — Homes.com data pipeline
- [gohighlevel.md](gohighlevel.md) — GoHighLevel CRM connector
```

- [ ] **Step 2: Create `projects/omega-docs.md`**

Write `~/Development/omega/omega-knowledge-base/projects/omega-docs.md`:

```markdown
# omega-docs

Public concept docs for the Omega platform.

- **Repo:** `~/Development/omega/omega-docs`
- **README:** [../../../omega-docs/README.md](../../omega-docs/README.md)
- **Scope:** OQL language reference, execution model, components, plugins, public API reference. Nothing private.

## What lives here today

`omega-docs` is **not yet migrated** to the Diátaxis+gotchas taxonomy. That migration is Plan 2 of this project. Until then, content lives in ad-hoc folders like `omegadb/language/`, `omega-ai/`, etc.

Key files to know:

- `omegadb/language/control-flow.md` — `with-table-if` including the schema-collapse write-up (reference for the gotcha by the same name)
- `omegadb/language/` — OQL language reference
- `omega-ai/` — component system docs

## Private notes

None. This repo is public-clean. Anything private about OQL goes into `projects/query-omega-oql/` instead.
```

- [ ] **Step 3: Create `projects/omega-db.md`**

Write:

```markdown
# omega-db

The OmegaDB engine — Clojure runtime with a CouchDB-backed storage layer.

- **Repo:** `~/Development/omega/omega-db`
- **README:** [../../omega-db/README.md](../../omega-db/README.md)

## What this project is

Core runtime that executes OQL queries. Implements the clause evaluator, the page system, functors, subscriptions, and the CouchDB data types (views, queries, row-index, sec-index).

## Known failure modes (stubs for future gotcha migration)

- **CouchDB Mango `skip` is O(skip) with `include_docs=true`.** Fixed in commit `5365d92` (two-step query: view for `_id`s, then `_all_docs?keys=[...]` for bodies). See `src/omega_db/datastore/couchdb/data_types/query.clj`. This is currently recorded under `projects/query-omega-oql/gotchas/page-query-skip-on-include-docs.md` because it was triggered via OQL and the fix note is short.
- **Event subscriptions die with JVM restart.** No primitive to survive deploys. Recorded under `projects/query-omega-oql/gotchas/event-subscriptions-die-on-jvm-restart.md`.

Both gotchas will likely migrate to a promoted `projects/omega-db/` subtree in a later rebalance.

## Private notes

- Do not restart the DB without asking Ryan.
- Do not drop or truncate tables with large data — all pipeline tables take hours to refill.
```

- [ ] **Step 4: Create `projects/query-omega-cli.md`**

Write:

```markdown
# query-omega-cli

The `qo` CLI — the primary user-facing interface for Omega.

- **Repo:** `~/Development/omega/query-omega-cli`
- **README:** [../../query-omega-cli/README.md](../../query-omega-cli/README.md)
- **CLAUDE.md:** [../../query-omega-cli/CLAUDE.md](../../query-omega-cli/CLAUDE.md) — mandatory read before editing this project.

## What this project is

Node-based CLI that wraps the public OQL API. Commands: `qo run`, `qo query`, `qo push`, `qo pull`, `qo logs`, etc. All code paths go through the public API layer (`query-omega-api`).

## Rules for this project

- **Never use internal datastores in the CLI** — public API endpoints only (`omega/query-omega/public/`).
- Use `qo` rather than `npx` for local invocations: `qo run <page>`.
- Always `cd` into the target project directory before running `qo` — the shell resets between invocations.
- Read the project's CLAUDE.md before editing any code here.
```

- [ ] **Step 5: Create `projects/query-omega-api.md`**

Write:

```markdown
# query-omega-api

Public OQL API layer.

- **Repo:** `~/Development/omega/query-omega-api`
- **README:** [../../query-omega-api/README.md](../../query-omega-api/README.md)

## What this project is

HTTP + WebSocket API that exposes OQL execution to clients (CLI, MCP server, frontend). Sits between `query-omega-cli` / frontend and `omega-db`.

## Private notes

None beyond the README.
```

- [ ] **Step 6: Create `projects/query-omega-mcp.md`**

Write:

```markdown
# query-omega-mcp

MCP server wrapping the `qo` CLI, exposed to AI agents.

- **Repo:** `~/Development/omega/query-omega-mcp`
- **README:** [../../query-omega-mcp/README.md](../../query-omega-mcp/README.md)

## What this project is

Model Context Protocol server that surfaces Omega tools (`query`, `run`, `logs`, `describe`, `ls`, etc.) to agents like Claude Code. Thin wrapper — business logic lives in the CLI and API.
```

- [ ] **Step 7: Create `projects/query-omega.md`**

Write:

```markdown
# query-omega

React frontend application for Omega.
1
- **Repo:** `~/Development/omega/query-omega`
- **README:** [../../query-omega/README.md](../../query-omega/README.md)

## What this project is

The primary web UI for interacting with Omega: browsing pages, running queries, inspecting results. Talks to `query-omega-api` over HTTP + WebSocket. Most user-facing Omega interactions go through this app.

## Private notes

- Frontend internals for *debugging* are recorded in the `reference_ui_frontend.md` memory. Concepts and architecture for day-to-day work live in `omega-docs`.
```

- [ ] **Step 8: Create `projects/query-omega-website.md`**

Write:

```markdown
# query-omega-website

Marketing + blog website for Omega.

- **Repo:** `~/Development/omega/query-omega-website`
- **README:** [../../query-omega-website/README.md](../../query-omega-website/README.md)

## What this project is

Public-facing marketing site and blog. Separate from `query-omega` (the React application). Contains docs landing pages, blog posts, and marketing content.

## Private notes

None — this repo is public-facing.
```

- [ ] **Step 9: Create `projects/omega-cli.md`**

Write:

```markdown
# omega-cli

Low-level CLI for running raw OQL files directly against omega-db.

- **Repo:** `~/Development/omega/omega-cli`
- **README:** [../../omega-cli/README.md](../../omega-cli/README.md)

## What this project is

Lower-level than `query-omega-cli` — `omega-cli run-query path/to/file.oql` executes a standalone OQL file against a running omega-db instance. Used for deploy-by-hand workflows and scratch debugging.

## Private notes

- Currently used as the workaround for deploying `query-omega-oql` while `git push origin production` is broken. See `projects/query-omega-oql/gotchas/deploy-broken-via-push.md` (future gap, not yet filled).
```

- [ ] **Step 10: Create `projects/omega-ai-components.md`**

Write:

```markdown
# omega-ai-components

Shared component library for the Omega ecosystem.

- **Repo:** `~/Development/omega/omega-ai-components`
- **README:** [../../omega-ai-components/README.md](../../omega-ai-components/README.md)

## What this project is

Reusable components: event handler, queries, protocols. Used by customer-specific plugins (`homes-dot-com`, `gohighlevel`) to avoid duplicating the same event-routing + query code.

## Rules for this project

- Coordinators are pure routers: no business logic, only dispatch to sibling event handlers.
- `on-event` dispatches via `run-page` — never `emit-event` inside `on-event`.
- `emit-event` lives in its own query, separate from `on-event`.
- Never violate the documented component design — read the project README before editing.
```

- [ ] **Step 11: Create `projects/homes-dot-com.md`**

Write:

```markdown
# homes-dot-com (HDC)

Customer data pipeline for homes.com → GoHighLevel sync.

- **Repo:** `~/Development/wttc/homes-dot-com`
- **README:** [../../../wttc/homes-dot-com/README.md](../../wttc/homes-dot-com/README.md)

## What this project is

Multi-stage data pipeline that ingests agents from homes.com, filters them, and syncs them into a GoHighLevel CRM instance as contacts. Each pipeline stage materializes its output to a database table (Dagster-style).

## Key tables

- `Data/Filtered Agents` — ~150k rows, the input universe
- `Data/GHL Contacts` — the pipeline output, one row per synced contact
- Intermediate tables per stage

## Private notes

- Pipeline is long-running (hours). Never drop its tables without explicit permission — refills take hours.
- The pipeline runs via a coordinator event loop. Deploys kill subscriptions (see `projects/query-omega-oql/gotchas/event-subscriptions-die-on-jvm-restart.md`).
- 422 responses from the GHL API fall through 2xx/400 dispatch and must be explicitly filtered. See `projects/query-omega-oql/gotchas/four-twenty-two-falls-through-dispatch.md`.

## Recent state

See `~/.claude/projects/-home-ryan-Development-omega-query-omega-oql/memory/next_session.md` for the current pipeline skip position and any known gaps. This is session state, not a doc — do not migrate it here.
```

- [ ] **Step 12: Create `projects/gohighlevel.md`**

Write:

```markdown
# gohighlevel (GHL)

GoHighLevel CRM connector.

- **Repo:** `~/Development/wttc/gohighlevel`
- **README:** [../../../wttc/gohighlevel/README.md](../../wttc/gohighlevel/README.md)

## What this project is

Query library + app connector for the GHL API. The query library provides wrappers like `get-contact`, `create-contact`, `update-contact` — each is an OQL file that makes an HTTP call via the `http-request` primitive and normalizes the response.

## Private notes

- Always use `http-request` — `http-get-request` and `http-post-request` are deprecated (they don't capture status codes).
- Watch for Cloudflare headers in responses (see the get-contact design doc in omega-docs specs).
```

- [ ] **Step 13: Verify and commit**

Run:
```bash
ls ~/Development/omega/omega-knowledge-base/projects/ && \
git -C ~/Development/omega/omega-knowledge-base add projects/ && \
git -C ~/Development/omega/omega-knowledge-base commit -m "docs: project pointer files (non-OQL)"
```

Expected: 12 new files (README + 11 pointer files), commit created.

---

## Task 4: query-omega-oql subtree scaffolding

**Files:**
- Create: `~/Development/omega/omega-knowledge-base/projects/query-omega-oql/README.md`
- Create: `~/Development/omega/omega-knowledge-base/projects/query-omega-oql/how-to/README.md`
- Create: `~/Development/omega/omega-knowledge-base/projects/query-omega-oql/reference/README.md`
- Create: `~/Development/omega/omega-knowledge-base/projects/query-omega-oql/explanation/README.md`
- Create: `~/Development/omega/omega-knowledge-base/projects/query-omega-oql/gotchas/README.md`

- [ ] **Step 1: Create the subtree `README.md`**

Write `~/Development/omega/omega-knowledge-base/projects/query-omega-oql/README.md`:

```markdown
# query-omega-oql

Core OQL library — the system pages + implementations that define the OQL language runtime (page, impl, property, sec-index systems).

- **Repo:** `~/Development/omega/query-omega-oql`
- **README:** [../../../query-omega-oql/README.md](../../../query-omega-oql/README.md)
- **Why this project is a subtree:** it accumulated enough OQL language gotchas during HDC debugging that a flat pointer file was insufficient.

## Rules for this project

- **Do not change without strong evidence of a bug.** This is the OQL standard library — changes have wide blast radius.
- Use library-level clauses for CRUD operations. Never call `defterm` operations directly.
- Never use boolean `true` / `false` — always use the strings `"true"` and `"false"` in OQL.

## how-to/

See [how-to/README.md](how-to/README.md).

## reference/

See [reference/README.md](reference/README.md).

## explanation/

See [explanation/README.md](explanation/README.md).

## gotchas/

See [gotchas/README.md](gotchas/README.md) — the main reason this project is a subtree. **Read these if you're about to debug an OQL surprise.**
```

- [ ] **Step 2: Create `how-to/README.md`**

Write:

```markdown
# query-omega-oql · how-to

Task recipes for working with the OQL core library.

(None yet. First entries will likely cover: deploying after a push-production break, adding a new library clause, promoting a scratch OQL file into an implementation.)
```

- [ ] **Step 3: Create `reference/README.md`**

Write:

```markdown
# query-omega-oql · reference

Factual lookup for OQL core library internals.

(None yet. First entries will likely cover: the page/impl/property/sec-index system layouts, library clause naming conventions, canonical schemas.)
```

- [ ] **Step 4: Create `explanation/README.md`**

Write:

```markdown
# query-omega-oql · explanation

Design and rationale for the OQL core library.

(None yet. Design material currently lives in `omega-docs`; once migrated, pointers will land here.)
```

- [ ] **Step 5: Create `gotchas/README.md` (will be expanded in Tasks 5 and 6)**

Write `~/Development/omega/omega-knowledge-base/projects/query-omega-oql/gotchas/README.md`:

```markdown
# query-omega-oql · gotchas

OQL language footguns — things that bit us and cost hours to diagnose. Read these *before* debugging an OQL surprise.

(Entries added in Task 5 and Task 6 of Plan 1.)
```

- [ ] **Step 6: Verify and commit**

Run:
```bash
find ~/Development/omega/omega-knowledge-base/projects/query-omega-oql -type f | sort && \
git -C ~/Development/omega/omega-knowledge-base add projects/query-omega-oql/ && \
git -C ~/Development/omega/omega-knowledge-base commit -m "docs: query-omega-oql subtree scaffolding"
```

Expected: 5 files, 1 commit.

---

## Task 5: Seed OQL gotchas (batch 1 of 2 — files 1–5)

**Files:**
- Create: `~/Development/omega/omega-knowledge-base/projects/query-omega-oql/gotchas/with-table-if-schema-collapse.md`
- Create: `~/Development/omega/omega-knowledge-base/projects/query-omega-oql/gotchas/defined-is-universal.md`
- Create: `~/Development/omega/omega-knowledge-base/projects/query-omega-oql/gotchas/mixed-literal-throw-run-page.md`
- Create: `~/Development/omega/omega-knowledge-base/projects/query-omega-oql/gotchas/fold-folds-unique-values.md`
- Create: `~/Development/omega/omega-knowledge-base/projects/query-omega-oql/gotchas/batch-size-dependent-bugs.md`
- Modify: `~/Development/omega/omega-knowledge-base/projects/query-omega-oql/gotchas/README.md` (update index)

- [ ] **Step 1: Create `with-table-if-schema-collapse.md`**

Write `~/Development/omega/omega-knowledge-base/projects/query-omega-oql/gotchas/with-table-if-schema-collapse.md`:

````markdown
# `with-table-if` schema collapse (silent row drops)

**Symptom:** A `with-table-if` branch drops rows silently. Row count going into the branch is N, coming out is M < N, and no error fires.

**Root cause:** The `with-table-if` schema did not include a per-row identity column (e.g. `AgentLink`). Without a unique identity in the schema, rows merge via an implicit `SELECT DISTINCT` on the schema columns. Rows with duplicate values across the schema columns collapse into one.

This is a structural property of how `with-table-if` builds the branch table, not a bug in the clause. The full SQL-analogy write-up lives in `omega-docs/omegadb/language/control-flow.md` (look for "descend-from-root" and "rebalancing" in that file).

## Minimal reproduction

```oql
(Qo.Db.Prop/prop-val! "FolderId" "PageId" "NeedsCreate" NeedsCreate)
(with-table-if [NeedsCreate] (= NeedsCreate "true")
  (http-request ...))
```

Expected: one HTTP call per row where `NeedsCreate = "true"`. Actual: one HTTP call total (all rows with `NeedsCreate = "true"` collapsed because `NeedsCreate` was the only schema column).

## The fix

Include a per-row identity column in the schema:

```oql
(with-table-if [AgentLink NeedsCreate] (= NeedsCreate "true")
  (http-request ...))
```

`AgentLink` is unique per row, so no collapse. Alternatively, include the page ID or any guaranteed-unique column.

## How to diagnose

1. Count rows going into the branch: `(fold 0 _ + _ Count)` before the `with-table-if`.
2. Count rows inside the branch's effect: same fold after.
3. If inside < outside, check the `with-table-if` schema. If it has no unique column, that's the bug.

## Related

- [defined-is-universal.md](defined-is-universal.md) — related footgun where the branch condition is always true.
- `omega-docs/omegadb/language/control-flow.md` — canonical language-level write-up.
````

- [ ] **Step 2: Create `defined-is-universal.md`**

Write `~/Development/omega/omega-knowledge-base/projects/query-omega-oql/gotchas/defined-is-universal.md`:

````markdown
# `(defined X)` is effectively universal

**Symptom:** A `with-table-if` branch with `(defined X)` as its condition fires for every row, even rows you thought X wasn't set for.

**Root cause:** `(defined X)` checks whether the symbol `X` is bound in the current lexical context, not whether a specific row has a value for a column named `X`. Inside a branch where `X` is in scope, `(defined X)` is true *universally* — every row sees a bound `X`, so the condition is always satisfied.

In practice this means `(defined X)` alone is never the right condition for `with-table-if`. It's the OQL equivalent of `WHERE true`.

## Minimal reproduction

```oql
(Qo.Db.Prop/prop-val! "FolderId" "PageId" "SomeOptionalField" SomeOptionalField)
(with-table-if [RowId SomeOptionalField] (defined SomeOptionalField)
  (http-request ...))
```

Expected (naive reading): HTTP call only for rows where `SomeOptionalField` has a value. Actual: HTTP call for every row, because `SomeOptionalField` is *always* in lexical scope after the `prop-val!` clause binds it.

## The fix

Combine `(defined X)` with a concrete per-row predicate:

```oql
(with-table-if [RowId SomeOptionalField] (= SomeOptionalField "some-sentinel")
  (http-request ...))
```

Or branch on a column that actually varies per row, like `NeedsCreate`:

```oql
(with-table-if [RowId NeedsCreate] (= NeedsCreate "true")
  (http-request ...))
```

## Why this exists

OQL's scoping model is lexical — symbols bind for the rest of the clause body. `(defined X)` is meaningful in macro contexts (compile-time) but degenerate in runtime branch conditions. It's kept because it's occasionally useful at the very top of a query to check what's available, not at the row-filtering level.

## Related

- [with-table-if-schema-collapse.md](with-table-if-schema-collapse.md) — a separate footgun where the branch *does* fire but drops rows on merge.
````

- [ ] **Step 3: Create `mixed-literal-throw-run-page.md`**

Write `~/Development/omega/omega-knowledge-base/projects/query-omega-oql/gotchas/mixed-literal-throw-run-page.md`:

````markdown
# Mixed-literal maps/lists in `throw` / `run-page`

**Symptom:** A `throw` or `run-page` call with a literal map or list argument containing symbol references produces an error about unbound symbols, or silently passes the symbol names as strings instead of their values.

**Root cause:** Symbol bindings do not resolve inside literal map or list arguments to the `throw` and `run-page` primitives. The contents of the literal are read at parse time, before the row context binds symbols — so `{"key" MySymbol}` becomes a literal map with the *text* `"MySymbol"`, not the value of `MySymbol`.

## Minimal reproduction

```oql
(Qo.Db.Prop/prop-val! "FolderId" "PageId" "Payload" Payload)

; ❌ does NOT work — Payload stays as a literal symbol name
(throw "error-tag" {"payload" Payload "count" 42})
```

The error tag fires, but `payload` in the thrown map is the literal string `"Payload"` (or an unbound-symbol error), not the row's value.

## The fix

Bind the map/list to a symbol first with `(= SymbolName {...})`, then pass the symbol:

```oql
(Qo.Db.Prop/prop-val! "FolderId" "PageId" "Payload" Payload)

; ✅ bind first, then throw
(= Data {"payload" Payload "count" 42})
(throw "error-tag" Data)
```

Same pattern for `run-page`:

```oql
(= Args {"row-id" RowId "status" Status})
(run-page "some-page" Args)
```

## Why

The primitives accept their arguments as values, not as expressions — the argument position is a value slot. When you write a literal map directly in the call site, OQL reads it as a literal value before evaluating expressions. Only when the map is bound to a symbol via `(=)` does the binder first evaluate the expressions inside the map and then assign the result.

## Related

- Internal literal-resolution semantics are documented in `omega-docs/omegadb/language/` (search for "lexical scope" and "expression vs literal").
````

- [ ] **Step 4: Create `fold-folds-unique-values.md`**

Write `~/Development/omega/omega-knowledge-base/projects/query-omega-oql/gotchas/fold-folds-unique-values.md`:

````markdown
# `fold` folds over unique values, not rows

**Symptom:** `(fold 0 X + Sum)` returns a value that maxes out at 1 (or at the count of distinct values of `X`), not the expected row count.

**Root cause:** `fold` folds over the *unique values* of its input symbol, not over rows. If `X` is a column whose values are `0` or `1`, `fold` sees two unique values (`0` and `1`) and returns a sum of 1, regardless of how many rows had `X = 1`.

## Minimal reproduction

```oql
(Qo.Db.Prop/prop-val! "FolderId" "PageId" "NeedsCreate" NeedsCreate)
(fold 0 NeedsCreate + Sum)  ; ❌ Sum maxes at 1 even if 500 rows have NeedsCreate
```

## The fix

Wrap the fold in `(with-unique-by [Key] ...)` where `Key` is a unique per-row column:

```oql
(Qo.Db.Prop/prop-val! "FolderId" "PageId" "NeedsCreate" NeedsCreate)
(with-unique-by [RowId]
  (fold 0 NeedsCreate + Sum))  ; ✅ Sum = count of rows with NeedsCreate = "true"
```

Or count rows directly with a literal `1` instead of a column:

```oql
(with-table-if [RowId NeedsCreate] (= NeedsCreate "true")
  (fold 0 1 + Count))
```

## Why

OQL's `fold` operates on the *multiset* of values that the symbol resolves to in the current solution. For a tabular input, a symbol resolving to `"true"` across 500 rows still has only one unique value, so the fold iterates once.

`with-unique-by` projects the solution to one row per unique `Key`, which is usually what you want for per-row aggregation.

## Related

- `omega-docs/omegadb/language/` — documentation of solutions and uniqueness semantics.
````

- [ ] **Step 5: Create `batch-size-dependent-bugs.md`**

Write `~/Development/omega/omega-knowledge-base/projects/query-omega-oql/gotchas/batch-size-dependent-bugs.md`:

````markdown
# Batch-size-dependent bugs

**Symptom:** A query works at `limit=1` but fails at `limit=100`. Or succeeds at `limit=100` and fails at `limit=1000`. The failure is some subtle row-drop, unbound-symbol, or write-table error that doesn't reproduce at smaller sizes.

**Root cause:** Several OQL constructs have behavior that depends on row count:

- **`with-table-if` schema collapse** ([see](with-table-if-schema-collapse.md)) only bites when the collapse's merge behavior matters — and "matters" depends on how many rows share schema values. At `limit=1` the collapse is invisible.
- **`fold` over unique values** ([see](fold-folds-unique-values.md)) at `limit=1` has one row with one value, and `Sum = 1` looks correct by coincidence.
- **Mixed-literal symbols** ([see](mixed-literal-throw-run-page.md)) may fail only when the literal actually differs across rows.
- **422 / non-2xx responses** ([see](four-twenty-two-falls-through-dispatch.md)) only surface when a batch contains a row that triggers them.

## The rule

**Always test at realistic batch sizes before declaring working.** If your pipeline runs at 100 rows per batch, test at 100 — not at 1.

## Diagnostic sequence when a batch-size bug appears

1. Reproduce at `limit=1`. Does it work? (Usually yes.)
2. Reproduce at `limit=100`. Does it break? Binary-search down to the smallest batch size that reproduces.
3. Identify the row that triggers the break at that batch size. Is it a specific row's data, or is it an interaction effect across rows?
4. If it's a specific row: the bug is probably data-shape (a 422, a bad value, an empty field).
5. If it's an interaction effect: the bug is probably one of the schema-collapse / fold / mixed-literal footguns listed above.

## Related

- All the footguns linked above.
````

- [ ] **Step 6: Update `gotchas/README.md` index**

Edit `~/Development/omega/omega-knowledge-base/projects/query-omega-oql/gotchas/README.md`:

Replace the `(Entries added in Task 5 and Task 6 of Plan 1.)` line with the five-entry index:

```markdown
# query-omega-oql · gotchas

OQL language footguns — things that bit us and cost hours to diagnose. Read these *before* debugging an OQL surprise.

- [with-table-if-schema-collapse.md](with-table-if-schema-collapse.md) — silent row drops when the branch schema lacks a unique column
- [defined-is-universal.md](defined-is-universal.md) — `(defined X)` is effectively `WHERE true`
- [mixed-literal-throw-run-page.md](mixed-literal-throw-run-page.md) — symbols don't resolve inside literal map/list arguments
- [fold-folds-unique-values.md](fold-folds-unique-values.md) — `fold` folds over unique values, not rows; need `with-unique-by`
- [batch-size-dependent-bugs.md](batch-size-dependent-bugs.md) — bugs that reproduce at `limit=100` but not `limit=1`
```

- [ ] **Step 7: Verify and commit**

Run:
```bash
ls ~/Development/omega/omega-knowledge-base/projects/query-omega-oql/gotchas/ && \
git -C ~/Development/omega/omega-knowledge-base add projects/query-omega-oql/gotchas/ && \
git -C ~/Development/omega/omega-knowledge-base commit -m "docs: seed OQL gotchas (batch 1/2) — schema/defined/literals/fold/batch-size"
```

Expected: 5 new files + README index update.

---

## Task 6: Seed OQL gotchas (batch 2 of 2 — files 6–9)

**Files:**
- Create: `~/Development/omega/omega-knowledge-base/projects/query-omega-oql/gotchas/page-query-skip-on-include-docs.md`
- Create: `~/Development/omega/omega-knowledge-base/projects/query-omega-oql/gotchas/row-drop-idiom.md`
- Create: `~/Development/omega/omega-knowledge-base/projects/query-omega-oql/gotchas/event-subscriptions-die-on-jvm-restart.md`
- Create: `~/Development/omega/omega-knowledge-base/projects/query-omega-oql/gotchas/four-twenty-two-falls-through-dispatch.md`
- Modify: `~/Development/omega/omega-knowledge-base/projects/query-omega-oql/gotchas/README.md` (extend index)

- [ ] **Step 1: Create `page-query-skip-on-include-docs.md`**

Write `~/Development/omega/omega-knowledge-base/projects/query-omega-oql/gotchas/page-query-skip-on-include-docs.md`:

````markdown
# `page-query` CouchDB `skip` is O(skip) with `include_docs=true`

**Symptom:** `page-query` gets dramatically slower at high skip values. Early batches (skip ~1,000) take ~1 second each. Later batches (skip ~95,000) take 30+ seconds each, dominating pipeline runtime.

**Root cause:** Under the hood, `page-query` was issuing a CouchDB Mango query with both `skip=N` and `include_docs=true`. CouchDB's Mango skip implementation is `O(skip)` — it walks the index from 0 to N and discards the first N results. With `include_docs=true`, each skipped row also materializes its document body, so the skip cost is O(skip × doc_size).

This isn't a bug in OQL so much as a bug in how `page-query` composed its CouchDB call. The fix was to **split the query into two steps** — first a view lookup for just the `_id`s (fast, skip-scaled only by index entries), then a bulk fetch via `_all_docs?keys=[...]&include_docs=true` to materialize the bodies.

## The fix

Landed in commit `5365d92`:

- **File:** `omega-db/src/omega_db/datastore/couchdb/data_types/query.clj`
- **Change:** `page-query` now does a view query for `_id` only, then `_all_docs?keys=[...]` for bodies.

If you see late-page slowness again, this is a regression in the same spot. Check that the two-step pattern is still in place.

## How to diagnose slow `page-query`

1. Check the pipeline's `ghl-sync:done` log entries. If skip goes up but batch duration grows roughly linearly with skip, you're hitting O(skip).
2. Inspect the relevant `query.clj` to confirm the two-step pattern is present.
3. If the one-step `include_docs=true` pattern crept back in, revert to the two-step version.

## Related

- The fix commit `5365d92` is the canonical reference. `git show 5365d92` in `omega-db`.
````

- [ ] **Step 2: Create `row-drop-idiom.md`**

Write `~/Development/omega/omega-knowledge-base/projects/query-omega-oql/gotchas/row-drop-idiom.md`:

````markdown
# The `(= true false)` row-drop idiom

**Not a bug — a pattern.** How to deliberately drop rows from a solution table in OQL.

**Use case:** Inside a `with-table-if` then-branch, you want to unify-fail so the rows in the branch are excluded from the downstream solution. There's no explicit "drop this row" primitive — you do it by forcing a unification failure.

## The idiom

```oql
(with-table-if [RowId HttpStatus] (= HttpStatus 422)
  (= true false))
```

`(= true false)` is a unification that cannot succeed. The solver records failure and the row is excluded from the downstream table.

## Why this works

OQL's solution model is relational: rows that fail unification are simply not in the solution. `(= true false)` is the shortest failing unification — any forced-fail expression works, but `(= true false)` is the idiom because it reads as "make this branch unsatisfiable."

## When to use it

Any time you want a `with-table-if` branch to drop rows rather than write them. Typical use cases:

- Filtering out HTTP responses that shouldn't be processed (e.g. 422s — see [four-twenty-two-falls-through-dispatch.md](four-twenty-two-falls-through-dispatch.md))
- Excluding rows that match a sentinel value from an otherwise-writable solution
- Short-circuiting a branch during debugging

## Anti-patterns

- Don't do `(= true false)` in a `with-table-if` *else*-branch — the else fires when the condition is false, and you usually want the else to pass rows through, not drop them.
- Don't do it in the top-level body of a query — you'll drop the entire solution.
- Don't use it where you actually want to raise an error. `(throw ...)` is the error primitive; `(= true false)` is a silent drop.

## Related

- [four-twenty-two-falls-through-dispatch.md](four-twenty-two-falls-through-dispatch.md) — the canonical use case from HDC debugging.
````

- [ ] **Step 3: Create `event-subscriptions-die-on-jvm-restart.md`**

Write `~/Development/omega/omega-knowledge-base/projects/query-omega-oql/gotchas/event-subscriptions-die-on-jvm-restart.md`:

````markdown
# Event subscriptions die on JVM restart

**Symptom:** A long-running pipeline driven by event subscriptions (e.g. the HDC coordinator event loop) stops making progress immediately after a deploy or JVM restart. Logs show no errors — the pipeline simply stops emitting continuation events.

**Root cause:** OQL event subscriptions live in JVM memory only. There is no persistence primitive that would let a subscription survive a JVM restart. When the JVM recycles (deploy, crash, manual restart), every subscription is lost and must be re-established.

## Impact

Any pipeline that uses the `emit-event` / `on-event` pattern to drive a continuation loop **stops** on deploy. The server comes back up, the coordinator page is reachable, but nobody is listening for the events that would fire the next batch.

## The recovery procedure

Manually re-trigger the coordinator via one of:

1. **Re-emit the starting event** through the `emit-event` pathway for that pipeline's coordinator.
2. **Run the coordinator `on-event` directly** with the starting args (bypasses the event bus).
3. **Push-and-run scratch** — the HDC debugging pattern: a scratch OQL file that imports the coordinator impl and calls `on-event` with bootstrap args. See `wttc/homes-dot-com/oql/scratch/push-and-run-ghl-sync.oql` for an example.

## How to detect it

- Pipeline was running, you deployed, logs went quiet.
- `ghl-sync:start` log entries stopped shortly after the deploy timestamp.
- The coordinator page exists and can be manually run.

## Why this exists

Subscriptions are registered in a runtime-only map inside the omega-db process. Moving them to persistent storage would make them part of the data model, which is a much larger design change. For now, the accepted workaround is: **if you deploy, you re-kick the pipeline**.

## Related

- `wttc/homes-dot-com/README.md` — HDC pipeline shape including the manual re-kick procedure.
- Future work: a "resume-on-boot" primitive in omega-db would remove this class of bug.
````

- [ ] **Step 4: Create `four-twenty-two-falls-through-dispatch.md`**

Write `~/Development/omega/omega-knowledge-base/projects/query-omega-oql/gotchas/four-twenty-two-falls-through-dispatch.md`:

````markdown
# 422 HTTP responses fall through 2xx/400 dispatch

**Symptom:** A pipeline that dispatches on HTTP status codes (`2xx` success path, `4xx` error path) silently leaks 422 responses into the success path. Downstream, the leaked rows trip a `WRITING_SYMBOLS` error in `write-table` because their success-path symbols are unbound.

**Root cause:** A dispatch that explicitly handles 2xx and 400 doesn't match 422. A `with-table-if (or (<= 200 HttpStatus 299) (= 400 HttpStatus))` branch leaves 422 (and every other non-200/400 status) unmatched. Those rows fall through to the downstream table with their branch-scoped symbols unbound.

`write-table` then tries to materialize those rows, can't resolve the unbound symbols, and errors with `WRITING_SYMBOLS`.

## The HDC trigger

The GHL API returns **422 Unprocessable Entity** for malformed email rows (invalid format, missing required fields). The HDC `ghl-sync` pipeline originally handled 2xx (success) and 400 (known client error) but not 422. Any row with a bad email would:

1. Make the HTTP call via `http-request`.
2. Get HttpStatus=422 back.
3. Fall through the `with-table-if` 2xx branch (not matched).
4. Fall through the `with-table-if` 400 branch (not matched).
5. Reach the `write-table` call with unbound symbols.
6. `write-table` errors with `WRITING_SYMBOLS`, pipeline halts.

## The fix

Add an explicit 422 filter *before* the write gate, using [the row-drop idiom](row-drop-idiom.md):

```oql
; ... http-request produces HttpStatus ...

(with-table-if [AgentLink HttpStatus] (= HttpStatus 422)
  (= true false))  ; drop 422 rows

; ... normal 2xx / 400 dispatch ...

(write-table ...)
```

The 422 rows are dropped before they reach the dispatch, so they never leak into the success path.

## General rule

Any dispatch on HTTP status **must** either:

1. Explicitly enumerate every status your API can return (2xx, 4xx known, 4xx unknown, 5xx), and handle each with either a write or a drop.
2. Have a catch-all `drop every row where HttpStatus is not 200` filter before the dispatch.

The previous debugging session spent hours chasing a false "write-table throws" theory before finding that the real bug was 422 leakage into the success path.

## How to diagnose `WRITING_SYMBOLS` errors

1. Check the failing rows — are any of their symbols unbound?
2. If yes, trace back to where those symbols should have been bound. Usually that's a `with-table-if` branch.
3. If the branch condition doesn't match those rows, you have a dispatch gap. Add a drop.

## Related

- [row-drop-idiom.md](row-drop-idiom.md) — the `(= true false)` pattern used in the fix.
- [with-table-if-schema-collapse.md](with-table-if-schema-collapse.md) — a different silent-row-drop failure mode in the same clause.
- `wttc/homes-dot-com/impl/18f2e4bb-homes-com-component-library/a31e2741-events-library/8e5821d0-ghl-sync/bfe95f5a-implementation.oql` — the final working `ghl-sync` implementation with the 422 filter in place.
````

- [ ] **Step 5: Update `gotchas/README.md` index to include all 9 entries**

Replace `~/Development/omega/omega-knowledge-base/projects/query-omega-oql/gotchas/README.md` with:

```markdown
# query-omega-oql · gotchas

OQL language footguns — things that bit us and cost hours to diagnose. Read these *before* debugging an OQL surprise.

## Language semantics

- [with-table-if-schema-collapse.md](with-table-if-schema-collapse.md) — silent row drops when the branch schema lacks a unique column
- [defined-is-universal.md](defined-is-universal.md) — `(defined X)` is effectively `WHERE true`
- [mixed-literal-throw-run-page.md](mixed-literal-throw-run-page.md) — symbols don't resolve inside literal map/list arguments
- [fold-folds-unique-values.md](fold-folds-unique-values.md) — `fold` folds over unique values, not rows; need `with-unique-by`
- [row-drop-idiom.md](row-drop-idiom.md) — the `(= true false)` pattern for deliberately dropping rows

## Testing discipline

- [batch-size-dependent-bugs.md](batch-size-dependent-bugs.md) — bugs that reproduce at `limit=100` but not `limit=1`

## Runtime / infrastructure

- [page-query-skip-on-include-docs.md](page-query-skip-on-include-docs.md) — CouchDB Mango `skip` is O(skip) with `include_docs=true`; two-step fix
- [event-subscriptions-die-on-jvm-restart.md](event-subscriptions-die-on-jvm-restart.md) — subscriptions die on deploy, pipelines must be manually re-kicked

## Integration

- [four-twenty-two-falls-through-dispatch.md](four-twenty-two-falls-through-dispatch.md) — 422 responses leak into success path and cause `WRITING_SYMBOLS`
```

> **Note:** 9 entries now puts this bucket at or above the ~7 entries promotion threshold. A fill-gap or rebalance agent may later propose promoting this into a subtree (e.g., `gotchas/language-semantics/`, `gotchas/runtime/`, `gotchas/integration/`). For Plan 1 we leave it flat — the grouping headings inside the README are sufficient navigation for 9 entries.

- [ ] **Step 6: Verify and commit**

Run:
```bash
ls ~/Development/omega/omega-knowledge-base/projects/query-omega-oql/gotchas/ && \
wc -l ~/Development/omega/omega-knowledge-base/projects/query-omega-oql/gotchas/*.md && \
git -C ~/Development/omega/omega-knowledge-base add projects/query-omega-oql/gotchas/ && \
git -C ~/Development/omega/omega-knowledge-base commit -m "docs: seed OQL gotchas (batch 2/2) — page-query/row-drop/subscriptions/422"
```

Expected: 4 new files + README updated.

---

## Task 7: Navigator skill

**Files:**
- Create: `~/Development/omega/omega-knowledge-base/skills/omega-docs-navigator/SKILL.md`

- [ ] **Step 1: Create the skills directory in the repo**

Run:
```bash
mkdir -p ~/Development/omega/omega-knowledge-base/skills/omega-docs-navigator
```

- [ ] **Step 2: Write the navigator SKILL.md**

Write `~/Development/omega/omega-knowledge-base/skills/omega-docs-navigator/SKILL.md`:

````markdown
---
name: omega-docs-navigator
description: Use at session start or whenever you need to orient in the Omega knowledge base. Loads the root README, presents navigation rules, announces companion skills. Invoke this before answering any question about Omega, OQL, or any project in the Omega ecosystem.
---

# Omega Docs Navigator

You are working in the Omega ecosystem. This skill orients you in the knowledge base.

## What to do right now

1. **Read the knowledge base root README.** Use the Read tool on:
   ```
   ~/Development/omega/omega-knowledge-base/README.md
   ```
   It contains the project list, taxonomy summary, and links to contributor rules. If the file does not exist, the knowledge base has not been installed yet — mention this to the user and stop. Do not try to create it.

2. **Commit to the descend-from-root navigation rule for this session.** The rule is below. Read it, then apply it for every Omega-related lookup until the session ends.

3. **Remember the companion skills:** `omega:omega-docs-mark-gap` and `omega:omega-docs-fill-gap`. You will invoke mark-gap any time a predicted path fails.

## Descend-from-root — the navigation rule

You MUST follow this rule when looking up anything in the Omega knowledge base:

1. **Start at** `~/Development/omega/omega-knowledge-base/README.md`. Always. Never jump to a leaf.
2. **Identify scope.** Which project is the question about? Pick a `projects/<name>` entry (or a `contributor/` topic if the question is about the knowledge base itself).
3. **Identify axis** using the axis decision rule (below) — `how-to`, `reference`, `explanation`, or `gotchas`.
4. **Walk the parent chain.** Read each `README.md` from the root down to the target — the root, then the scope's `README.md`, then the axis's `README.md`, then the target file. Parent READMEs carry context that changes how you read leaves.
5. **Construct the predicted path:** `projects/<scope>/<axis>/<topic>.md`.
6. **If the path resolves** → read the target. Done.
7. **If the path does not resolve or the content is insufficient** → invoke `omega:omega-docs-mark-gap` BEFORE falling back to code.

## Axis decision rule

| Phrasing of Q | Axis |
|---|---|
| "How do I…" / "How should I…" | `how-to/` |
| "What is…" / "Where is the X ID" / "What's the schema for…" | `reference/` |
| "Why does…" / "How does … work" / "What's the design of…" | `explanation/` |
| "Why did X fail" / "Why won't X work" / "What happened when I…" | `gotchas/` |

Two edge cases:

- **"How does X work"** is `explanation/`, not `how-to/`. Mechanism = explanation. Commands to run = how-to.
- **"Why does this bug happen"** is `gotchas/`, not `explanation/`. Things that bit us = gotchas. Design itself = explanation.

## Non-negotiables

- **Never jump directly to a leaf.** Always walk the parent chain.
- **Never fall back to grep/code without first marking a gap.**
- **Never mark a gap without a complete parent chain** — mark-gap will refuse.

## When this skill does NOT fire

- You are working outside the Omega ecosystem (no cwd under `~/Development/omega/` or `~/Development/wttc/`, and the user's question is not about Omega).
- You are already mid-descent from a prior invocation in the same session and just need to read the next level.

## Companion skills

- `omega:omega-docs-mark-gap` — record a gap you hit. Batch multiple gaps in one invocation when possible.
- `omega:omega-docs-fill-gap` — fill a gap by reading research sources and writing the missing doc. Runs in background.
````

- [ ] **Step 3: Verify the file**

Run:
```bash
cat ~/Development/omega/omega-knowledge-base/skills/omega-docs-navigator/SKILL.md | head -20
```

Expected: frontmatter with `name: omega-docs-navigator` + the body content.

- [ ] **Step 4: Commit**

Run:
```bash
git -C ~/Development/omega/omega-knowledge-base add skills/omega-docs-navigator/ && \
git -C ~/Development/omega/omega-knowledge-base commit -m "skill: add omega-docs-navigator"
```

---

## Task 8: Mark-gap skill

**Files:**
- Create: `~/Development/omega/omega-knowledge-base/skills/omega-docs-mark-gap/SKILL.md`

- [ ] **Step 1: Create directory and write SKILL.md**

Run:
```bash
mkdir -p ~/Development/omega/omega-knowledge-base/skills/omega-docs-mark-gap
```

Write `~/Development/omega/omega-knowledge-base/skills/omega-docs-mark-gap/SKILL.md`:

````markdown
---
name: omega-docs-mark-gap
description: Record one or more gaps in the Omega knowledge base when a predicted doc path didn't exist or the content was insufficient. Validates parent chain and spawns fill-gap agents. Invoke this BEFORE falling back to code for an Omega-related question.
---

# Omega Docs Mark Gap

Invoke this skill when you looked up something in the Omega knowledge base and the predicted path didn't resolve (or resolved to stub content). Batch as many gaps as you've accumulated in a single invocation — don't stream.

## Inputs

You need, for each gap:

- **`question`** — the question you were trying to answer (paraphrased, 1 line)
- **`predicted_path`** — the path you tried, relative to `~/Development/omega/omega-knowledge-base/` (no leading slash)
- **`parent_chain`** — the list of `README.md` paths you read during descent. MUST cover the full depth of `predicted_path`.
- **`fallback_source`** — what you read instead: file path, command, or the string `"none"`.

## Procedure

1. **Validate parent chain.** For each gap, count the path segments in `predicted_path` (split on `/`). The `parent_chain` must have at least one `README.md` per segment, starting from the root. If it doesn't, STOP and complete the descent first — this skill refuses to record gaps without the full chain because it breaks descend-from-root discipline.

2. **Compute the key.** For each gap, compute:
   ```python
   import hashlib
   key = hashlib.sha1((predicted_path + "|" + question).encode()).hexdigest()[:8]
   ```
   Use Python (via the Bash tool) or reproduce the equivalent in shell with `sha1sum`:
   ```bash
   printf '%s|%s' "$predicted_path" "$question" | sha1sum | cut -c1-8
   ```

3. **Check for duplicates.** Read `~/Development/omega/omega-knowledge-base/gaps.md`. If a `gap-<key>` entry already exists, skip that gap — the existing entry represents the same thing.

4. **Append new gaps** to `gaps.md` using the Edit tool. Each entry follows this format, separated by `---`:

   ```markdown
   ## gap-<key>

   - **question**: <question>
   - **predicted-path**: <predicted_path>
   - **parent-chain**:
     - <path-1>
     - <path-2>
     - <path-3>
   - **fallback-source**: <fallback_source>
   - **recorded-at**: <ISO 8601 UTC, e.g. 2026-04-12T15:30:00Z>
   - **status**: open
   - **claimed-by**:
   - **notes**:

   ---
   ```

5. **Commit the new gaps:**
   ```bash
   git -C ~/Development/omega/omega-knowledge-base add gaps.md
   git -C ~/Development/omega/omega-knowledge-base commit -m "gap: record <N> gap(s)"
   ```

6. **Spawn fill-gap agents** via the Agent tool's `run_in_background: true` parameter. Dispatch-count rules:
   - **1 gap** → 1 agent
   - **2–3 gaps** → 1–2 agents (sequential usually fine; you can also batch 2–3 into one agent)
   - **4–5 gaps** → 2–3 agents in parallel
   - **6+ gaps** → cap at 3 parallel; dispatch the first 3, let the caller re-invoke mark-gap for the rest later (or record them as open and rely on a later invocation)

   **Each agent is assigned a specific gap key**, not "pick from the queue." Pass the key in the Agent prompt.

## Agent dispatch prompt template

When you dispatch a fill-gap agent, use a prompt like:

```
Invoke the omega:omega-docs-fill-gap skill with gap_key=<key>. The gap was recorded in ~/Development/omega/omega-knowledge-base/gaps.md. Research the answer from the fallback source, write the missing doc at the predicted path (or a better-fitting path), update the parent README, and delete the gap entry. Commit atomically with message "docs: fill gap <key> — <topic>". If you cannot find the answer in the fallback source, bail out with the gap left open and commit a note — do not write speculation.
```

## Refuse if

- **Parent chain is incomplete** — shorter than predicted_path depth.
- **Predicted path is outside** `omega-knowledge-base/` or a project's `/docs/` tree. We don't track gaps in arbitrary code.
- **Predicted path is absolute** or contains `..` — must be relative to the knowledge base root.

## After marking

You can now fall back to reading code or commits to answer the original question. The fill-gap agents are running in the background — do not block on them.
````

- [ ] **Step 2: Verify**

Run:
```bash
head -5 ~/Development/omega/omega-knowledge-base/skills/omega-docs-mark-gap/SKILL.md
```

Expected: frontmatter with `name: omega-docs-mark-gap`.

- [ ] **Step 3: Commit**

Run:
```bash
git -C ~/Development/omega/omega-knowledge-base add skills/omega-docs-mark-gap/ && \
git -C ~/Development/omega/omega-knowledge-base commit -m "skill: add omega-docs-mark-gap"
```

---

## Task 9: Fill-gap skill

**Files:**
- Create: `~/Development/omega/omega-knowledge-base/skills/omega-docs-fill-gap/SKILL.md`

- [ ] **Step 1: Create directory and write SKILL.md**

Run:
```bash
mkdir -p ~/Development/omega/omega-knowledge-base/skills/omega-docs-fill-gap
```

Write `~/Development/omega/omega-knowledge-base/skills/omega-docs-fill-gap/SKILL.md`:

````markdown
---
name: omega-docs-fill-gap
description: Read a gap from the Omega knowledge base gaps.md, research the answer, write the missing doc, and clean up the entry. Designed to run as a background agent dispatched by omega-docs-mark-gap, but can also be invoked manually. Atomic commits, never writes code.
---

# Omega Docs Fill Gap

You were dispatched (or manually invoked) to fill a documentation gap in the Omega knowledge base.

## Arguments

- **`gap_key`** — the 8-char hash of the gap to fill. If not specified, pick the oldest `open` gap in `gaps.md`.

## Procedure

### 1. Read gaps.md and locate the gap

Read `~/Development/omega/omega-knowledge-base/gaps.md`. Find the `## gap-<gap_key>` entry. If the entry is missing, another agent already resolved it — bail out as a no-op.

### 2. Atomically claim the gap

Edit the entry to set:
- `status: claimed`
- `claimed-by: <your session id, or "agent">`

Commit:
```bash
git -C ~/Development/omega/omega-knowledge-base add gaps.md
git -C ~/Development/omega/omega-knowledge-base commit -m "gap: claim <gap_key>"
```

If this commit fails because the file changed under you (another agent claimed it first), bail out as a no-op. **Do not retry.** Better to lose a race than double-write.

### 3. Rebuild the context

Read each file in the gap's `parent-chain`. This reconstructs the reading mind of the original agent at the point of failure. Walking the chain ensures you write the missing doc in the same context the reader will find it in.

### 4. Research the answer

Read the `fallback-source` — the file path, command output, or other source the original agent fell back to. This is where the answer lives.

Also read:
- **Sibling docs** in the predicted path's parent directory for style consistency (e.g., if filling `projects/omega-db/how-to/deploy.md`, read the other files in `projects/omega-db/how-to/`).
- **Related docs** referenced by the fallback source.

**Do NOT speculate.** If the fallback source does not contain the answer, bail out in step 9.

### 5. Write the missing doc

Two cases:

**5a. Prediction was structurally correct.** Write the doc at `predicted_path`. Match the style of sibling docs.

**5b. Prediction was structurally wrong** (e.g., the topic belongs in `explanation/` but was predicted in `how-to/`). Write the doc at the path that correctly fits the taxonomy (see `contributor/reference/path-prediction-rules.md` if you need to recheck axis rules). If you choose a different path, you MUST also update the parent README of the new path to index the new doc.

### 6. Update the parent README

The immediate parent directory of the new doc has a `README.md` that lists its children. Add a line for the new doc with a one-line description:

```markdown
- [new-doc.md](new-doc.md) — short description
```

Keep the list alphabetical or grouped by subtopic, matching the existing style of that README.

### 7. Check rebalancing triggers

Count the entries in the bucket you just added to. If that bucket now has **more than ~7 entries** AND one file is clearly the biggest (screen-plus of content), consider promoting it to a subtree per `contributor/how-to/rebalance-a-subtree.md`.

- **If the trigger is clear**, execute the rebalance: create the subtree, move the fattest file(s), update all intra-kb links to them, update parent READMEs.
- **If it's marginal**, skip for this invocation — the next gap fill may clarify the shape.

### 8. Delete the gap entry

Edit `gaps.md` to remove the `## gap-<gap_key>` block entirely, along with its trailing `---`. **Resolved gaps are deleted, not marked.** History lives in git.

### 9. Commit atomically

All changes in ONE commit:

```bash
git -C ~/Development/omega/omega-knowledge-base add \
  <new-doc-path> \
  <parent-readme-path> \
  gaps.md
git -C ~/Development/omega/omega-knowledge-base commit -m "docs: fill gap <gap_key> — <topic>"
```

If you did a rebalance as part of step 7, include all the moved/updated files in the same commit.

### Bail out (no-op) if

- The gap entry is missing from `gaps.md` when you try to claim it (step 1).
- The claim commit fails due to a race (step 2).
- The fallback source does not contain the answer (step 4). In this case, revert the claim: edit the entry to set `status: open` and clear `claimed-by`, commit with message `gap: release <gap_key> (no source)`, and end.

## Safety rails — absolutely mandatory

- **NEVER write code.** Only write files inside `~/Development/omega/omega-knowledge-base/` or inside a project's `/docs/` tree.
- **NEVER remove a gap** without writing the corresponding doc.
- **NEVER split** the gap fill and doc write across multiple commits — the gap deletion and the new doc must land in the same commit.
- **NEVER guess.** Bail out of uncertainty. Better to leave the gap open than write wrong docs.
- **NEVER touch files outside the knowledge base** except to *read* them as research.
````

- [ ] **Step 2: Verify**

Run:
```bash
head -5 ~/Development/omega/omega-knowledge-base/skills/omega-docs-fill-gap/SKILL.md
```

Expected: frontmatter with `name: omega-docs-fill-gap`.

- [ ] **Step 3: Commit**

Run:
```bash
git -C ~/Development/omega/omega-knowledge-base add skills/omega-docs-fill-gap/ && \
git -C ~/Development/omega/omega-knowledge-base commit -m "skill: add omega-docs-fill-gap"
```

---

## Task 10: Hook + install scripts

**Files:**
- Create: `~/Development/omega/omega-knowledge-base/scripts/omega-session-start.sh`
- Create: `~/Development/omega/omega-knowledge-base/scripts/install-skills.sh`
- Create: `~/Development/omega/omega-knowledge-base/scripts/install-hook.sh`

- [ ] **Step 1: Create scripts directory**

Run:
```bash
mkdir -p ~/Development/omega/omega-knowledge-base/scripts
```

- [ ] **Step 2: Write `omega-session-start.sh`**

Write `~/Development/omega/omega-knowledge-base/scripts/omega-session-start.sh`:

```bash
#!/usr/bin/env bash
# SessionStart hook for Claude Code. Fires when a session starts in an
# Omega-ecosystem directory. Emits an additionalContext block instructing
# the agent to invoke omega:omega-docs-navigator.
#
# Installed at: ~/.claude/hooks/omega-session-start.sh
# Registered in: ~/.claude/settings.json under hooks.SessionStart
#
# The script is a silent no-op in any of these cases:
#   - cwd is not under ~/Development/omega/ or ~/Development/wttc/
#   - the knowledge base repo does not exist yet
#
# Output (when firing) is a single JSON object on stdout, parsed by
# Claude Code as SessionStart hook output.

set -euo pipefail

cwd="$(pwd)"
kb_root="$HOME/Development/omega/omega-knowledge-base"

# Silent no-op unless we're in the Omega ecosystem.
case "$cwd" in
  "$HOME/Development/omega/"*|"$HOME/Development/wttc/"*)
    ;;
  *)
    exit 0
    ;;
esac

# Silent no-op if the knowledge base hasn't been installed yet.
if [ ! -f "$kb_root/README.md" ]; then
  exit 0
fi

cat <<'JSON'
{
  "hookSpecificOutput": {
    "hookEventName": "SessionStart",
    "additionalContext": "Before answering any question about Omega, OQL, or any project in the Omega ecosystem, invoke the omega:omega-docs-navigator skill. It will orient you in the knowledge base tree at ~/Development/omega/omega-knowledge-base and establish the descend-from-root navigation rules for this session."
  }
}
JSON
```

- [ ] **Step 3: Make it executable and smoke-test**

Run:
```bash
chmod +x ~/Development/omega/omega-knowledge-base/scripts/omega-session-start.sh
```

Test 1 — from an Omega cwd, verify the JSON fires:

```bash
cd ~/Development/omega/omega-knowledge-base && \
~/Development/omega/omega-knowledge-base/scripts/omega-session-start.sh
```

Expected output: a JSON object with `hookSpecificOutput` and an `additionalContext` value mentioning `omega:omega-docs-navigator`.

Test 2 — from a non-Omega cwd, verify silence:

```bash
cd /tmp && \
~/Development/omega/omega-knowledge-base/scripts/omega-session-start.sh && \
echo "silent no-op OK"
```

Expected output: `silent no-op OK` (and nothing from the script itself).

- [ ] **Step 4: Write `install-skills.sh`**

Write `~/Development/omega/omega-knowledge-base/scripts/install-skills.sh`:

```bash
#!/usr/bin/env bash
# Install the omega:* skills from the knowledge base into ~/.claude/skills/omega/.
#
# Copies each skills/omega-docs-*/SKILL.md into ~/.claude/skills/omega/<name>/SKILL.md.
# Re-run this after editing any skill in the knowledge base.

set -euo pipefail

kb_root="$HOME/Development/omega/omega-knowledge-base"
target_root="$HOME/.claude/skills/omega"

if [ ! -d "$kb_root/skills" ]; then
  echo "ERROR: $kb_root/skills not found — is the knowledge base initialized?" >&2
  exit 1
fi

mkdir -p "$target_root"

for skill_dir in "$kb_root"/skills/*/; do
  skill_name="$(basename "$skill_dir")"
  target_dir="$target_root/$skill_name"
  mkdir -p "$target_dir"
  cp "$skill_dir/SKILL.md" "$target_dir/SKILL.md"
  echo "installed: $skill_name -> $target_dir/SKILL.md"
done

echo
echo "Done. Skills available at $target_root/"
ls -la "$target_root"
```

- [ ] **Step 5: Write `install-hook.sh`**

Write `~/Development/omega/omega-knowledge-base/scripts/install-hook.sh`:

```bash
#!/usr/bin/env bash
# Install the SessionStart hook script into ~/.claude/hooks/.
# Does NOT modify ~/.claude/settings.json — registration is a manual step
# documented in the plan (Task 11) because settings.json may already have
# other hooks that shouldn't be touched.

set -euo pipefail

kb_root="$HOME/Development/omega/omega-knowledge-base"
target_dir="$HOME/.claude/hooks"
src="$kb_root/scripts/omega-session-start.sh"
target="$target_dir/omega-session-start.sh"

if [ ! -f "$src" ]; then
  echo "ERROR: $src not found" >&2
  exit 1
fi

mkdir -p "$target_dir"
cp "$src" "$target"
chmod +x "$target"

echo "installed: $target"
echo
echo "NEXT STEP: add the hook entry to ~/.claude/settings.json under hooks.SessionStart."
echo "See $kb_root/docs/superpowers/plans/2026-04-12-omega-knowledge-base-plan1-core.md Task 11."
```

- [ ] **Step 6: Make install scripts executable**

Run:
```bash
chmod +x ~/Development/omega/omega-knowledge-base/scripts/install-skills.sh \
         ~/Development/omega/omega-knowledge-base/scripts/install-hook.sh
```

- [ ] **Step 7: Commit**

Run:
```bash
git -C ~/Development/omega/omega-knowledge-base add scripts/ && \
git -C ~/Development/omega/omega-knowledge-base commit -m "scripts: session-start hook + skill/hook install scripts"
```

---

## Task 11: Install skills and hook, register hook in settings.json

This task modifies files outside the knowledge base repo. No commit — `~/.claude/` is user config.

**Files:**
- Create: `~/.claude/skills/omega/omega-docs-navigator/SKILL.md` (copied from repo)
- Create: `~/.claude/skills/omega/omega-docs-mark-gap/SKILL.md` (copied from repo)
- Create: `~/.claude/skills/omega/omega-docs-fill-gap/SKILL.md` (copied from repo)
- Create: `~/.claude/hooks/omega-session-start.sh` (copied from repo)
- Modify: `~/.claude/settings.json` (add new hook entry alongside existing)

- [ ] **Step 1: Run the skill install script**

Run:
```bash
~/Development/omega/omega-knowledge-base/scripts/install-skills.sh
```

Expected output: three lines of `installed: omega-docs-*`, then a directory listing showing the three skill directories.

- [ ] **Step 2: Verify skill installation**

Run:
```bash
ls ~/.claude/skills/omega/ && \
head -5 ~/.claude/skills/omega/omega-docs-navigator/SKILL.md && \
head -5 ~/.claude/skills/omega/omega-docs-mark-gap/SKILL.md && \
head -5 ~/.claude/skills/omega/omega-docs-fill-gap/SKILL.md
```

Expected: three directories listed, each file's frontmatter header visible.

- [ ] **Step 3: Run the hook install script**

Run:
```bash
~/Development/omega/omega-knowledge-base/scripts/install-hook.sh
```

Expected: `installed: /home/ryan/.claude/hooks/omega-session-start.sh` plus the NEXT STEP reminder.

- [ ] **Step 4: Verify the hook script is installed and executable**

Run:
```bash
ls -la ~/.claude/hooks/omega-session-start.sh && \
cd ~/Development/omega/omega-knowledge-base && ~/.claude/hooks/omega-session-start.sh
```

Expected: the file exists with `+x`, and running it from the kb root prints the JSON hook output.

- [ ] **Step 5: Read current `~/.claude/settings.json`**

Use the Read tool on `~/.claude/settings.json`. The current `hooks.SessionStart` section looks like:

```jsonc
"hooks": {
  "SessionStart": [
    {
      "hooks": [
        {
          "type": "command",
          "command": "CONTENT=$(cat /home/ryan/.claude/projects/-home-ryan-Development-omega-query-omega-oql/memory/project_current_work.md 2>/dev/null) && printf '{\"hookSpecificOutput\":{\"hookEventName\":\"SessionStart\",\"additionalContext\":\"Current work context loaded automatically:\\n%s\"}}\\n' \"$(echo \"$CONTENT\" | jq -Rs . | sed 's/^\"//' | sed 's/\"$//')\"",
          "timeout": 5,
          "statusMessage": "Loading current work context..."
        }
      ]
    }
  ]
}
```

The new hook must be **added alongside** the existing one. Both should fire — the existing one loads session work state, the new one instructs the agent to invoke the navigator skill.

- [ ] **Step 6: Add the new hook entry**

Use the Edit tool to replace:

```jsonc
  "hooks": {
    "SessionStart": [
      {
        "hooks": [
          {
            "type": "command",
            "command": "CONTENT=$(cat /home/ryan/.claude/projects/-home-ryan-Development-omega-query-omega-oql/memory/project_current_work.md 2>/dev/null) && printf '{\"hookSpecificOutput\":{\"hookEventName\":\"SessionStart\",\"additionalContext\":\"Current work context loaded automatically:\\n%s\"}}\\n' \"$(echo \"$CONTENT\" | jq -Rs . | sed 's/^\"//' | sed 's/\"$//')\"",
            "timeout": 5,
            "statusMessage": "Loading current work context..."
          }
        ]
      }
    ]
  },
```

With:

```jsonc
  "hooks": {
    "SessionStart": [
      {
        "hooks": [
          {
            "type": "command",
            "command": "CONTENT=$(cat /home/ryan/.claude/projects/-home-ryan-Development-omega-query-omega-oql/memory/project_current_work.md 2>/dev/null) && printf '{\"hookSpecificOutput\":{\"hookEventName\":\"SessionStart\",\"additionalContext\":\"Current work context loaded automatically:\\n%s\"}}\\n' \"$(echo \"$CONTENT\" | jq -Rs . | sed 's/^\"//' | sed 's/\"$//')\"",
            "timeout": 5,
            "statusMessage": "Loading current work context..."
          },
          {
            "type": "command",
            "command": "~/.claude/hooks/omega-session-start.sh",
            "timeout": 5,
            "statusMessage": "Checking Omega ecosystem context..."
          }
        ]
      }
    ]
  },
```

The two command entries live inside the same `hooks:` array — they both fire on SessionStart. If the cwd is not an Omega directory, the second command is a silent no-op.

- [ ] **Step 7: Verify settings.json is still valid JSON**

Run:
```bash
python3 -c 'import json; json.load(open("/home/ryan/.claude/settings.json")); print("valid")'
```

Expected: `valid`.

- [ ] **Step 8: Fire the hook manually to confirm Claude accepts it**

Run:
```bash
cd ~/Development/omega/omega-knowledge-base && ~/.claude/hooks/omega-session-start.sh
```

Expected: the JSON output appears on stdout. (Actual SessionStart firing only happens when a new Claude Code session boots — that is the smoke test in Task 12.)

---

## Task 12: Smoke test

No files modified in this task — purely verification that the installed system works end-to-end.

- [ ] **Step 1: Verify the knowledge base tree is intact**

Run:
```bash
find ~/Development/omega/omega-knowledge-base -type f -name "*.md" | sort | wc -l && \
find ~/Development/omega/omega-knowledge-base -type f -name "*.md" | sort
```

Expected: approximately 35 markdown files total (root README + gaps + contributor tree + projects tree + OQL gotchas + 3 skills). Count should be stable across runs.

- [ ] **Step 2: Verify every parent directory has a README.md**

Run:
```bash
cd ~/Development/omega/omega-knowledge-base && \
find . -type d -not -path './.git*' -not -path './skills*' -not -path './scripts*' | while read d; do
  if [ ! -f "$d/README.md" ]; then
    echo "MISSING README: $d"
  fi
done && \
echo "done"
```

Expected: `done` with no `MISSING README:` lines. (We exempt `skills/`, `scripts/`, and `.git/` because they aren't part of the descend-from-root tree.)

- [ ] **Step 3: Verify the hook fires in an Omega cwd**

Run:
```bash
cd ~/Development/omega/omega-knowledge-base && \
~/.claude/hooks/omega-session-start.sh | python3 -c 'import json, sys; d = json.load(sys.stdin); assert "additionalContext" in d["hookSpecificOutput"]; print("hook OK:", d["hookSpecificOutput"]["additionalContext"][:60])'
```

Expected: `hook OK: Before answering any question about Omega, OQL, or any pr...`.

- [ ] **Step 4: Verify the hook is silent outside the Omega ecosystem**

Run:
```bash
cd /tmp && \
output=$(~/.claude/hooks/omega-session-start.sh) && \
if [ -z "$output" ]; then echo "silent OK"; else echo "FAIL: got output: $output"; fi
```

Expected: `silent OK`.

- [ ] **Step 5: Manually invoke the navigator skill in this session**

Use the Skill tool: `Skill(skill: "omega:omega-docs-navigator")`.

Expected: the skill loads and presents the navigation rules and a Read on `~/Development/omega/omega-knowledge-base/README.md`. If the skill is not discoverable by the Skill tool, the skill install did not register it — re-run `scripts/install-skills.sh` and restart Claude Code.

(**Note on skill discovery:** Claude Code discovers skills at session start by scanning `~/.claude/skills/`. If the skills were installed mid-session, you may need a new Claude Code session for the Skill tool to list them. This is expected behavior — document but don't treat as a failure.)

- [ ] **Step 6: Dry-run the mark-gap flow**

Without actually invoking the skill, walk through what a mark-gap call would look like with fake inputs to confirm the procedure in `SKILL.md` is concrete enough to execute:

1. Pick a fake question: `"What's the schema for Data/Filtered Agents?"`.
2. Fake predicted path: `projects/homes-dot-com/reference/filtered-agents-schema.md`.
3. Fake parent chain: `["README.md", "projects/README.md", "projects/homes-dot-com.md"]`.
   - **Question:** Does this chain cover the depth of the predicted path? `projects/homes-dot-com/reference/filtered-agents-schema.md` has depth 4 (projects, homes-dot-com, reference, filtered-agents-schema.md). The chain has 3 entries and the third is `projects/homes-dot-com.md` which is a *pointer file*, not a README in a `projects/homes-dot-com/` directory. **The chain is invalid** — it points at a file, not a directory README.
   - **This is the refuse case.** Mark-gap would reject this because homes-dot-com is not yet a subtree (it's a pointer file), so the predicted path can't exist in the tree-as-it-is. The correct action: this is a gap of a *different kind* — "homes-dot-com needs to be promoted to a subtree before we can store a schema doc for it." Record that as the gap, not the schema path.
4. Revised gap: predicted path `projects/homes-dot-com.md` (the existing pointer), question `"Where does the Filtered Agents schema live?"`, fallback source `"wttc/homes-dot-com/README.md"`. This gap is recordable.

The dry-run confirms the skill's validation logic: it refuses gaps when the parent chain doesn't exist yet, and forces the agent to either (a) walk a different chain, or (b) record a different gap.

- [ ] **Step 7: Verify git log**

Run:
```bash
git -C ~/Development/omega/omega-knowledge-base log --oneline
```

Expected: approximately 10 commits — one per task (init, contributor how-to, contributor rest, projects, oql subtree, gotchas batch 1, gotchas batch 2, navigator skill, mark-gap skill, fill-gap skill, scripts).

- [ ] **Step 8: Print summary**

Run:
```bash
cd ~/Development/omega/omega-knowledge-base && \
echo "=== Knowledge base summary ===" && \
echo "Repo: $(pwd)" && \
echo "Commits: $(git rev-list --count HEAD)" && \
echo "Markdown files: $(find . -type f -name '*.md' -not -path './.git*' | wc -l)" && \
echo "Top-level containers: $(ls -d */ | grep -v .git)" && \
echo "Skills installed: $(ls ~/.claude/skills/omega/ 2>/dev/null | wc -l)" && \
echo "Hook installed: $(ls -la ~/.claude/hooks/omega-session-start.sh 2>/dev/null | head -1)" && \
echo "=== Plan 1 complete ==="
```

Expected: a printed summary showing the repo state, commit count, markdown file count, 3 skills, and the hook script.

---

## End of Plan 1

### What Plan 1 delivers

- A new `omega-knowledge-base` git repo at `~/Development/omega/omega-knowledge-base/` with:
  - Contributor tree (how-to, reference, explanation, gotchas) fully populated
  - Projects tree with 9 pointer files + one promoted `query-omega-oql/` subtree
  - `query-omega-oql/gotchas/` populated with 9 OQL gotcha entries from the previous debugging session
  - `gaps.md` empty queue ready to accept entries
  - Skills sources in `skills/omega-docs-*/SKILL.md`
  - Scripts in `scripts/` for installing skills and the session-start hook
- Installed skills at `~/.claude/skills/omega/omega-docs-*/SKILL.md`
- Installed hook script at `~/.claude/hooks/omega-session-start.sh`
- Updated `~/.claude/settings.json` to register the new SessionStart hook alongside the existing one

### What Plan 1 does NOT deliver (scope for Plan 2)

- Migrating `omega-docs` into the Diátaxis+gotchas taxonomy
- Creating per-project `/docs/` trees inside each individual project repo
- Pruning `MEMORY.md` and deleting obsolete memory files
- Setting up a private GitHub remote for `omega-knowledge-base`
- Any content beyond what this plan explicitly seeds
- A Plan-2-level rebalance of the `query-omega-oql/gotchas/` bucket even though it sits at 9 entries (over the 7-entry threshold) — left for a later rebalance pass once more usage signal accumulates

### Open questions deferred from the spec

1. **Exact matching rules for the SessionStart hook's "is this an Omega session" check.** Current implementation uses a simple `cwd` prefix match on `~/Development/omega/` and `~/Development/wttc/`. Refine when edge cases appear (e.g., sessions that start in `~` then cd in).
2. **Where do skills physically live for multi-machine workflows?** Plan 1 puts them in `~/.claude/skills/omega/` on one machine. For shared distribution, a plugin repo would be appropriate — out of scope for Plan 1.
3. **Gap key collision handling.** 8-char hash gives ~4B keys; collisions are astronomically unlikely. Monitor for collisions in practice; extend to 12 chars if one ever appears.
4. **First-time setup.** The hook silently no-ops when the repo is missing. Documented in the script's comment block.
