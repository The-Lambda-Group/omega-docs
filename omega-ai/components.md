[< Home](../README.md) | [OmegaAI](../README.md#omegaai)

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

### Workspace Structure

```
Workspace Root (folder)/
  Component Libraries/              <- source code (qo pull / qo push)
    My Library/
      My Component/
        {id}-implementation.oql     <- OQL source
        {id}-my-component.component.json
        page.json
  Component Installs/               <- running instances (qo run / qo ls / qo logs)
    My Plugin/
      Event Listener/               <- event handler + log stream
      Events/
        Coordinator/
        Score Agents/
      Data/
        My Table/                   <- database page
      Queries/
        emit-event/
        print-stream/
```

## The App Is the Source of Truth

OQL code and data live in the database, not on disk. Files on disk (`impl/`) are a local mirror — a convenience for editing. Git history is a recovery safety net, not a deployment mechanism.

```
Disk (impl/)  ──push──▶  OmegaDB  ◀──run──  qo run / UI
                  ◀──pull──
```

- **Pull** syncs from the app to disk
- **Push** syncs from disk to the app — code is evaluated on write
- **Run** invokes a page's implementation through its protocol

Always `qo pull` before editing to get the latest. If the app and disk diverge, the app wins.

## The Exec Object

When a clause is invoked via `run-page` at runtime, it receives an `Exec` object containing the method metadata and caller's arguments:

```oql
(:- (OnEvent Exec Result)
    ;; The page this implementation is running on
    (get-in Exec ["method" "page-id"] PageId)

    ;; The implementation being executed
    (get-in Exec ["method" "implementation-id"] ImplId)

    ;; The method name and arity
    (get-in Exec ["method" "method-name"] MethodName)
    (get-in Exec ["method" "method-arity"] Arity)

    ;; The arguments passed by the caller
    (get Exec "args" Args)
    (get Args 0 Data))
```

The `page-id` in `Exec` is how implementations discover their context — they walk the page tree from their own page to find sibling pages, parent config, and data tables.

## $$ARG$$

Available at **push time** only (when `qo push` evaluates the file). Contains the implementation metadata:

```oql
(get $$ARG$$ "implementation-id" ImplId)  ;; the implementation being pushed
(get $$ARG$$ "app-id" AppId)              ;; the app it belongs to
(get $$ARG$$ "component-id" CompId)       ;; the component it implements
(get $$ARG$$ "protocol-id" ProtoId)       ;; the protocol it fulfills
```

Used at the bottom of every implementation file to register clauses:

```oql
(= Clauses {"on-event" {"2" OnEvent}})
(get $$ARG$$ "implementation-id" ImplId)
(Qo.Public.OqlApi.Impl/set-implementation-clauses ImplId Clauses Result)
```

The map key is the method name, the nested key is the arity (as a string), and the value is the in-memory clause symbol.
