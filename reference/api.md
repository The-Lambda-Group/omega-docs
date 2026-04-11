[< Home](../README.md) | [Reference](README.md)

# Public API Reference

All public OQL API clauses. These are the stable interfaces for component code -- never call internal datastores directly.

## Qo.Public.OqlApi.Page

Page tree navigation and mutation.

Datastore: `omega/query-omega/public/oql-api/page`

| Clause | Signature |
|--------|-----------|
| `root-folder` | `(Qo.Public.OqlApi.Page/root-folder AppId RootFolderId)` |
| `page-by-id` | `(Qo.Public.OqlApi.Page/page-by-id PageId Page)` |
| `pages-by-folder` | `(Qo.Public.OqlApi.Page/pages-by-folder FolderId Page)` |
| `parent-page` | `(Qo.Public.OqlApi.Page/parent-page Page ParentPage)` |
| `child-by-name` | `(Qo.Public.OqlApi.Page/child-by-name ParentPageId Name ChildPage)` |
| `child-page-by-name` | `(Qo.Public.OqlApi.Page/child-page-by-name ParentPage Name ChildPage)` |
| `child-page` | `(Qo.Public.OqlApi.Page/child-page ParentPage ChildPage)` |
| `add-sub-page` | `(Qo.Public.OqlApi.Page/add-sub-page ParentPage PageName Result)` |
| `add-or-get-sub-page-by-name` | `(Qo.Public.OqlApi.Page/add-or-get-sub-page-by-name Parent PageName SubPage)` |
| `add-sub-database` | `(Qo.Public.OqlApi.Page/add-sub-database ParentPage PageName Result)` |
| `add-or-get-sub-database-by-name` | `(Qo.Public.OqlApi.Page/add-or-get-sub-database-by-name Parent PageName SubPage)` |
| `page-query` | `(Qo.Public.OqlApi.Page/page-query Q Page)` |
| `page-search` | `(Qo.Public.OqlApi.Page/page-search Q Page)` |
| `page-blocks` | `(Qo.Public.OqlApi.Page/page-blocks AppId PageId Blocks)` |
| `add-table-view` | `(Qo.Public.OqlApi.Page/add-table-view Page ViewName Result)` |
| `add-or-get-table-view` | `(Qo.Public.OqlApi.Page/add-or-get-table-view Page ViewName Result)` |

## Qo.Public.OqlApi.Page.Bl.LogStream

Log stream blocks on pages.

Datastore: `omega/query-omega/public/oql-api/page/block/log-stream`

| Clause | Signature |
|--------|-----------|
| `add-by-name` | `(Qo.Public.OqlApi.Page.Bl.LogStream/add-by-name Page Name Block)` |

## Qo.Public.OqlApi.Page.Bl.PushCon

Push connector blocks on pages.

Datastore: `omega/query-omega/public/oql-api/page/block/push-connector`

| Clause | Signature |
|--------|-----------|
| `add-by-name` | `(Qo.Public.OqlApi.Page.Bl.PushCon/add-by-name Page Name Block)` |
| `get-by-name` | `(Qo.Public.OqlApi.Page.Bl.PushCon/get-by-name Page Name PushConSpec)` |
| `set-data` | `(Qo.Public.OqlApi.Page.Bl.PushCon/set-data Page Name NewSpec Result)` |

## Qo.Public.OqlApi.Comp

Component definitions.

Datastore: `omega/query-omega/public/oql-api/component`

| Clause | Signature |
|--------|-----------|
| `component` | `(Qo.Public.OqlApi.Comp/component AppId CompId Comp)` |
| `add-or-get-by-name` | `(Qo.Public.OqlApi.Comp/add-or-get-by-name Page Name Comp)` |

## Qo.Public.OqlApi.Proto

Protocol definitions.

Datastore: `omega/query-omega/public/oql-api/protocol`

| Clause | Signature |
|--------|-----------|
| `protocol` | `(Qo.Public.OqlApi.Proto/protocol AppId ProtoId Proto)` |
| `add-or-get-by-name` | `(Qo.Public.OqlApi.Proto/add-or-get-by-name Page Name Spec Proto)` |

## Qo.Public.OqlApi.Impl

Implementation management.

Datastore: `omega/query-omega/public/oql-api/implementation`

| Clause | Signature |
|--------|-----------|
| `set-implementation-clauses` | `(Qo.Public.OqlApi.Impl/set-implementation-clauses ImplId Clauses Result)` |
| `implementation` | `(Qo.Public.OqlApi.Impl/implementation AppId ImplId CompId ProtoId Impl)` |
| `add-or-get` | `(Qo.Public.OqlApi.Impl/add-or-get Page Proto Comp Name Impl)` |

## Qo.Public.OqlApi.Db.Prop

Database property system -- schema, primary keys, secondary indexes, table writes.

Datastore: `omega/query-omega/public/oql-api/database/property`

| Clause | Signature |
|--------|-----------|
| `init-properties` | `(Qo.Public.OqlApi.Db.Prop/init-properties Page Columns Result)` |
| `init-primary-key` | `(Qo.Public.OqlApi.Db.Prop/init-primary-key Page PKey Result)` |
| `init-sec-indexes` | `(Qo.Public.OqlApi.Db.Prop/init-sec-indexes Page SecIndexes Result)` |
| `register-sec-index` | `(Qo.Public.OqlApi.Db.Prop/register-sec-index Page Column Result)` |
| `query-by-index` | `(Qo.Public.OqlApi.Db.Prop/query-by-index Q Result)` |
| `write-table` | `(Qo.Public.OqlApi.Db.Prop/write-table Input Result)` |
| `truncate-database` | `(Qo.Public.OqlApi.Db.Prop/truncate-database FolderId Result)` |

## Qo.Public.OqlApi.PageSub

Page event subscriptions -- register, emit, and query subscriptions.

Datastore: `omega/query-omega/public/oql-api/page-subscription`

| Clause | Signature |
|--------|-----------|
| `write-page-subscription` | `(Qo.Public.OqlApi.PageSub/write-page-subscription Page HandlerFunc Result)` |
| `delete-page-subscription` | `(Qo.Public.OqlApi.PageSub/delete-page-subscription Page Result)` |
| `get-page-subscription` | `(Qo.Public.OqlApi.PageSub/get-page-subscription Page Result)` |
| `emit-page-subscription` | `(Qo.Public.OqlApi.PageSub/emit-page-subscription Page EventData Result)` |
| `page-event-handler` | `(Qo.Public.OqlApi.PageSub/page-event-handler ComponentPage HandlerFunc)` |
| `installed-page-event-handler` | `(Qo.Public.OqlApi.PageSub/installed-page-event-handler InstalledPage HandlerFunc)` |

## Qo.Public.Api.Run

Run implementations by page reference or page ID.

Datastore: `omega/query-omega/public/api/run`

| Clause | Signature |
|--------|-----------|
| `run-data` | `(Qo.Public.Api.Run/run-data Request Response)` |
| `run-page` | `(Qo.Public.Api.Run/run-page Proto Page Name Args Result)` |
| `run-page-by-id` | `(Qo.Public.Api.Run/run-page-by-id PageId ProtoName Name Args Result)` |

## Qo.Public.Api.LogStream

Read log stream data.

Datastore: `omega/query-omega/public/api/log-stream`

| Clause | Signature |
|--------|-----------|
| `get-log-stream` | `(Qo.Public.Api.LogStream/get-log-stream Input Result)` |
| `read-logs` | `(Qo.Public.Api.LogStream/read-logs Input Result)` |
