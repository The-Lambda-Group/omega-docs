[< Home](../README.md) | [Reference](../README.md#reference)

# Commands

## Library Management

These commands operate on the component library (`QO_LIBRARY_PAGE_ID`).

### `qo pull`

Sync all implementations from the app to disk. Creates the `impl/` directory tree mirroring the page structure.

```bash
qo pull
```

### `qo push <files...>`

Push implementation file(s) to OmegaDB. OQL code is evaluated on write via `Qo.Impl/write-data`.

```bash
qo push impl/my-library/my-component/implementation.oql
```

## Workspace Operations

These commands operate on the workspace (`QO_WORKSPACE_ID`). The workspace root is an app **folder** (not a page) that contains both libraries and component installs as top-level siblings.

### `qo ls [page-path]`

List child pages under a path.

```bash
qo ls                              # list workspace root
qo ls "Component Installs"         # list installed components
qo ls "Component Installs/My App"  # list sub-pages of an install
qo ls "Component Installs/My App" -v  # include page IDs in output
```

Options:
- `-v, --verbose` - Show page IDs alongside names

### `qo run <page-path>`

Run a page's implementation by navigating the workspace page tree.

```bash
# Install a component
qo run "Component Installs/My App" -p "Installable" -m "install"

# Run a query
qo run "Component Installs/My App/Queries/list-data" -p "Query" -m "run" -a '[{}]'

# Trigger an event handler
qo run "Component Installs/My App/Events/Coordinator" -p "Event Handler" -m "on-event" -a '[{}]'
```

Options:
- `-p, --protocol <name>` - Protocol name (required)
- `-m, --method <method>` - Method to invoke (required)
- `-a, --args <json>` - JSON args array

### `qo logs [page-path]`

Read log stream entries for a page.

```bash
qo logs "Component Installs/My App/Event Listener"
```

### `qo query <page-path>`

Query a database page's rows. Supports sorting by secondary index.

```bash
qo query "Component Installs/My App/Data/Agents" -c "score" -s "desc" -l 10
```

Options:
- `-c, --column <name>` - Column to sort by (uses sec-index)
- `-s, --sort <direction>` - Sort direction (asc/desc)
- `-l, --limit <n>` - Max rows (default: 10)

### `qo write <page-path>`

Write rows to a database page via stdin. Input is JSON: `{"header": [...], "rows": [[...], ...]}`.

```bash
echo '{"header": ["name", "email"], "rows": [["Alice", "a@b.com"]]}' | qo write "Data/My Table"
```

### `qo describe <page-path>`

Show the schema for a database page: properties, primary key, and secondary indexes.

```bash
qo describe "Component Installs/My App/Data/Agents"
```

### `qo delete-page <page-path>`

Delete a page by removing the block on its parent page, then deleting the page itself. Falls back to direct delete for orphaned pages that have no block.

```bash
qo delete-page "Component Installs/My App/Data/Old Table"
```

### `qo drop <page-path>`

Drop row data from a database page. Schema (properties, primary key, secondary indexes) is preserved.

```bash
qo drop "Component Installs/My App/Data/Agents"
```

## Page & Component Management

### `qo add-page <parent-page.json> <name>`

Create or get a sub-page by name under a parent.

```bash
qo add-page impl/my-library/page.json "My Component"
```

### `qo add-protocol <page.json> <name> <spec.json>`

Create or get a protocol on a page.

```bash
qo add-protocol impl/protocols/page.json "Query" query-spec.json
```

### `qo add-component <page.json> <name>`

Create or get a component definition on a library page.

```bash
qo add-component impl/my-component/page.json "My Component"
```

### `qo add-implementation <page.json> <protocol-id> <component-id>`

Create or get an implementation linking a component to a protocol.

```bash
qo add-implementation impl/my-component/page.json "<protocol-id>" "<component-id>"
```

## Configuration

### `qo set-component <page-id> <app-id> <component-id>`

Attach an existing component to a workspace page. Used for top-level component installs before running install.

```bash
qo set-component "<page-id>" "<app-id>" "<component-id>"
```

### `qo set-push-connector <page-path> <name>`

Add or update a push connector (key-value config) on a page. Used for storing API credentials and cross-component references.

```bash
qo set-push-connector "Component Installs/My App" "MyService" \
  -d '{"name": "MyService", "api-key": "secret", "endpoint": "https://api.example.com"}'
```
