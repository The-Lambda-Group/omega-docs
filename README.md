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
