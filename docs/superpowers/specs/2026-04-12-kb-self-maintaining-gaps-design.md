# Self-Maintaining Knowledge Base — Design Spec

**Date:** 2026-04-12 (UTC)
**Status:** Design, pre-implementation
**Scope:** Redesign the omega-knowledge-base plugin's gap-filing workflow from a heavyweight 5-step ceremony to a fire-and-forget single-agent dispatch, with firing rules integrated into the KB's self-similar README structure.

## Problem

The current mark-gap skill requires the assistant to orchestrate 5+ tool calls in the main thread: compute a sha1 key, read gaps.md for duplicates, edit gaps.md with a full entry, commit, dispatch a fill-gap agent. This takes 30-60 seconds and breaks the assistant's flow. The result is all-or-nothing: either the assistant stops working to file a gap (expensive), or it doesn't file at all (documentation debt accumulates). In a typical session, the assistant notices 5-10 potential gaps but only files 2-3 because the overhead is too high.

Additionally, gap keys are opaque sha1 hashes (`gap-7184bd0f`) that are meaningless when scanning gaps.md. And there are no behavioral rules telling the assistant WHEN to file gaps — it's left to ad-hoc judgment, which means gaps get filed reactively (after the user points them out) rather than proactively (when the assistant notices something undocumented).

## Solution

### 1. Fire-and-forget gap dispatch

Mark-gap becomes a single background agent dispatch. The assistant's cost: one `Agent(run_in_background=true)` call with a templated prompt. The background agent handles the full ceremony internally.

**The dispatch template** (injected by the SessionStart hook):

```
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
2. Append a new entry using the slug as the key (format: ## gap-<slug>).
3. Commit: git -C ~/Development/omega/omega-knowledge-base add gaps.md && git commit -m 'gap: <slug>'
4. Invoke the omega:omega-docs-fill-gap skill to research and write the doc at the predicted path.
5. If the fill succeeds, delete the gap entry from gaps.md and commit atomically with the new doc.
6. If the fill fails (can't find the answer), leave the gap open with a note explaining why."
)
```

The assistant fills in `<slug>`, `<predicted-path>`, `<question>`, and `<source>`. Everything else is boilerplate. One tool call, main thread never blocks.

### 2. Human-readable slugs

Gap keys are derived from the predicted path's filename stem:

- Predicted path: `projects/query-omega-oql/gotchas/fold-is-global-not-per-row.md`
- Slug: `fold-is-global-not-per-row`
- Gap entry header: `## gap-fold-is-global-not-per-row`

Dedup is on the slug. If two gaps predict different paths but the same filename stem, the second one appends a disambiguator (e.g., `fold-is-global-not-per-row-2`). In practice this is rare because predicted paths are specific.

The `gaps.md` entry format changes from:

```markdown
## gap-7184bd0f

- **question**: ...
- **predicted-path**: ...
```

To:

```markdown
## gap-fold-is-global-not-per-row

- **question**: ...
- **predicted-path**: ...
```

Everything else stays the same. The fill-gap skill accepts a slug instead of a hash key.

### 3. Firing rules in the self-similar README structure

Every README.md in the knowledge base can carry a `## Gap rules` section with firing triggers specific to that subtree. The assistant reads READMEs as it descends (per the navigator discipline), so gap-filing rules are discovered naturally alongside the content they govern.

**Root README (`README.md`)** — global rules:

```markdown
## Gap rules

File a documentation gap (fire-and-forget, do not stop your current work) when:

- A predicted KB path doesn't resolve — file before falling back to code
- You debug something for >5 minutes that had a simple fix — file a gotcha for the missing doc
- You notice undocumented behavior while working — file without stopping
- You learn something by reading code that should have been in a doc — file
- You make a mistake that a pre-trained model would get wrong without project-specific docs — file
- You get corrected by the user on something basic — file the correction as a gotcha
- A subagent asks a question that the KB should have answered — file

When in doubt, file. The cost of a false positive (a gap that turns out to be unnecessary) is one background agent that bails. The cost of a false negative (a gap that doesn't get filed) is repeating the same mistake in a future session.
```

**Project README (e.g., `projects/query-omega-oql/README.md`)** — project-specific rules:

```markdown
## Gap rules

In addition to global rules:
- If an OQL primitive behaves differently than Prolog/Datalog semantics predict, file under gotchas/
- If a `with-table-if` pattern causes silent row drops or schema collapse, file under gotchas/
- If you discover a term that requires specific argument binding order (non-relational), file under gotchas/
- If you use fold where append was needed (or vice versa), file under gotchas/
```

**Bucket README (e.g., `projects/query-omega-oql/gotchas/README.md`)** — bucket-specific rules:

```markdown
## Gap rules

File a gotcha here when:
- Debugging revealed a footgun that cost >5 minutes
- An OQL pattern worked at limit=1 but broke at limit=100 (batch-dependent bug)
- A silent failure (no error, just wrong results) was caused by a language semantic
```

The self-similar structure means every new subtree that gets promoted during rebalancing inherits the gap-filing pattern. The system's documentation discipline scales with its content.

### 4. SessionStart hook changes

The existing SessionStart hook script is modified to:

1. **Inject the dispatch template** — the exact `Agent(...)` call the assistant copy-pastes when filing a gap. This is static text, always injected.

2. **Read and inject the root README's gap rules** — so the assistant has the global firing rules in context from the start, without needing to invoke the navigator first.

3. **Report unclaimed gap count** — if gaps.md has any entries with `status: open`, mention the count so the assistant can dispatch fill agents if appropriate. Example: "3 unclaimed gaps in gaps.md — consider dispatching fill agents."

The hook does NOT read project-specific or bucket-specific gap rules at session start. Those are discovered during normal navigator descent when the assistant enters a project. This keeps the hook output small (~200 words) and project-agnostic.

### 5. Skill changes

**`omega-docs-mark-gap` SKILL.md** — simplified from a 5-step orchestration procedure to:

1. A reference for the dispatch template (same as what the hook injects, for cases where the hook didn't fire or the assistant needs a refresher)
2. The slug derivation rule (filename stem of predicted path)
3. The gaps.md entry format
4. A note: "Prefer the fire-and-forget dispatch. Only use the full mark-gap ceremony when you need to batch multiple tightly-coupled gaps in one commit."

The skill is still invokable for edge cases (batching, complex gaps that need manual curation) but the primary path is the hook-injected template.

**`omega-docs-fill-gap` SKILL.md** — one change: accept a slug instead of (or in addition to) a hash key. The `gap_key` parameter becomes `gap_slug`. Everything else stays the same — the fill-gap agent still reads the gap entry from gaps.md, researches, writes the doc, updates the parent README, deletes the gap entry, and commits atomically.

**`omega-docs-navigator` SKILL.md** — add a line to the "Companion skills" section:

> Gap-filing rules are embedded in README.md files throughout the tree (look for `## Gap rules` sections). Follow them as you descend — they tell you when to fire a gap-fill agent for the subtree you're in.

### 6. What this does NOT include

- **No cron/loop for periodic maintenance.** The assistant IS the maintenance process. SessionStart cleanup + in-session discipline is sufficient.
- **No automated debug-pattern detection.** The firing rules are behavioral — the assistant follows them based on judgment. If we need more automation later, it can be added as a hook that fires after tool errors, but that's a separate project.
- **No changes to the fill-gap agent internals.** It already works well as a single background agent. The only change is accepting slugs.
- **No migration of existing hash-based gaps.** Any existing `gap-<hash>` entries in gaps.md are handled by the fill-gap agent as before. New gaps use slugs. The two formats coexist; eventually all hash entries get filled and deleted, leaving only slugs.

## Success criteria

1. The assistant can file a gap in ONE tool call (background agent dispatch) without stopping its current work.
2. Gap entries in gaps.md are scannable by a human (slug-based keys).
3. The assistant proactively files gaps based on firing rules discovered during normal KB navigation.
4. Project-specific firing rules can be added by editing a README — no plugin changes needed.
5. The SessionStart hook injects the dispatch template so the assistant is always primed to file.
