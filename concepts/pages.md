# Pages

Everything in Omega lives on a page. A page is a container that can hold code, data, configuration, and logs. Pages are organized into a tree — every page has a parent, and can have children.

## Page Tree

The page tree is how Omega organizes an application:

```
My App/
  Event Listener/          <- has a log stream, receives events
  Events/
    Coordinator/           <- code block: routes events to steps
    Score Agents/          <- code block: scoring logic
    Filter Agents/         <- code block: filtering logic
  Data/
    Scored Agents/         <- data block: database table
    Filtered Agents/       <- data block: database table
  Queries/
    emit-event/            <- code block: utility query
    print-stream/          <- code block: logging query
```

Each page in this tree has a unique **page ID** and belongs to a **folder** (its parent). You navigate the tree by path, just like a filesystem:

```bash
qo ls "Events"                    # list children of Events
qo ls "Events/Coordinator"        # list children of Coordinator
qo ls -v "Data"                   # list Data children with page IDs
```

## Blocks

A page can have multiple blocks. Each block adds a capability to the page:

### Code Blocks (Implementations)

A code block contains OQL code that implements a protocol method. When the code is saved (pushed), it's evaluated — clause definitions are stored in the database. When someone runs the page, the stored clauses execute.

```bash
# Push code to a page
qo push impl/my-component/implementation.oql

# Run a page's code
qo run "Events/Coordinator" -p "Event Handler" -m "on-event" -a '[{}]'
```

### Data Blocks (Databases)

A data block turns a page into a database table. The page gets properties (columns), a primary key, optional secondary indexes, and rows.

```bash
# Query a data page
qo query "Data/Scored Agents" -c score -s desc -l 10

# Describe its schema
qo describe "Data/Scored Agents"

# Write rows to it
echo '{"header": ["name", "score"], "rows": [["Alice", "95"]]}' | qo write "Data/Scored Agents"
```

### Config Blocks (Push Connectors)

A push connector is a key-value config block — like an `.env` file for a page. Plugins use push connectors to store API keys, reference sibling page IDs, and hold runtime configuration.

```bash
# Set a push connector
qo set-push-connector "My App" "MyService" \
  -d '{"api-key": "secret", "endpoint": "https://api.example.com"}'
```

### Log Blocks (Log Streams)

A log stream captures timestamped messages from a page. Plugins log through the shared `print-stream` query, and you read logs with the CLI.

```bash
qo logs "Event Listener" -l 20
```

## Pages Are the Interface

Everything in Omega is a page operation. You don't interact with "tables" or "functions" or "services" directly — you interact with pages that *have* those capabilities. A page with a data block is a table. A page with a code block is a function. A page with a log block is observable.

This is why the CLI is so uniform: `qo run`, `qo query`, `qo logs`, `qo describe` all take a page path. The page is the unit of everything.
