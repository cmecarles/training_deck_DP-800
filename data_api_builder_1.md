# SQL Server question — Data API builder 1

## Statement

A city museum stores its art collection in an Azure SQL database named `MuseumHub`.

The database contains the following objects in the `Art` schema:

- **`Art.Exhibits`** — A table with a primary key on `ExhibitId`. It contains all exhibits, including items in restoration that are not shown to the public.
- **`Art.PublicCatalog`** — A view over `Art.Exhibits` that projects only publicly visible items. The view exposes the columns `CatalogId`, `Title`, `Artist`, and `Era`. Because it is a view, the database defines **no primary key** on it.
- **`Art.GetExhibitsByEra`** — A stored procedure with a single parameter `@Era nvarchar(50)`. When the parameter is omitted it should default to `Modern`.

The museum wants to expose the database as an API by using **Data API builder (DAB)** with both the REST and GraphQL endpoints enabled at the runtime level, and with global caching enabled in the `runtime` section (`runtime.cache.enabled: true`).

You must configure the `entities` section of `dab-config.json` to meet **all** of the following requirements:

1. Anonymous visitors must be able to read the `PublicCatalog` entity through REST and GraphQL, including retrieving a **single item by key**, for example:

   ```http
   GET /api/PublicCatalog/CatalogId/17
   ```

2. Responses for the `PublicCatalog` entity must be **cached for 60 seconds**.
3. Anonymous visitors must be able to call the stored procedure through a REST **GET** request, passing the parameter in the query string, for example:

   ```http
   GET /api/GetExhibitsByEra?Era=Baroque
   ```

4. In the GraphQL schema, the stored procedure must be exposed under the **`Query`** type, not under `Mutation`, because it only reads data.
5. Anonymous visitors must be able to **read** the `Exhibit` entity (backed by `Art.Exhibits`), and the custom role `curator` must be able to create, read, update, and delete exhibits.
6. The anonymous role must never be able to modify data through any entity.

Which `entities` fragment of `dab-config.json` should you use?

### a.

```json
{
  "entities": {
    "Exhibit": {
      "source": {
        "type": "table",
        "object": "Art.Exhibits"
      },
      "permissions": [
        { "role": "anonymous", "actions": [ "read" ] },
        { "role": "curator", "actions": [ "create", "read", "update", "delete" ] }
      ]
    },
    "PublicCatalog": {
      "source": {
        "type": "view",
        "object": "Art.PublicCatalog",
        "key-fields": [ "CatalogId" ]
      },
      "cache": { "enabled": true, "ttl-seconds": 60 },
      "permissions": [
        { "role": "anonymous", "actions": [ "read" ] }
      ]
    },
    "GetExhibitsByEra": {
      "source": {
        "type": "stored-procedure",
        "object": "Art.GetExhibitsByEra",
        "parameters": [
          { "name": "Era", "required": false, "default": "Modern" }
        ]
      },
      "rest": { "methods": [ "get" ] },
      "graphql": { "operation": "query" },
      "permissions": [
        { "role": "anonymous", "actions": [ "read" ] }
      ]
    }
  }
}
```

### b.

```json
{
  "entities": {
    "Exhibit": {
      "source": {
        "type": "table",
        "object": "Art.Exhibits"
      },
      "permissions": [
        { "role": "anonymous", "actions": [ "read" ] },
        { "role": "curator", "actions": [ "create", "read", "update", "delete" ] }
      ]
    },
    "PublicCatalog": {
      "source": {
        "type": "view",
        "object": "Art.PublicCatalog",
        "key-fields": [ "CatalogId" ]
      },
      "cache": { "enabled": true, "ttl-seconds": 60 },
      "permissions": [
        { "role": "anonymous", "actions": [ "read" ] }
      ]
    },
    "GetExhibitsByEra": {
      "source": {
        "type": "stored-procedure",
        "object": "Art.GetExhibitsByEra",
        "parameters": [
          { "name": "Era", "required": false, "default": "Modern" }
        ]
      },
      "rest": { "methods": [ "get" ] },
      "graphql": { "operation": "query" },
      "permissions": [
        { "role": "anonymous", "actions": [ "execute" ] }
      ]
    }
  }
}
```

### c.

```json
{
  "entities": {
    "Exhibit": {
      "source": {
        "type": "table",
        "object": "Art.Exhibits"
      },
      "permissions": [
        { "role": "anonymous", "actions": [ "read" ] },
        { "role": "curator", "actions": [ "create", "read", "update", "delete" ] }
      ]
    },
    "PublicCatalog": {
      "source": {
        "type": "view",
        "object": "Art.PublicCatalog"
      },
      "cache": { "enabled": true, "ttl-seconds": 60 },
      "permissions": [
        { "role": "anonymous", "actions": [ "read" ] }
      ]
    },
    "GetExhibitsByEra": {
      "source": {
        "type": "stored-procedure",
        "object": "Art.GetExhibitsByEra",
        "parameters": [
          { "name": "Era", "required": false, "default": "Modern" }
        ]
      },
      "rest": { "methods": [ "get" ] },
      "graphql": { "operation": "query" },
      "permissions": [
        { "role": "anonymous", "actions": [ "execute" ] }
      ]
    }
  }
}
```

### d.

```json
{
  "entities": {
    "Exhibit": {
      "source": {
        "type": "table",
        "object": "Art.Exhibits"
      },
      "permissions": [
        { "role": "anonymous", "actions": [ "read" ] },
        { "role": "curator", "actions": [ "create", "read", "update", "delete" ] }
      ]
    },
    "PublicCatalog": {
      "source": {
        "type": "view",
        "object": "Art.PublicCatalog",
        "key-fields": [ "CatalogId" ]
      },
      "cache": { "enabled": true, "ttl-seconds": 60 },
      "permissions": [
        { "role": "anonymous", "actions": [ "read" ] }
      ]
    },
    "GetExhibitsByEra": {
      "source": {
        "type": "stored-procedure",
        "object": "Art.GetExhibitsByEra",
        "parameters": [
          { "name": "Era", "required": false, "default": "Modern" }
        ]
      },
      "rest": { "enabled": true },
      "graphql": { "enabled": true },
      "permissions": [
        { "role": "anonymous", "actions": [ "execute" ] }
      ]
    }
  }
}
```

## Correct Answer

**b**

## Explanation

The correct answer is **b**.

This question tests three DAB configuration concepts that behave differently for each kind of database object:

1. **Views** do not carry a primary key, so DAB cannot infer one. The entity must declare `source.key-fields` (or, in DAB 2.0, a `fields` array with `primary-key: true`) so that single-item REST routes such as `/api/PublicCatalog/CatalogId/17` can address a row.
2. **Stored procedures** have non-obvious defaults: the REST endpoint supports **only `POST`** unless `rest.methods` explicitly lists `get`, and the GraphQL field is placed under **`Mutation`** unless `graphql.operation` is set to `query`.
3. **Permission actions depend on the object type**: `create`/`read`/`update`/`delete` apply to tables and views, while stored procedures accept only `execute` (the `*` wildcard expands to `execute` for a stored procedure).

### How option b satisfies every requirement

| Requirement | Configuration key in option b |
| --- | --- |
| 1. Anonymous read of the view, including single item by key | `PublicCatalog.source.type: "view"` + `source.key-fields: [ "CatalogId" ]` + `permissions` role `anonymous` with action `read` |
| 2. Cache view responses for 60 seconds | `PublicCatalog.cache.enabled: true` + `cache.ttl-seconds: 60` (entity caching is **disabled by default**, so `enabled` must be set; it takes effect only because the statement also enables `runtime.cache.enabled` globally) |
| 3. Call the procedure with REST `GET` and query-string parameters | `GetExhibitsByEra.source.type: "stored-procedure"` + `rest.methods: [ "get" ]` (overrides the `POST`-only default; `GET` sends parameters via the query string) |
| 4. Procedure under the GraphQL `Query` type | `GetExhibitsByEra.graphql.operation: "query"` (overrides the default of `mutation`) |
| 5. Anonymous read-only plus full CRUD for `curator` on the table | `Exhibit.permissions`: role `anonymous` with `read`; role `curator` with `create`, `read`, `update`, `delete` |
| 6. Anonymous can never modify data | Anonymous holds only `read` on `Exhibit` and `PublicCatalog` and `execute` on a procedure that only reads; no `create`/`update`/`delete` is granted to `anonymous` anywhere |

The `parameters` array on the stored-procedure source supplies the default value `Modern` for `@Era`, so the parameter can be omitted by callers.

### Why option a is wrong

Option a grants the anonymous role the following on the stored-procedure entity:

```json
"permissions": [
  { "role": "anonymous", "actions": [ "read" ] }
]
```

The `read` action is valid only for tables and views. A `stored-procedure` entity supports exactly one action: **`execute`**. Because `execute` is never granted, anonymous visitors cannot invoke `GetExhibitsByEra` at all, violating requirement 3 (and requirement 4 becomes moot because the role cannot reach the operation).

The violating key is `entities.GetExhibitsByEra.permissions[0].actions` — it must contain `"execute"`, not `"read"`.

### Why option c is wrong

Option c defines the view source as:

```json
"source": {
  "type": "view",
  "object": "Art.PublicCatalog"
}
```

The `key-fields` property is missing. For an entity whose `source.type` is `view` (and that does not use the DAB 2.0 `fields` array with `primary-key: true`), `key-fields` is **required**: a view has no primary key that DAB can discover, so without declared key fields DAB cannot uniquely address a row, and the single-item route `GET /api/PublicCatalog/CatalogId/17` from requirement 1 cannot work.

The violating (absent) key is `entities.PublicCatalog.source.key-fields`.

### Why option d is wrong

Option d configures the stored-procedure endpoints as:

```json
"rest": { "enabled": true },
"graphql": { "enabled": true }
```

Enabling the endpoints is not enough — this option silently accepts **both** stored-procedure defaults:

- `rest.methods` is omitted, so the REST endpoint accepts **only `POST`**. The request `GET /api/GetExhibitsByEra?Era=Baroque` fails, violating requirement 3.
- `graphql.operation` is omitted, so it defaults to **`mutation`**, placing `executeGetExhibitsByEra` under the GraphQL `Mutation` type and violating requirement 4.

The violating keys are the missing `entities.GetExhibitsByEra.rest.methods` (must be `[ "get" ]`) and the missing `entities.GetExhibitsByEra.graphql.operation` (must be `"query"`).

## DP-800 Exam Rule to Remember

In a DAB `dab-config.json`, the object type drives the configuration:

```text
table            → actions: create / read / update / delete
view             → same actions as a table
                   + key-fields REQUIRED (a view has no primary key)
stored-procedure → action: execute ONLY
                   + REST defaults to POST only  → set rest.methods: ["get"] for GET
                   + GraphQL defaults to mutation → set graphql.operation: "query"
```

Two more defaults worth memorizing:

- Entity caching is **off** by default: you must set `cache.enabled: true`, and `ttl-seconds` controls the lifetime (inheriting the runtime value when omitted). Entity-level cache works only when `runtime.cache.enabled` is also `true` — if the global cache is off, the entity-level setting has no effect.
- `source.parameters` is needed only for stored-procedure parameters that require a default value; parameters without defaults can simply be passed by the caller.

When an exam option exposes a stored procedure, immediately check three keys: the permission **action** (`execute`), the REST **methods** (`POST`-only default), and the GraphQL **operation** (`mutation` default).
