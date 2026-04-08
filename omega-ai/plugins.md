[< Home](../README.md) | [OmegaAI](../README.md#omegaai)

# Plugins

A plugin is an installable component that creates its own page tree in your workspace. Think of it like `apt-get install` — you install a plugin, it sets up everything it needs, and you operate it with standard tools.

## What a Plugin Looks Like

When you install a plugin, it creates a page tree under your workspace:

```
Component Installs/
  Homes.com Connector/              <- the plugin's root page
    Homes.com Event Listener/       <- event handler + log stream
    Homes.com Data/                 <- source data table
    Events/
      Coordinator/                  <- routes events between steps
      Score Agents/                 <- scoring logic
      Filter Agents/                <- filtering logic
      GHL Sync/                     <- CRM sync
    Data/
      Scored Agents/                <- output table
      Filtered Agents/              <- output table
      GHL Contacts/                 <- sync ledger
```

Every page in this tree is inspectable:

```bash
qo ls "Component Installs/Homes.com Connector" -v   # see the tree
qo query "Component Installs/.../Data/Scored Agents"  # query a table
qo logs "Component Installs/.../Homes.com Event Listener"  # read logs
qo describe "Component Installs/.../Data/Scored Agents"    # see schema
```

## Unix-Style Philosophy

Plugins should feel like Unix services:

- **Do one thing well.** A plugin has a clear purpose. The Homes.com Connector scrapes agents, scores them, and syncs to a CRM. That's it.
- **Inspectable state.** All data lives in database tables you can query. No hidden internal state.
- **Standard lifecycle.** Every plugin supports install, start, stop, status through standard protocols.
- **Composable.** Plugins communicate through events, not direct coupling. You can swap out a scoring plugin without touching the sync plugin.
- **Configuration as data.** Push connectors are the plugin's `.env` — API keys, endpoint URLs, and cross-component references.

## Plugin Lifecycle

A plugin is operated entirely through protocols via `qo run`:

```bash
# Install — creates page tree, databases, config blocks
qo run "Component Installs/My Plugin" -p "Installable" -m "install"

# Start — registers event subscriptions, begins processing
qo run "Component Installs/My Plugin/Event Listener" -p "Service" -m "start"

# Stop — deletes event subscriptions, halts processing
qo run "Component Installs/My Plugin/Event Listener" -p "Service" -m "stop"

# Status — check if subscriptions are active
qo run "Component Installs/My Plugin/Event Listener" -p "Service" -m "status"

# Trigger work
qo run "Component Installs/My Plugin/Events/Coordinator" \
  -p "Event Handler" -m "on-event" -a '[{"step": "score"}]'
```

No special CLI commands are needed. The `qo run` command invokes any protocol method on any page. The plugin defines its behavior through protocol implementations.

## Actor Model

Plugins communicate through events, following the actor model:

- **Events are messages.** A component emits an event and moves on. It doesn't wait for the handler to finish.
- **Subscriptions are mailboxes.** Each event listener page has a subscription that receives events and dispatches them to handlers.
- **Handlers are isolated.** An `on-event` implementation processes one message at a time — reads tables, writes tables, optionally signals continuation.
- **No shared mutable state.** Components share data through database tables, not in-memory state.

This means a plugin can process a batch of 300,000 records by emitting events to itself — each batch processes, writes results to a table, and emits a continuation event for the next batch. If something goes wrong, you can `stop` the service to kill the event loop.

## Building a Plugin

A plugin is built as a component library:

1. **Define your component** — create a library page, add a component
2. **Implement Installable** — write an install method that creates your page tree
3. **Implement your logic** — event handlers, queries, data processing
4. **Push your code** — `qo push` deploys implementations to the database
5. **Install and run** — `qo run` installs your plugin in a workspace

Plugins use the shared component library (omega-ai-components) for common functionality:
- **Event handling** — subscriptions, dispatch, continuation
- **Logging** — `print-stream` query for structured logging
- **Event emission** — `emit-event` query for fire-and-forget messaging
- **Lifecycle management** — Service protocol for start/stop/status

Components should never use raw OQL built-in terms for system operations. Always use the shared library or public API:

| Instead of | Use |
|-----------|-----|
| `emit-event` / `new-subscription-event` | `emit-event` query |
| `write-event-subscription` | `Qo.Public.OqlApi.PageSub/write-page-subscription` |
| `println-stream` | `print-stream` query |
