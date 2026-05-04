[< Reference](README.md)

# Docs Folder Convention

Naming and placement rules for synced project doc trees in an OmegaAI workspace.

## Workspace layout

All synced doc trees live under a single `Docs/` folder at the workspace root:

```
<workspace root>/
  Docs/
    MAB/
    Redefine Reach/
    <Other Project>/
```

`Docs/` is created once, idempotently:

```
qo add-page "." "Docs"
```

Re-running this command is safe — it is a no-op if the page already exists.

## Subfolder naming

Each project gets its own subfolder named with the project's **display name** — human-readable, not a slug.

| Project | Subfolder |
|---------|-----------|
| MAB | `Docs/MAB` |
| Redefine Reach | `Docs/Redefine Reach` |

Use spaces and capital letters as they appear in the project's display name. Do not use slugs (`docs/mab`, `docs/redefine-reach`) or snake_case.

## Local source layout

The local source directory mirrors the workspace layout. By convention, synced pages live in a `pages/` directory inside the project's git repo:

```
<repo>/
  pages/
    <project-slug>/   ← local source for Docs/<Project Display Name>
      README.html
      ...
```

Examples:

| Local source path | Syncs to |
|-------------------|----------|
| `mab/pages/mab/` | `Docs/MAB` |
| `mab/pages/redefine-reach/` | `Docs/Redefine Reach` |

The `pages/` directory is the local source of truth. HTML files inside it are tracked in git alongside the markdown source they were generated from.

## Which repo owns which pages

The `pages/` directory lives in whichever project repo contains the relevant markdown source. This is a pragmatic choice, not a strict rule.

In the WTTC workspace, both `Docs/MAB` and `Docs/Redefine Reach` pages live in the `mab` repo (`~/Development/wttc/mab/pages/`) because `mab` is the active WTTC project repo. Pages for a different project could live in any repo that has the relevant docs.

## Sync command

Push a local source directory to the corresponding workspace page:

```
qo doc push pages/<project-slug> "Docs/<Project Display Name>"
```

This command is source-authoritative and idempotent — re-running it always reflects the latest state of the local `pages/<project-slug>/` directory.

## Related

- For step-by-step instructions on converting markdown docs and pushing them to OmegaAI, see `query-omega-cli/docs/how-to/publish-docs-for-agent-access.md`.
