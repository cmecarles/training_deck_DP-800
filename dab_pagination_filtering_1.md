# SQL Server question — DAB Pagination and Filtering 1

## Statement

A plant nursery exposes its catalogue through **Data API builder (DAB)** in front of the Azure SQL database `SeedVault`:

```sql
CREATE DATABASE SeedVault;
GO
USE SeedVault;
GO
CREATE SCHEMA Catalog;
GO
CREATE TABLE Catalog.Plant
(
    PlantId    INT           NOT NULL PRIMARY KEY,
    CommonName NVARCHAR(80)  NOT NULL,
    Price      DECIMAL(8,2)  NOT NULL,
    InStock    BIT           NOT NULL,
    AddedOn    DATETIME2(0)  NOT NULL
);
GO
```

The entity is defined in `dab-config.json` with **mappings** that rename the columns for the API, and a custom GraphQL type name:

```json
{
  "entities": {
    "Plant": {
      "source": { "type": "table", "object": "Catalog.Plant" },
      "mappings": { "PlantId": "plantId", "CommonName": "name", "Price": "price",
                    "InStock": "inStock", "AddedOn": "addedOn" },
      "graphql": { "type": { "singular": "plant", "plural": "plants" } },
      "permissions": [ { "role": "anonymous", "actions": [ "read" ] } ]
    }
  }
}
```

The web team must implement a catalogue page that satisfies **all** of the following:

1. A REST request returns the **first 20** plants that cost **more than 100** and are in stock, sorted by **name ascending, then price descending**, returning **only the `name` and `price` fields**.
2. The client fetches subsequent pages by **continuing from where the previous page ended**, without missing or duplicating rows even if plants are inserted between requests, until there are no more rows.
3. The same first page is available through GraphQL, and the response must tell the client whether another page exists and how to request it.
4. A caller must never get more than **500** rows in one response; a request that does not specify a page size must return **25** rows; a request for the "maximum" page size must return at most 500.
5. GraphQL queries deeper than **three** nested levels must be rejected (to stop abusive nested relationship queries).

Which combination of requests and runtime configuration meets every requirement?

### a.

```http
GET /api/Plant?$filter=price gt 100 and inStock eq true&$orderby=name asc, price desc&$select=name,price&$first=20
```

Next pages: follow the `nextLink` property of each response (it carries `$after=<token>`) until a response has no `nextLink`.

```graphql
{
  plants(first: 20,
         filter: { and: [ { price: { gt: 100 } }, { inStock: { eq: true } } ] },
         orderBy: { name: ASC, price: DESC }) {
    items { name price }
    hasNextPage
    endCursor
  }
}
```

```json
"runtime": {
  "pagination": { "default-page-size": 25, "max-page-size": 500 },
  "graphql":    { "depth-limit": 3 }
}
```

### b.

```http
GET /api/Plant?$filter=price > 100 and inStock eq true&$orderby=name asc, price desc&$select=name,price&$top=20
```

Next pages: add `$skip=20`, `$skip=40`, ... to the same URL.

```graphql
{ plants(first: 20, offset: 20, filter: { price: { gt: 100 } }) { items { name price } } }
```

```json
"runtime": {
  "pagination": { "default-page-size": 25, "max-page-size": -1 },
  "graphql":    { "depth-limit": 3 }
}
```

### c.

```http
GET /api/Plant?$filter=price gt 100 and inStock eq true&$orderby=name asc, price desc&$select=name,price&$first=20
```

Next pages: read the `name` of the last item and request
`$filter=price gt 100 and inStock eq true and name gt '<last name>'` with the same `$orderby`, `$select` and `$first`.

```graphql
{
  plants(first: 20,
         filter: { and: [ { price: { gt: 100 } }, { inStock: { eq: true } } ] },
         orderBy: { name: ASC, price: DESC }) {
    items { name price }
    hasNextPage
    endCursor
  }
}
```

```json
"runtime": {
  "pagination": { "default-page-size": 25, "max-page-size": 500, "depth-limit": 3 }
}
```

### d.

```http
GET /api/Plant?$filter=price gt 100 && inStock eq true&$orderby=name asc, price desc&$select=CommonName,Price&$first=20
```

Next pages: follow `nextLink`.

```graphql
{
  plants(first: 20,
         filter: { and: [ { price: { ge: 100 } }, { inStock: { eq: true } } ] },
         orderBy: { name: ASC, price: DESC }) {
    items { name price }
    hasNextPage
    endCursor
  }
}
```

```json
"runtime": {
  "pagination": { "default-page-size": 25, "max-page-size": 500 },
  "graphql":    { "depth-limit": 3 }
}
```

## Correct Answer

**a**

## Explanation

DAB's read API has a small, fixed vocabulary. In REST it is OData-inspired: `$select` (projection), `$filter` (predicate), `$orderby` (sort), `$first` (page size) and `$after` (continuation). In GraphQL the same concepts are the `items` selection, the `filter` input object, `orderBy`, `first` and `after`, plus the `hasNextPage` / `endCursor` fields of every list result. Page limits live in `runtime.pagination`, and the query-depth guard in `runtime.graphql`.

### Why option a is correct

- **Requirement 1 — the REST query.** `$filter` supports the comparison operators `eq`, `ne`, `gt`, `ge`, `lt`, `le`, the logical operators `and`, `or`, `not` and parentheses; string literals are single-quoted, booleans and numbers are bare, so `price gt 100 and inStock eq true` is valid. `$orderby` accepts a comma-separated list with `asc`/`desc` per field (`name asc, price desc`). `$select=name,price` projects the two fields; because the entity declares `mappings`, requests and responses use the **exposed** names (`name`, `price`), never the column names. `$first=20` caps the page at 20 rows. Every filter compiles to parameterized SQL.
- **Requirement 2 — continuation.** DAB implements **keyset pagination**: when more rows exist, the response carries a `nextLink` whose URL includes an opaque, base64-encoded `$after` token that "marks the position of the last record from the previous page", so the next request "continues from that point" and, unlike offset paging, "prevents missing or duplicated rows when data changes between requests". The token embeds the ordering and filter, must be treated as immutable, and is invalidated by any schema or ordering change. The last page has **no** `nextLink` — that is the termination condition.
- **Requirement 3 — GraphQL.** The list field takes `first`, `after`, `filter` and `orderBy`. Filters are structured input objects `{ field: { operator: value } }`; the GraphQL operator names are `eq`, `neq`, `gt`, `gte`, `lt`, `lte`, `contains`, `notContains`, `startsWith`, `endsWith`, `in` and `isNull`, combined with `and: [...]` / `or: [...]`. `orderBy: { name: ASC, price: DESC }` sorts by both fields. Selecting `hasNextPage` and `endCursor` gives the client the "is there more?" flag and the cursor to pass as `after: "<endCursor>"` on the next call.
- **Requirement 4 — page limits.** `runtime.pagination.default-page-size` (default 100) is used when `$first`/`first` is omitted; `runtime.pagination.max-page-size` (default 100,000) is the hard ceiling: a `$first` above it is rejected with **400**, `$first=0` or values below `-1` are 400 too, and `$first=-1` (or `first: -1`) is "expanded to `max-page-size`". Setting them to 25 and 500 satisfies all three sub-requirements. `default-page-size` must be a positive integer **less than** `max-page-size`.
- **Requirement 5 — depth.** `runtime.graphql.depth-limit` (default `null`, unlimited) is "the maximum allowed depth of a GraphQL query"; `3` rejects deeper nested-relationship queries.

### Why option b is wrong

`$top` and `$skip` are OData keywords that DAB **does not implement** — its page-size parameter is `$first` and its continuation parameter is `$after`; likewise `>` is not a `$filter` operator (`gt` is), and there is no `offset` argument in the GraphQL schema. Even if offset paging existed, it is exactly the pattern the documentation contrasts with keyset paging as one that "misses or duplicates rows when data changes between requests" (requirement 2). Finally, `"max-page-size": -1` does not mean 500: `-1` "defaults to the maximum supported value", so a caller could ask for far more than 500 rows (requirement 4).

### Why option c is wrong

This is the subtle distractor: the first-page request and the GraphQL query are identical to option a, and hand-built keyset paging on `name gt '<last name>'` even appears to work on a small catalogue. It breaks requirement 2 as soon as two plants share a `name` (the second one on the boundary is skipped) and it ignores the secondary sort key `price desc` — the opaque `$after` token exists precisely because DAB encodes *all* the ordering columns in it. The configuration is also invalid: `depth-limit` is a property of `runtime.graphql`, not of `runtime.pagination`, so requirement 5 is not enforced.

### Why option d is wrong

Three syntax errors, each producing a 400 or a schema error rather than a page of plants:

- `&&` is not a `$filter` operator; the logical operators are the words `and`, `or`, `not`.
- `$select=CommonName,Price` uses the **database column names**; with `mappings` in place the API only knows `name` and `price`, and "if a field is missing or not accessible, DAB returns 400 Bad Request".
- In GraphQL the greater-or-equal operator is `gte`; `ge` belongs to the REST `$filter` grammar and is not a field of the GraphQL filter input type, so the query fails validation.

Conceptual question (Azure / tooling); not executed against an engine.

## DP-800 Exam Rule to Remember

```text
REST  (/api/<Entity>)                      GraphQL (POST /graphql)
  $select=f1,f2                              items { f1 f2 }
  $filter=price gt 100 and inStock eq true   filter: { and: [ {price:{gt:100}}, {inStock:{eq:true}} ] }
     ops: eq ne gt ge lt le, and or not ( )     ops: eq neq gt gte lt lte contains notContains
     strings 'quoted', eq null / ne null            startsWith endsWith in isNull, and:[ ] or:[ ]
  $orderby=name asc, price desc              orderBy: { name: ASC, price: DESC }
  $first=N   (-1 = max-page-size; 0 / >max -> 400)   first: N
  $after=<opaque token from nextLink>        after: "<endCursor>"  + hasNextPage
  last page: no nextLink                     hasNextPage: false
  single item: /api/<Entity>/<key>/<value>   <entity>_by_pk(<key>: <value>)

runtime.pagination.default-page-size (100) < runtime.pagination.max-page-size (100000)
runtime.graphql.depth-limit (null = unlimited)   runtime.graphql.multiple-mutations.create.enabled (false)
mappings -> requests and responses use the EXPOSED names, never the column names
```

Keyset (`$after`) paging is the only continuation mechanism DAB offers: there is no `$top`, `$skip` or `offset`. If an option builds its own "next page" predicate or uses offsets, it is the wrong one.
