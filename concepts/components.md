# Components, Protocols, and Implementations

Omega's code model has three parts: **components**, **protocols**, and **implementations**. Together they define what a page can do and how it does it.

## Protocols

A protocol is an interface definition. It declares a set of methods with names and arities (argument counts). Protocols don't contain any code — they just describe what methods exist.

```json
{
  "functions": [
    { "name": "install", "args": ["self"] },
    { "name": "start",   "args": ["self"] },
    { "name": "stop",    "args": ["self"] },
    { "name": "status",  "args": ["self"] }
  ]
}
```

The standard protocols are:

| Protocol | Methods | Purpose |
|----------|---------|---------|
| Installable | `install(self)` | Set up a plugin's page tree and config |
| Service | `start(self)`, `stop(self)`, `status(self)` | Manage a plugin's runtime lifecycle |
| Event Handler | `on-event(self, event)` | Handle dispatched events |
| Query | `run(self, input)` | Execute a query with input/output |

Protocols are defined once in the shared component library and reused across all plugins.

## Components

A component is a named unit of code that lives on a library page. It's the thing you build — the "package" that can be installed elsewhere. A component implements one or more protocols by providing implementations for their methods.

For example, the "Omega Event Handler" component implements three protocols:
- **Installable** — creates log stream, registers subscription
- **Service** — start/stop/status for the event subscription
- **Event Handler** — dispatches events to target pages

## Implementations

An implementation is the actual OQL code that fulfills a protocol method for a component. It's a code block on a page — an `.oql` file that you push to the database.

When you push an implementation:
1. The OQL file is sent to the database
2. The code is **evaluated** — top-level statements run immediately
3. Clause definitions (`(:- ...)`) are stored in the datastore
4. The stored clauses are available to be called later via `qo run`

When you run an implementation:
1. `qo run` finds the page, looks up its component and protocol
2. Finds the matching implementation (component + protocol)
3. Loads the stored clauses
4. Calls the clause matching the method name and arity
5. The clause receives an `Exec` object with the method metadata and args

### Push Time vs Runtime

This distinction is important:

**Push time** — when you `qo push` an implementation file. Top-level statements execute. `$$ARG$$` is available and contains the implementation metadata (app-id, component-id, protocol-id, implementation-id). Use this for one-time setup like defining shared handler clauses.

**Runtime** — when someone calls `qo run` on a page. Stored clauses execute. `Exec` is available and contains the method metadata, page-id, and args. Use this for the actual behavior.

```oql
;; This runs at PUSH TIME — defines a clause and stores it
(:- (Start Exec Result)
    (get-in Exec ["method" "page-id"] PageId)
    ;; ... clause body ...
    (= Result {"status" "started"}))

;; This runs at PUSH TIME — registers the clause with the system
(= Clauses {"start" {"1" Start}})
(get $$ARG$$ "implementation-id" ImplId)
(Qo.Public.OqlApi.Impl/set-implementation-clauses ImplId Clauses Result)
```

The `(:- ...)` form doesn't execute the clause body — it defines it. The last three lines run at push time to register the clause so it can be called later.

## How They Connect

```
Protocol (interface)          Component (package)          Implementation (code)
┌──────────────────┐         ┌───────────────────┐        ┌──────────────────────┐
│ Service          │         │ Event Handler     │        │ start: Start clause  │
│  - start(self)   │────────▶│  Component        │───────▶│ stop: Stop clause    │
│  - stop(self)    │         │                   │        │ status: Status clause│
│  - status(self)  │         │                   │        └──────────────────────┘
└──────────────────┘         │                   │        ┌──────────────────────┐
┌──────────────────┐         │                   │        │ on-event: OnEvent    │
│ Event Handler    │────────▶│                   │───────▶│   clause             │
│  - on-event(s,e) │         │                   │        └──────────────────────┘
└──────────────────┘         │                   │        ┌──────────────────────┐
┌──────────────────┐         │                   │        │ install: Install     │
│ Installable      │────────▶│                   │───────▶│   clause             │
│  - install(self) │         └───────────────────┘        └──────────────────────┘
└──────────────────┘
```

A page in the workspace has a **component attached**. When you run a method on that page, the system looks up which implementation handles that protocol for that component, loads the clause, and executes it.

## Libraries vs Installs

Components live in two places:

**Component Libraries** — the source code. This is where you write implementations, define components, and push code. Managed with `qo pull` and `qo push`.

**Component Installs** — running instances. When you run a component's `install` method, it creates a page tree in your workspace with sub-pages, databases, event handlers, and config. Managed with `qo run`, `qo ls`, `qo logs`, `qo query`.

The library is like the source code on GitHub. The install is like the running service on your server.
