[< Home](../README.md) | [Explanation](README.md)

# What is Omega?

Omega is a notebook-based application platform backed by a logic database.

In Omega, everything is a **page**. A page can hold code, data, configuration, and logs — just like a cell in a notebook can hold different kinds of content. Pages are organized into trees, and the whole system is queryable and programmable through OQL (Omega Query Language), a datalog-style logic language.

## The Notebook

The core of Omega is a notebook. You create pages, add blocks to them, and build applications by composing pages together. The notebook runs in a browser, but you can also interact with it entirely from the command line using the `qo` CLI.

A page can have several kinds of blocks:

- **Code blocks** — OQL implementations that define behavior
- **Data blocks** — database tables with rows, properties, indexes
- **Config blocks** — push connectors that store key-value configuration (like environment variables)
- **Log blocks** — log streams for observability

When you write code in a page's code block and save it, the code is evaluated immediately — clause definitions are stored in the database, ready to be invoked later. This is how Omega works: code and data live together in the same system.

## The Database

Underneath the notebook is OmegaDB, a logic database. OmegaDB stores rules, facts, and JSON documents. When you query it, you're not running SQL — you're asking the database to find all possible values that satisfy a set of logical constraints.

```oql
;; "Find all pages where the name is 'My Component'"
(Qo.Page/page-object _ _ PageId Page)
(get-in Page ["page-data" "name"] "My Component")
```

This is fundamentally different from a traditional database. Instead of telling the system *how* to find data (joins, filters, aggregations), you describe *what* you're looking for and the engine figures out how to get there.

## The Platform

On top of the notebook and database, Omega is a platform for building applications. Applications are built as **plugins** — installable components that set up their own page trees, databases, event handlers, and queries.

A plugin is like a Unix package. You install it, it creates its own structure, and you operate it with standard tools:

```bash
# Install a plugin
qo run "Component Installs/My Plugin" -p "Installable" -m "install"

# Start its event processing
qo run "Component Installs/My Plugin/Event Listener" -p "Service" -m "start"

# Query its data
qo query "Component Installs/My Plugin/Data/My Table" -l 10

# Check its logs
qo logs "Component Installs/My Plugin/Event Listener"
```

Every plugin follows the same patterns, uses the same CLI, and stores its data in the same inspectable way. You don't need to read a plugin's source code to understand its state — you can `qo query` its tables, `qo logs` its event listener, and `qo describe` its schema.

## How It Fits Together

```
                    ┌─────────────────────────┐
                    │     Notebook UI          │  React frontend
                    │  (query-omega)           │  
                    └────────────┬────────────┘
                                 │
┌────────────────┐  ┌────────────┴────────────┐
│   MCP Server   │  │      qo CLI             │  Command-line interface
│ (query-omega-  │──│  (query-omega-cli)       │  
│  mcp)          │  └────────────┬────────────┘
└────────────────┘               │
                    ┌────────────┴────────────┐
                    │    Public OQL API        │  The interface components use
                    │  (query-omega-api)       │  
                    └────────────┬────────────┘
                                 │
                    ┌────────────┴────────────┐
                    │    Core OQL Library      │  Page, property, implementation systems
                    │  (query-omega-oql)       │  
                    └────────────┬────────────┘
                                 │
                    ┌────────────┴────────────┐
                    │      OmegaDB            │  Logic database engine
                    │  (omega-db)              │  
                    └─────────────────────────┘
```

The **UI**, **CLI**, and **MCP Server** are three ways to interact with the same system. The MCP Server wraps the CLI for AI agent use. All call the **Public OQL API**, which is the only interface components should use. The API wraps the **Core Library**, which talks directly to the **database engine**.

**Plugins** (like the Homes.com Connector) are built on top of this stack. They use the **Shared Component Library** (omega-ai-components) for common functionality like event handling, logging, and queries.
