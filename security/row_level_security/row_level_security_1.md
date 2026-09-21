# SQL Server question — Row-Level Security 1

## Statement

An organization hosts a multi-tenant application in Azure SQL Database. All application users connect to the database through the same database principal, `AppUser`.

At the beginning of each request, the application sets the tenant ID:

```sql
EXEC sys.sp_set_session_context
    @key = N'tenant_id',
    @value = 42;
```

The following table stores orders for all tenants:

```sql
CREATE TABLE Sales.Orders
(
    OrderId   bigint NOT NULL PRIMARY KEY,
    TenantId  int NOT NULL,
    Amount    decimal(12,2) NOT NULL
);
```

You must implement security that meets these requirements:

- `SELECT` returns only rows belonging to the tenant stored in `SESSION_CONTEXT`.
- A tenant can `UPDATE` or `DELETE` only its own existing rows.
- An `INSERT` that specifies another tenant's `TenantId` must fail.
- An `UPDATE` that changes an existing row's `TenantId` to another tenant must fail.
- Enforcement must occur in the database and must not depend on application `WHERE` clauses.

You create the schema:

```sql
CREATE SCHEMA Security;
GO
```

Which implementation should you use?

### a.

```sql
CREATE FUNCTION Security.fn_TenantPredicate
(
    @TenantId int
)
RETURNS TABLE
WITH SCHEMABINDING
AS
RETURN
(
    SELECT 1 AS AccessResult
    WHERE @TenantId =
          CAST(SESSION_CONTEXT(N'tenant_id') AS int)
);
GO

CREATE SECURITY POLICY Security.TenantPolicy
ADD FILTER PREDICATE
    Security.fn_TenantPredicate(TenantId)
    ON Sales.Orders
WITH (STATE = ON);
GO
```

### b.

```sql
CREATE FUNCTION Security.fn_TenantPredicate
(
    @TenantId int
)
RETURNS TABLE
WITH SCHEMABINDING
AS
RETURN
(
    SELECT 1 AS AccessResult
    WHERE @TenantId =
          CAST(SESSION_CONTEXT(N'tenant_id') AS int)
);
GO

CREATE SECURITY POLICY Security.TenantPolicy
ADD BLOCK PREDICATE
    Security.fn_TenantPredicate(TenantId)
    ON Sales.Orders AFTER INSERT,
ADD BLOCK PREDICATE
    Security.fn_TenantPredicate(TenantId)
    ON Sales.Orders AFTER UPDATE
WITH (STATE = ON);
GO
```

### c.

```sql
CREATE FUNCTION Security.fn_TenantPredicate
(
    @TenantId int
)
RETURNS TABLE
WITH SCHEMABINDING
AS
RETURN
(
    SELECT 1 AS AccessResult
    WHERE @TenantId =
          CAST(SESSION_CONTEXT(N'tenant_id') AS int)
);
GO

CREATE SECURITY POLICY Security.TenantPolicy
ADD FILTER PREDICATE
    Security.fn_TenantPredicate(TenantId)
    ON Sales.Orders,
ADD BLOCK PREDICATE
    Security.fn_TenantPredicate(TenantId)
    ON Sales.Orders AFTER INSERT,
ADD BLOCK PREDICATE
    Security.fn_TenantPredicate(TenantId)
    ON Sales.Orders AFTER UPDATE
WITH (STATE = ON);
GO
```

### d.

```sql
CREATE FUNCTION Security.fn_TenantPredicate
(
    @TenantId int
)
RETURNS TABLE
WITH SCHEMABINDING
AS
RETURN
(
    SELECT 1 AS AccessResult
    WHERE @TenantId =
          CAST(SESSION_CONTEXT(N'tenant_id') AS int)
);
GO

CREATE SECURITY POLICY Security.TenantPolicy
ADD FILTER PREDICATE
    Security.fn_TenantPredicate(TenantId)
    ON Sales.Orders,
ADD BLOCK PREDICATE
    Security.fn_TenantPredicate(TenantId)
    ON Sales.Orders AFTER INSERT,
ADD BLOCK PREDICATE
    Security.fn_TenantPredicate(TenantId)
    ON Sales.Orders BEFORE UPDATE
WITH (STATE = ON);
GO
```

## Correct Answer

### Option C — FILTER + AFTER INSERT + AFTER UPDATE

Option c combines:

1. **FILTER PREDICATE**
   - Restricts `SELECT` to the current tenant's rows.
   - Restricts `UPDATE` and `DELETE` to the current tenant's existing rows.
2. **BLOCK PREDICATE AFTER INSERT**
   - Prevents creating rows for another tenant.
3. **BLOCK PREDICATE AFTER UPDATE**
   - Prevents modifying a row so that it belongs to another tenant.

```mermaid
sequenceDiagram
    actor User as Tenant 42
    participant App
    participant DB as Sales.Orders
    participant Filter as FILTER Predicate
    participant Block as BLOCK Predicate

    User->>App: SELECT / UPDATE / DELETE
    App->>DB: Execute operation
    DB->>Filter: Check existing TenantId
    Filter-->>DB: Allow only TenantId = 42
    DB-->>User: Only own rows accessible

    User->>App: INSERT TenantId = 99
    App->>DB: INSERT
    DB->>Block: AFTER INSERT: check new TenantId
    Block-->>DB: 99 != 42 → Reject
    DB-->>User: Insert fails

    User->>App: INSERT TenantId = 42
    App->>DB: INSERT
    DB->>Block: AFTER INSERT: check new TenantId
    Block-->>DB: 42 == 42 → Allow
    DB-->>User: Insert succeeds

    User->>App: UPDATE own row TenantId 42 → 99
    App->>DB: UPDATE
    DB->>Filter: Can user access existing row?
    Filter-->>DB: 42 == 42 → Yes
    DB->>Block: AFTER UPDATE: check new TenantId
    Block-->>DB: 99 != 42 → Reject
    DB-->>User: Update fails

    Note over User,DB: ✅ Existing rows isolated<br/>✅ Invalid inserts blocked<br/>✅ Invalid new values blocked
```

## Explanation

### Option A — FILTER only

Option A violates both the `INSERT` requirement and the `UPDATE`-of-`TenantId` requirement.

A FILTER predicate restricts which existing rows are visible or accessible to `SELECT`, `UPDATE`, and `DELETE` operations. 

However, a FILTER predicate by itself does not prevent a user from inserting a row that belongs to another tenant. 

It also does not prevent an `UPDATE` that changes a visible row's `TenantId` to another tenant: the filter is evaluated against the existing row (which the tenant owns), the update succeeds, and the row simply disappears from the tenant's view afterwards. 

```mermaid
sequenceDiagram
    actor User as Tenant 42
    participant App
    participant DB as Sales.Orders
    participant RLS as FILTER Predicate

    User->>App: SELECT / UPDATE / DELETE
    App->>DB: Execute operation
    DB->>RLS: Check existing row TenantId
    RLS-->>DB: Allow only TenantId = 42
    DB-->>User: Only own rows accessible

    User->>App: INSERT row with TenantId = 99
    App->>DB: INSERT
    Note over DB,RLS: FILTER does not block INSERT
    DB-->>User: Insert succeeds

    User->>App: SELECT inserted row
    App->>DB: SELECT
    DB->>RLS: TenantId 99 == Session 42?
    RLS-->>DB: No
    DB-->>User: Row hidden

    Note over User,DB: ❌ Tenant 42 can create data for another tenant
```

### Option B — BLOCK predicates only

Option b uses BLOCK predicates to protect `INSERT` and `UPDATE` operations, but it does not contain a FILTER predicate. As a result, it does not restrict which existing rows `AppUser` can see.

```mermaid
sequenceDiagram
    actor User as Tenant 42
    participant App
    participant DB as Sales.Orders
    participant Block as BLOCK Predicate

    User->>App: SELECT
    App->>DB: SELECT all rows
    Note over DB,Block: No FILTER predicate
    DB-->>User: Rows for Tenant 42 AND Tenant 99

    Note over User,DB: ❌ Existing rows are not isolated

    User->>App: INSERT TenantId = 99
    App->>DB: INSERT
    DB->>Block: AFTER INSERT check
    Block-->>DB: 99 != 42 → Reject
    DB-->>User: Insert fails

    User->>App: UPDATE own row TenantId 42 → 99
    App->>DB: UPDATE
    DB->>Block: AFTER UPDATE check
    Block-->>DB: 99 != 42 → Reject
    DB-->>User: Update fails

    Note over User,DB: ✅ Writes protected<br/>❌ Reads not protected
```

### Option D — FILTER + AFTER INSERT + BEFORE UPDATE

A `BEFORE UPDATE` block predicate evaluates the existing row before the modification. For example, assume tenant 42 owns this row:

| OrderId | TenantId | Amount |
|---|---|---|
| 1001 | 42 | 500 |

Tenant 42 attempts:

```sql
UPDATE Sales.Orders
SET TenantId = 99
WHERE OrderId = 1001;
```

A `BEFORE UPDATE` predicate evaluates the original `TenantId` value of 42, which is valid for the current tenant. It therefore does not enforce the requirement that the new `TenantId` must also belong to the current tenant.

An `AFTER UPDATE` block predicate evaluates the resulting row state. It can therefore reject the change when `TenantId` becomes 99.

```mermaid
sequenceDiagram
    actor User as Tenant 42
    participant App
    participant DB as Sales.Orders
    participant Filter as FILTER Predicate
    participant Block as BLOCK Predicate

    User->>App: SELECT / UPDATE / DELETE
    App->>DB: Execute operation
    DB->>Filter: Check existing TenantId
    Filter-->>DB: Allow only TenantId = 42
    DB-->>User: Only own rows accessible

    User->>App: INSERT TenantId = 99
    App->>DB: INSERT
    DB->>Block: AFTER INSERT check
    Block-->>DB: 99 != 42 → Reject
    DB-->>User: Insert fails

    User->>App: UPDATE TenantId 42 → 99
    App->>DB: UPDATE
    DB->>Filter: Can user access existing row?
    Filter-->>DB: 42 == 42 → Yes

    DB->>Block: BEFORE UPDATE: check OLD TenantId
    Block-->>DB: 42 == 42 → Allow

    DB->>DB: Change TenantId to 99
    DB-->>User: Update succeeds

    Note over User,DB: ❌ BEFORE UPDATE validates the old value<br/>not the resulting TenantId
```


---
