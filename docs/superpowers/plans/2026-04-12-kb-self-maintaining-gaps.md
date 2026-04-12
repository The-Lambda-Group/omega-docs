# Self-Maintaining Knowledge Base Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Redesign the omega-knowledge-base plugin's gap-filing workflow from a 5-step ceremony to a fire-and-forget single-agent dispatch, with firing rules integrated into the KB's self-similar README structure and KB position awareness enforced via the SessionStart hook.

**Architecture:** The mark-gap skill is simplified to a reference doc. The SessionStart hook injects a dispatch template + firing rules + position-awareness rule into every session. Gap keys use human-readable slugs instead of sha1 hashes. Firing rules live in README.md files at every level of the tree, discovered during normal navigator descent.

**Tech Stack:** Bash (hook script), Markdown (skills + READMEs), Claude Code plugin system

**Spec:** [2026-04-12-kb-self-maintaining-gaps-design.md](../specs/2026-04-12-kb-self-maintaining-gaps-design.md)

---

## All files touched

| File | Action | Task |
|---|---|---|
| `gaps.md` | Modify — update format template to use slugs | 1 |
| `README.md` (root) | Modify — add `## Gap rules` section | 2 |
| `projects/query-omega-oql/README.md` | Modify — add `## Gap rules` section | 3 |
| `projects/query-omega-oql/gotchas/README.md` | Modify — add `## Gap rules` section | 3 |
| `projects/homes-dot-com/README.md` | Modify — add `## Gap rules` section | 3 |
| `projects/gohighlevel.md` | Modify — add `## Gap rules` section | 3 |
| `skills/omega-docs-mark-gap/SKILL.md` | Rewrite | 4 |
| `skills/omega-docs-fill-gap/SKILL.md` | Modify — slug support | 5 |
| `skills/omega-docs-navigator/SKILL.md` | Modify — position awareness + gap rules | 6 |
| `scripts/omega-session-start.sh` | Rewrite — inject template + rules + position awareness | 7 |

All paths relative to `~/Development/omega/omega-knowledge-base/`.

---

## Task 1: Update gaps.md format to use slugs

**File:** `~/Development/omega/omega-knowledge-base/gaps.md`

- [ ] **Step 1: Update the format template**

Change the gap entry format section in `gaps.md` from:

```markdown
## gap-<8-char-hash>
```

To:

```markdown
## gap-<slug>
```

And remove the sha1 computation instructions from the format block. The full replacement for the format block:

```markdown
Gap entry format:

\```
## gap-<slug>

- **question**: <paraphrased question that triggered the lookup>
- **predicted-path**: <path from knowledge-base root, no leading slash>
- **fallback-source**: <what you learned — be specific, include the actual answer>
- **recorded-at**: <ISO 8601 UTC>
- **status**: open | claimed
- **claimed-by**: <agent id or empty>
- **notes**: <optional>

---
\```

Slug derivation: use the filename stem of the predicted path. Example:
- Predicted path: `projects/query-omega-oql/gotchas/fold-is-global-not-per-row.md`
- Slug: `fold-is-global-not-per-row`
- Header: `## gap-fold-is-global-not-per-row`

If a slug already exists in gaps.md, append `-2` (or `-3`, etc.).
```

Note: the `parent-chain` field is removed from the entry format. The fire-and-forget agent handles parent-chain validation internally — the caller doesn't need to track it.

- [ ] **Step 2: Commit**

```bash
git -C ~/Development/omega/omega-knowledge-base add gaps.md
git -C ~/Development/omega/omega-knowledge-base commit -m "chore: update gaps.md format to slug-based keys"
```

---

## Task 2: Add gap rules to root README

**File:** `~/Development/omega/omega-knowledge-base/README.md`

- [ ] **Step 1: Add the gap rules section**

Add the following section after the existing `## Gap queue` section at the bottom of the root README:

```markdown
## Gap rules

File a documentation gap (fire-and-forget, do not stop your current work) when:

- A predicted KB path doesn't resolve — file before falling back to code
- You debug something for >5 minutes that had a simple fix — file a gotcha
- You notice undocumented behavior while working — file without stopping
- You learn something by reading code that should have been in a doc — file
- You make a mistake that a pre-trained model would get wrong without project-specific docs — file
- You get corrected by the user on something basic — file the correction as a gotcha
- A subagent asks a question that the KB should have answered — file

When in doubt, file. The cost of a false positive (a gap that turns out unnecessary) is one background agent that bails. The cost of a false negative (a gap not filed) is repeating the same mistake in a future session.
```

- [ ] **Step 2: Commit**

```bash
git -C ~/Development/omega/omega-knowledge-base add README.md
git -C ~/Development/omega/omega-knowledge-base commit -m "docs: add global gap-filing rules to root README"
```

---

## Task 3: Add gap rules to project READMEs

**Files:**
- `projects/query-omega-oql/README.md`
- `projects/query-omega-oql/gotchas/README.md`
- `projects/homes-dot-com/README.md`
- `projects/gohighlevel.md`

- [ ] **Step 1: Add gap rules to `projects/query-omega-oql/README.md`**

Add after the existing `## gotchas/` section:

```markdown
## Gap rules

In addition to global rules:
- If an OQL primitive behaves differently than Prolog/Datalog semantics predict, file under `gotchas/`
- If a `with-table-if` pattern causes silent row drops or schema collapse, file under `gotchas/`
- If you discover a term that requires specific argument binding order (non-relational), file under `gotchas/`
- If you use `fold` where `append` was needed (or vice versa), file under `gotchas/`
- If an OQL error message is misleading or absent (silent failure), file under `gotchas/`
```

- [ ] **Step 2: Add gap rules to `projects/query-omega-oql/gotchas/README.md`**

Add at the bottom:

```markdown
## Gap rules

File a gotcha here when:
- Debugging revealed a footgun that cost >5 minutes
- An OQL pattern worked at limit=1 but broke at limit=100 (batch-dependent bug)
- A silent failure (no error, just wrong results) was caused by a language semantic
- You spent significant time on something that had a simple, documented answer elsewhere but the gotcha index didn't surface it
```

- [ ] **Step 3: Add gap rules to `projects/homes-dot-com/README.md`**

Add after the `## gotchas/` section:

```markdown
## Gap rules

In addition to global rules:
- If a GHL API behavior surprises you (silent ignores, unexpected status codes), file under `gotchas/`
- If pipeline debugging reveals a data-dependent failure, file under `gotchas/`
- If a pipeline step's interface (args, return shape) differs from what the README says, file as a reference update
```

- [ ] **Step 4: Add gap rules to `projects/gohighlevel.md`**

Add at the bottom:

```markdown
## Gap rules

In addition to global rules:
- If a GHL API endpoint behaves differently than documented (silent field drops, unexpected status codes), file under the GHL connector's gotchas or as a note in the private notes section above
- If a new query needs a response format that differs from the existing pattern, file as a reference update
```

- [ ] **Step 5: Commit**

```bash
git -C ~/Development/omega/omega-knowledge-base add \
  projects/query-omega-oql/README.md \
  projects/query-omega-oql/gotchas/README.md \
  projects/homes-dot-com/README.md \
  projects/gohighlevel.md
git -C ~/Development/omega/omega-knowledge-base commit -m "docs: add project-specific gap-filing rules to READMEs"
```

---

## Task 4: Rewrite mark-gap skill

**File:** `~/Development/omega/omega-knowledge-base/skills/omega-docs-mark-gap/SKILL.md`

- [ ] **Step 1: Replace the entire SKILL.md with the simplified version**

```markdown
---
name: omega-docs-mark-gap
description: Record one or more gaps in the Omega knowledge base when a predicted doc path didn't exist or the content was insufficient. Validates parent chain and spawns fill-gap agents. Invoke this BEFORE falling back to code for an Omega-related question.
---

# Omega Docs Mark Gap

## Preferred path: fire-and-forget dispatch

The SessionStart hook injects a dispatch template and firing rules into every session. The fastest way to file a gap is to dispatch a single background agent:

\```
Agent(
  description="Fill gap: <slug>",
  run_in_background=true,
  prompt="Record and fill a documentation gap in ~/Development/omega/omega-knowledge-base.

Slug: <slug>
Predicted path: <predicted-path>
Question: <question>
Source: <what-you-learned — be specific, include the actual answer if you know it>

Steps:
1. Read gaps.md. If a gap with this slug already exists, stop (duplicate).
2. Append a new entry: ## gap-<slug> with question, predicted-path, fallback-source, recorded-at (ISO 8601 UTC), status: open.
3. git -C ~/Development/omega/omega-knowledge-base add gaps.md && git commit -m 'gap: <slug>'
4. Invoke the omega:omega-docs-fill-gap skill with gap_slug=<slug> to research and write the doc.
5. If the fill succeeds, the fill-gap agent deletes the gap entry and commits atomically with the new doc.
6. If the fill fails, leave the gap open with a note."
)
\```

Fill in `<slug>`, `<predicted-path>`, `<question>`, `<source>`. Everything else is boilerplate. **One tool call, main thread never blocks.**

## Slug derivation

Use the filename stem of the predicted path:
- Predicted path: `projects/query-omega-oql/gotchas/fold-is-global-not-per-row.md`
- Slug: `fold-is-global-not-per-row`

If the slug already exists in gaps.md, append `-2` (or `-3`, etc.).

## When to fire

Gap-filing rules are embedded in `## Gap rules` sections throughout the KB's README tree. The global rules are in the root README. Project-specific rules are in each project's README. Read them during navigator descent — they tell you when to fire for the subtree you're in.

## Batching (edge case)

For tightly-coupled gaps that should land in one commit, use the full ceremony:
1. Derive slugs for each gap
2. Append all entries to `gaps.md`
3. Commit once
4. Dispatch one background agent per gap (max 3 parallel)

Only use this when the gaps are interdependent. For independent gaps, dispatch separate fire-and-forget agents.

## Refuse if

- Predicted path is outside `omega-knowledge-base/` or a project's `/docs/` tree
- Predicted path is absolute or contains `..`
```

- [ ] **Step 2: Commit**

```bash
git -C ~/Development/omega/omega-knowledge-base add skills/omega-docs-mark-gap/SKILL.md
git -C ~/Development/omega/omega-knowledge-base commit -m "feat: rewrite mark-gap skill as fire-and-forget dispatch"
```

---

## Task 5: Update fill-gap skill for slugs

**File:** `~/Development/omega/omega-knowledge-base/skills/omega-docs-fill-gap/SKILL.md`

- [ ] **Step 1: Update the Arguments section**

Change:

```markdown
## Arguments

- **`gap_key`** — the 8-char hash of the gap to fill. If not specified, pick the oldest `open` gap in `gaps.md`.
```

To:

```markdown
## Arguments

- **`gap_slug`** — the human-readable slug of the gap to fill (e.g., `fold-is-global-not-per-row`). Also accepts legacy 8-char hash keys for backward compatibility. If not specified, pick the oldest `open` gap in `gaps.md`.
```

- [ ] **Step 2: Update all references to `gap_key` in the file**

Replace every occurrence of `gap_key` with `gap_slug` throughout the file. Also replace `<gap_key>` with `<gap_slug>` in commit messages and section headers. Use replace_all.

- [ ] **Step 3: Update the gap lookup instruction**

Change step 1 from:

```markdown
Read `~/Development/omega/omega-knowledge-base/gaps.md`. Find the `## gap-<gap_key>` entry.
```

To:

```markdown
Read `~/Development/omega/omega-knowledge-base/gaps.md`. Find the `## gap-<gap_slug>` entry (match on the slug after `gap-`).
```

- [ ] **Step 4: Commit**

```bash
git -C ~/Development/omega/omega-knowledge-base add skills/omega-docs-fill-gap/SKILL.md
git -C ~/Development/omega/omega-knowledge-base commit -m "feat: fill-gap skill accepts slugs instead of hash keys"
```

---

## Task 6: Update navigator skill with position awareness

**File:** `~/Development/omega/omega-knowledge-base/skills/omega-docs-navigator/SKILL.md`

- [ ] **Step 1: Add position awareness rule**

Add a new section after "## Non-negotiables":

```markdown
## KB position awareness

Before starting any task, identify which KB subtree it belongs to. State it explicitly: "This task lives at `projects/<name>/` in the KB."

- **If the task has NO corresponding KB subtree or entry** — this is a BLOCKER. File a gap to create the subtree BEFORE starting the task. The gap-fill agent will create at minimum a pointer file; for substantial work, it will promote to a subtree.
- **When switching tasks or projects mid-session** — re-orient: "Switching to `projects/<other-name>/`." Read that subtree's README for its gap rules.
- **When descending into a bucket** — note the bucket's `## Gap rules` section if it exists. These refine your firing discipline for the work you're about to do.
```

- [ ] **Step 2: Update the companion skills section**

Replace:

```markdown
## Companion skills

- `omega:omega-docs-mark-gap` — record a gap you hit. Batch multiple gaps in one invocation when possible.
- `omega:omega-docs-fill-gap` — fill a gap by reading research sources and writing the missing doc. Runs in background.
```

With:

```markdown
## Companion skills

- `omega:omega-docs-mark-gap` — file a gap as a fire-and-forget background agent dispatch. See the dispatch template injected by the SessionStart hook.
- `omega:omega-docs-fill-gap` — fill a gap by reading research sources and writing the missing doc. Runs in background.

Gap-filing rules are embedded in `## Gap rules` sections in README.md files throughout the tree. Read them as you descend — they tell you when to fire a gap-fill agent for the subtree you're in.
```

- [ ] **Step 3: Commit**

```bash
git -C ~/Development/omega/omega-knowledge-base add skills/omega-docs-navigator/SKILL.md
git -C ~/Development/omega/omega-knowledge-base commit -m "feat: add KB position awareness and gap rules to navigator"
```

---

## Task 7: Update SessionStart hook script

**File:** `~/Development/omega/omega-knowledge-base/scripts/omega-session-start.sh`

- [ ] **Step 1: Rewrite the hook script**

Replace the entire script with:

```bash
#!/usr/bin/env bash
# SessionStart hook for Claude Code. Fires when a session starts in an
# Omega-ecosystem directory. Injects:
#   1. Navigator instruction (invoke omega:omega-docs-navigator)
#   2. KB position awareness rule
#   3. Fire-and-forget gap dispatch template
#   4. Gap-filing rules from the root README
#   5. Unclaimed gap count from gaps.md
#
# Wired up via hooks/hooks.json → ${CLAUDE_PLUGIN_ROOT}/scripts/...

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

# Count unclaimed gaps
unclaimed=0
if [ -f "$kb_root/gaps.md" ]; then
  unclaimed=$(grep -c '^- \*\*status\*\*: open' "$kb_root/gaps.md" 2>/dev/null || echo 0)
fi

gap_notice=""
if [ "$unclaimed" -gt 0 ]; then
  gap_notice="
NOTE: $unclaimed unclaimed gap(s) in gaps.md. Consider dispatching fill-gap agents if relevant to this session's work."
fi

cat <<ENDJSON
{
  "hookSpecificOutput": {
    "hookEventName": "SessionStart",
    "additionalContext": "Before answering any question about Omega, OQL, or any project in the Omega ecosystem, invoke the omega:omega-docs-navigator skill. It will orient you in the knowledge base tree at ~/Development/omega/omega-knowledge-base and establish the descend-from-root navigation rules for this session.\nBefore answering any question about Omega, OQL, or any project in the Omega ecosystem, invoke the omega:omega-docs-navigator skill. It will orient you in the knowledge base tree at ~/Development/omega/omega-knowledge-base and establish the descend-from-root navigation rules for this session.\n\nKB POSITION AWARENESS — hard rule for this session:\nBefore starting any task, identify which KB subtree it belongs to. State it: \"This task lives at projects/<name>/ in the KB.\" If no subtree exists, file a gap to create one BEFORE starting work. When switching projects, re-orient and read the new subtree's README.\n\nFIRE-AND-FORGET GAP DISPATCH — use this template whenever you notice a documentation gap:\nAgent(description=\"Fill gap: <slug>\", run_in_background=true, prompt=\"Record and fill a documentation gap in ~/Development/omega/omega-knowledge-base. Slug: <slug>. Predicted path: <predicted-path>. Question: <question>. Source: <what-you-learned>. Steps: (1) check gaps.md for duplicate slug, (2) append gap entry, (3) commit, (4) invoke omega:omega-docs-fill-gap with gap_slug=<slug>, (5) fill-gap agent writes the doc and deletes the entry atomically.\")\n\nGAP-FILING RULES (from root README — project-specific rules in each project's README):\n- Predicted KB path doesn't resolve → file before falling back to code\n- Debugged >5 min, simple fix → file a gotcha\n- Undocumented behavior noticed mid-task → file without stopping\n- Learned something from code that should be in a doc → file\n- Made a mistake a pre-trained model would get wrong without project docs → file\n- User corrected you on something basic → file the correction\n- When in doubt, file. False positives cost one background agent. False negatives cost repeated mistakes.${gap_notice}"
  }
}
ENDJSON
```

- [ ] **Step 2: Verify the script runs without errors**

```bash
cd ~/Development/omega/omega-knowledge-base
bash scripts/omega-session-start.sh | python3 -m json.tool
```

Expected: valid JSON with the `hookSpecificOutput.additionalContext` field containing the full injected text. No syntax errors.

- [ ] **Step 3: Commit**

```bash
git -C ~/Development/omega/omega-knowledge-base add scripts/omega-session-start.sh
git -C ~/Development/omega/omega-knowledge-base commit -m "feat: SessionStart hook injects gap dispatch template + firing rules + position awareness"
```

---

## Task 8: End-to-end test

**No files changed.** Verify the full flow works.

- [ ] **Step 1: Simulate a session start**

```bash
cd ~/Development/omega/omega-knowledge-base
bash scripts/omega-session-start.sh | python3 -m json.tool
```

Verify the output contains:
1. Navigator invocation instruction
2. KB position awareness rule
3. Fire-and-forget dispatch template
4. Gap-filing rules
5. Unclaimed gap count (if any)

- [ ] **Step 2: Test a fire-and-forget gap dispatch**

From any Omega project directory, dispatch a test gap as a background agent:

```
Agent(
  description="Fill gap: test-gap-plugin-verification",
  run_in_background=true,
  prompt="Record and fill a documentation gap in ~/Development/omega/omega-knowledge-base.

Slug: test-gap-plugin-verification
Predicted path: contributor/how-to/test-gap-plugin-verification.md
Question: Does the fire-and-forget gap dispatch work end-to-end?
Source: This is a test gap to verify the plugin redesign. The doc should say 'This gap was filed as a test of the fire-and-forget dispatch template. If you see this file, the plugin works. Delete this file and its gap entry.'

Steps:
1. Read gaps.md. If a gap with this slug already exists, stop (duplicate).
2. Append a new entry: ## gap-test-gap-plugin-verification with question, predicted-path, fallback-source, recorded-at, status: open.
3. git -C ~/Development/omega/omega-knowledge-base add gaps.md && git commit -m 'gap: test-gap-plugin-verification'
4. Invoke the omega:omega-docs-fill-gap skill with gap_slug=test-gap-plugin-verification to research and write the doc.
5. If the fill succeeds, the fill-gap agent deletes the gap entry and commits atomically with the new doc.
6. If the fill fails, leave the gap open with a note."
)
```

Wait for the background agent to complete. Expected:
- `gaps.md` temporarily has a `## gap-test-gap-plugin-verification` entry (then deleted)
- `contributor/how-to/test-gap-plugin-verification.md` is created with test content
- One atomic commit lands with the doc + gap deletion

- [ ] **Step 3: Clean up the test**

```bash
git -C ~/Development/omega/omega-knowledge-base rm contributor/how-to/test-gap-plugin-verification.md
git -C ~/Development/omega/omega-knowledge-base commit -m "chore: remove test gap verification file"
```

---

## After this plan

1. **Start a fresh session** to verify the hook fires correctly and the injected context includes all the new rules.
2. **Work on any Omega task** and verify that the assistant naturally follows the gap-filing rules without being prompted.
3. **Monitor gap-filing frequency** — the goal is that gaps are filed proactively (3-5 per session as a baseline) rather than reactively (only when the user points them out).
