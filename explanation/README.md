[< Home](../README.md)

# Explanation

> **Confidentiality: PUBLIC.** All content in this directory is public platform documentation.

The "why" of Omega -- design rationale, architecture, mechanisms. Read these when you want to understand how a piece of the system works, not how to use it. These are "how does X work" / "why is ..." docs -- if you need "how do I ..." task recipes, see [how-to/](../how-to/README.md); if you need exact signatures, see [reference/](../reference/README.md).

## Information ownership

This bucket owns public platform architecture and design rationale. Internal engine implementation details (CouchDB internals, index structures, replication mechanics) are documented in `omega-knowledge-base`. Project-specific design decisions live in their respective project repos.

## Navigation map

### What is Omega

- [What is Omega?](what-is-omega.md) -- The elevator pitch. Covers Omega as a notebook-based application platform backed by a logic database, the four block types (code, data, config, log), how OmegaDB differs from traditional databases (you describe what you want, not how to get it), the plugin model (installable components with standard lifecycle), and the full system architecture diagram showing how the UI, CLI, MCP Server, Public OQL API, Core Library, and OmegaDB stack together. Start here if you are new to Omega. Keywords: Omega, notebook, logic database, OmegaDB, page, block, plugin, architecture, stack diagram, qo CLI, MCP.

- [Philosophy](philosophy.md) -- Deep design rationale and intellectual lineage. Covers the ontological claim (all computation is relational), the convergence thesis (software is trending toward declarative), the complexity diagnosis (Out of the Tar Pit -- essential vs accidental complexity), the engine thesis (optimization belongs in the engine, not the code), the architecture principle (Unix philosophy in logic), the consistency thesis (eventual consistency as the correct model, CouchDB's multi-master replication), the business strategy (OmegaAI as the product shell around OmegaDB, AI as the unlock, the ERP marketplace play), and the intellectual lineage (Datalog, miniKanren, Prolog, SQL, Clojure, Erlang, CouchDB, Out of the Tar Pit, expression problem). Keywords: philosophy, relational, declarative, Tar Pit, accidental complexity, convergence, ERP, CouchDB, eventual consistency, actor model, logic programming.

### OQL

- [OQL Language](oql-language.md) -- Core language concepts for OQL. Covers terms (the basic unit -- statements that are true or false), symbols and unification (how the engine finds consistent bindings, how unification applies to every term not just `=`), maps and access patterns (`get`/`get-in`/`set` with unification semantics, using `get` as both a reader and a constraint), and links to sub-topic reference pages for datastores, clauses, control flow, built-ins, reducers, and conventions. Keywords: OQL, term, symbol, unification, binding, get, get-in, set, map, constraint, logic programming, Datalog.

- [Execution Model](oql-execution-model.md) -- The most important conceptual doc for writing correct OQL. Covers solution sets (the fundamental table-of-bindings data structure), why terms are not expressions (terms transform solution sets, they cannot be nested inside literals), terms as joins (each term is like a CTE in SQL), clauses as solution boundaries (how in-memory `(:- ...)` clauses keep solutions small, and why `def-query` / `with-query` are deprecated), clause types (stored, in-memory), control flow reference (`when`/`when-not`/`with-table-if`/`and`/`or`/`fold`/`with-count`/`with-order-by`), index matching (queries must match indexes exactly or trigger full table scans), execution semantics (no transactions, fresh context on handler invocation, `println-stream` corrupts solution context), and performance rules of thumb. Keywords: solution set, execution model, join, CTE, expression, term, clause boundary, in-memory clause, def-query deprecated, with-query deprecated, index matching, full table scan, no transactions, println-stream, performance.

### OmegaAI

- [Pages](pages.md) -- The page model in depth. Covers the page tree structure (hierarchical organization with unique page IDs and folder parents), all four block types with CLI examples (code blocks / implementations with `qo push`/`qo run`, data blocks / databases with `qo query`/`qo describe`/`qo write`, config blocks / push connectors with `qo set-push-connector`, log blocks / log streams with `qo logs`), and the "pages are the interface" principle (everything is a page operation -- the page is the unit of everything). Keywords: page, page tree, block, code block, data block, push connector, log stream, page ID, folder, qo ls, qo query, qo describe, qo logs.

- [Components](components.md) -- The component/protocol/implementation architecture. Covers protocols (interface definitions with method names and arities), the four standard protocols (Installable, Service, Event Handler, Query), components (named units of code that implement protocols), implementations (OQL code blocks that fulfill protocol methods), push time vs runtime distinction (`$$ARG$$` at push time, `Exec` object at runtime), `set-implementation-clauses` registration, libraries vs installs (source code vs running instances), the Exec object structure, event subscriptions (event types, sub-locations, handler funcs, event flow diagram), and subscription registration. Keywords: component, protocol, implementation, Installable, Service, Event Handler, Query, Exec, $$ARG$$, push time, runtime, set-implementation-clauses, library, install, event subscription, handler func, emit-event.

- [Plugins](plugins.md) -- Plugin lifecycle and the actor model. Covers what a plugin looks like (the page tree created by install), the Unix-style philosophy (do one thing well, inspectable state, standard lifecycle, composable, config as data), the full plugin lifecycle via `qo run` (install, start, stop, status, trigger), the actor model for inter-plugin communication (events as messages, subscriptions as mailboxes, isolated handlers, no shared mutable state, batch processing via continuation events), building a plugin (component library workflow, shared library usage), and debugging (querying log streams directly via mango, listing event subscriptions). Keywords: plugin, install, lifecycle, actor model, event, subscription, continuation, batch processing, Unix philosophy, composable, qo run, debugging, log stream, mango query.
