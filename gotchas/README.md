[< Home](../README.md)

# Gotchas

Failure modes, footguns, and debugging patterns. Read these when something surprising happened or when you're trying to diagnose a bug. Design-level rationale lives in `explanation/`; this bucket is for "things that bit us."

## Language-level

- [Lexical scope](lexical-scope.md) — how scoping works in clause bodies; subtle stale-value bugs
- [Conventions](conventions.md) — boolean stringification, return-binding rules, common footguns

Deeper project-specific OQL gotchas live in `omega-knowledge-base/projects/query-omega-oql/gotchas/`. This bucket is for gotchas that are part of the public language surface.
