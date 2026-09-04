# SQL Server question — Full-Text Search 1

## Statement

A legal-tech company stores contracts in a SQL Server 2025 database named `LegalStack` on an instance where the **Full-Text and Semantic Extractions for Search** feature is installed (`SELECT FULLTEXTSERVICEPROPERTY('IsFullTextInstalled')` returns 1).

```sql
CREATE DATABASE LegalStack;
GO
USE LegalStack;
GO
CREATE SCHEMA Legal;
GO
CREATE TABLE Legal.Contracts
(
    ContractId INT           NOT NULL CONSTRAINT PK_Contracts PRIMARY KEY,
    Title      NVARCHAR(200) NOT NULL,
    Body       NVARCHAR(MAX) NOT NULL,
    ClauseTags NVARCHAR(200) NULL
);
GO
CREATE NONCLUSTERED INDEX IX_Contracts_Title ON Legal.Contracts (Title, ContractId);
GO
```

Paralegals need a search with these requirements:

1. Find contracts whose `Body` contains any word that **starts with** `indemnif` (indemnify, indemnified, indemnification, …) **within 10 words** of the word `supplier`, in any order.
2. Exclude contracts whose `Body` contains the exact phrase `limited liability`.
3. Return the **20 best-ranked** contracts, highest relevance first, with the rank value visible.
4. The full-text index must be **populated immediately** and then **kept up to date automatically** as contracts are inserted or edited.
5. Use the system stoplist so that words such as *the* and *of* are neither indexed nor searched.

Which script (index definition plus query) meets all five requirements?

### a.

```sql
CREATE FULLTEXT CATALOG LegalCatalog AS DEFAULT;
GO
CREATE FULLTEXT INDEX ON Legal.Contracts (Title LANGUAGE 1033, Body LANGUAGE 1033)
    KEY INDEX PK_Contracts
    ON LegalCatalog
    WITH (CHANGE_TRACKING = AUTO, STOPLIST = SYSTEM);
GO
SELECT TOP (20) c.ContractId, c.Title, k.[RANK]
FROM Legal.Contracts AS c
INNER JOIN CONTAINSTABLE(Legal.Contracts, Body,
        'NEAR(("indemnif*", supplier), 10) AND NOT "limited liability"') AS k
    ON k.[KEY] = c.ContractId
ORDER BY k.[RANK] DESC;
```

### b.

```sql
CREATE FULLTEXT CATALOG LegalCatalog AS DEFAULT;
GO
CREATE FULLTEXT INDEX ON Legal.Contracts (Title LANGUAGE 1033, Body LANGUAGE 1033)
    KEY INDEX PK_Contracts
    ON LegalCatalog
    WITH (CHANGE_TRACKING = AUTO, STOPLIST = SYSTEM);
GO
SELECT TOP (20) ContractId, Title, [RANK]
FROM Legal.Contracts
WHERE CONTAINS(Body, 'NEAR(("indemnif*", supplier), 10) AND NOT "limited liability"')
ORDER BY [RANK] DESC;
```

### c.

```sql
CREATE FULLTEXT CATALOG LegalCatalog AS DEFAULT;
GO
CREATE FULLTEXT INDEX ON Legal.Contracts (Body LANGUAGE 1033)
    KEY INDEX IX_Contracts_Title
    ON LegalCatalog
    WITH (CHANGE_TRACKING = AUTO, STOPLIST = SYSTEM);
GO
SELECT TOP (20) c.ContractId, c.Title, k.[RANK]
FROM Legal.Contracts AS c
INNER JOIN FREETEXTTABLE(Legal.Contracts, Body,
        'indemnify supplier NOT "limited liability"') AS k
    ON k.[KEY] = c.ContractId
ORDER BY k.[RANK] DESC;
```

### d.

```sql
CREATE FULLTEXT CATALOG LegalCatalog AS DEFAULT;
GO
CREATE FULLTEXT INDEX ON Legal.Contracts (Title LANGUAGE 1033, Body LANGUAGE 1033)
    KEY INDEX PK_Contracts
    ON LegalCatalog
    WITH (CHANGE_TRACKING OFF, NO POPULATION, STOPLIST = SYSTEM);
GO
SELECT TOP (20) c.ContractId, c.Title, k.[RANK]
FROM Legal.Contracts AS c
INNER JOIN CONTAINSTABLE(Legal.Contracts, Body,
        'indemnif* NEAR supplier AND NOT "limited liability"') AS k
    ON k.[KEY] = c.ContractId
ORDER BY k.[RANK] DESC;
```

## Correct Answer

**a**

## Explanation

### Why option a is correct

- **Catalog and index.** A full-text index lives in a full-text catalog (`CREATE FULLTEXT CATALOG ... AS DEFAULT` makes the `ON` clause optional). `CREATE FULLTEXT INDEX` requires a `KEY INDEX` that is "a unique, single-key, non-nullable column" — the primary key `PK_Contracts` on the `INT` column qualifies, and the documentation recommends an integer key "for the best performance". Only one full-text index is allowed per table; listing `Title` and `Body` puts both columns in that one index.
- **Population.** `CHANGE_TRACKING = AUTO` is the default and gives exactly requirement 4: "When you enable change tracking during index creation, SQL Server fully populates the new full-text index immediately after it is created. Thereafter, changes are tracked and propagated to the full-text index." Propagation is asynchronous ("changes might not be reflected immediately in the full-text index"), so a loader that queries right after inserting must poll `OBJECTPROPERTYEX(OBJECT_ID('Legal.Contracts'), 'TableFullTextPopulateStatus')` (0 = idle) or `sys.dm_fts_index_population` before asserting results. (`FULLTEXTCATALOGPROPERTY(cat, 'PopulateStatus')` still works but is documented as scheduled for removal; the docs point to the table-level `OBJECTPROPERTYEX` property instead.)
- **Stoplist.** `STOPLIST = SYSTEM` associates the system stoplist: "The index isn't populated with any tokens that are part of the specified stoplist", and the same stoplist strips those words from queries.
- **The query.** `CONTAINSTABLE(table, column, '<condition>')` is a rowset function that returns one row per matching key with two columns, `KEY` and `RANK`; you join `KEY` to the table's unique key column and order by `RANK` (higher = more relevant; ties are possible). Inside the condition:
  - `"indemnif*"` is a *prefix term* — the asterisk must be inside double quotes: `CONTAINS (column, '"text*"')`; without the quotes "full-text search considers the asterisk as a character and searches for exact matches to `text*`".
  - `NEAR((term1, term2), 10)` is the *custom proximity term*: "specifies a positive integer ... This value controls how many non-search terms can occur between the first and last search terms"; the default match order is `FALSE` (any order), so `NEAR(("indemnif*", supplier), 10)` is requirement 1 exactly. Prefix terms are allowed inside `NEAR`.
  - `AND NOT "limited liability"` is the documented boolean form for exclusion ("NOT can only occur after AND, as in AND NOT"); a quoted multi-word simple term is an exact phrase match.

### Why option b is wrong

Index definition is fine; the query is not. `CONTAINS` is a **predicate** — "The `CONTAINS` and `FREETEXT` predicates return a `TRUE` or `FALSE` value" — it filters rows but produces **no rank**. The `RANK` column exists only in the rowsets returned by `CONTAINSTABLE` and `FREETEXTTABLE`, so `SELECT ... [RANK] ... ORDER BY [RANK]` refers to a column that `Legal.Contracts` does not have and the query fails to compile. Requirement 3 (ranked top 20) forces the table-valued function.

### Why option c is wrong

Two faults:

- `KEY INDEX IX_Contracts_Title` is a **composite, non-unique** index; the key index "must be a unique, single-key, non-nullable column". The statement is rejected, so no full-text index exists.
- Even with a valid key, `FREETEXTTABLE` is the wrong tool. `FREETEXT` / `FREETEXTTABLE` "match the meaning, but not the exact wording": they break the string into words, apply stemming and thesaurus expansion, and match **any** of the words. There is no prefix, no proximity, and — per the documentation — "`FREETEXT` and `FREETEXTTABLE` treat the Boolean terms as words to be searched": `NOT` becomes a search word, so contracts containing "limited liability" are *not* excluded (they may even rank higher because they match more words). Requirements 1 and 2 are lost.

### Why option d is wrong

This is the subtle distractor. `CHANGE_TRACKING OFF, NO POPULATION` is a legitimate combination — it is used to create a large index during peak hours and populate it later (`ALTER FULLTEXT INDEX ... START FULL POPULATION`) — but it violates requirement 4 twice: the index is not populated at creation ("no population of the index occurs until an ALTER FULLTEXT INDEX...START POPULATION statement is issued"), and with change tracking off later edits are never propagated. Until someone runs the population manually, every query returns nothing. The search condition has two further problems: `indemnif*` without double quotes is a literal search for the token `indemnif*` (the prefix wildcard is not recognised), and the bare `term NEAR term` form is the *generic* proximity term that has no distance limit and is documented as deprecated ("We recommend that you use `<custom_proximity_term>`") — it cannot express "within 10 words".

### A note on environments without full-text

The feature is an installable component of SQL Server (always present in Azure SQL Database and Managed Instance). On an instance where it is missing, `CREATE FULLTEXT CATALOG` succeeds but the index cannot be created — the engine returns `Msg 7609 Full-Text Search is not installed, or a full-text component cannot be loaded.` — and any `CONTAINS`/`FREETEXT` query against the table then fails with `Msg 7601 Cannot use a CONTAINS or FREETEXT predicate on table or indexed view 'Legal.Contracts' because it is not full-text indexed.` (both messages captured on the lab instance used for this deck, where `FULLTEXTSERVICEPROPERTY('IsFullTextInstalled')` returns 0). Always check the property first.

Conceptual question (the lab instance has no full-text component; behaviour is taken from the official documentation, and the two error messages above are the engine's literal output). Not executed against a full-text-enabled engine.

## DP-800 Exam Rule to Remember

```text
CREATE FULLTEXT CATALOG cat AS DEFAULT;
CREATE FULLTEXT INDEX ON tbl (col LANGUAGE 1033, ...)
    KEY INDEX <unique, single-column, NOT NULL index — int preferred>
    ON cat WITH (CHANGE_TRACKING = AUTO | MANUAL | OFF [, NO POPULATION], STOPLIST = SYSTEM | OFF | name);
    -- one full-text index per table; AUTO = full population now + automatic (async) propagation
```

Query side:

- `CONTAINS` / `FREETEXT` = predicates in `WHERE` (true/false, **no rank**); `CONTAINSTABLE` / `FREETEXTTABLE` = rowsets with `KEY` + `RANK` — join on `KEY`, `ORDER BY RANK DESC`, `TOP (n)` or the `top_n_by_rank` argument.
- `CONTAINS` grammar: `word`, `"phrase"`, `"prefix*"` (quotes required), `FORMSOF(INFLECTIONAL | THESAURUS, word)`, `NEAR((t1, t2), max_distance, match_order)`, `ISABOUT(... WEIGHT(0.8))`, and `AND` / `OR` / `AND NOT` (never a leading `NOT`).
- `FREETEXT` matches any word form of any term and treats `AND`/`NOT` as ordinary words — meaning-ish, never exact.
- Population is asynchronous: poll `OBJECTPROPERTYEX(OBJECT_ID('tbl'), 'TableFullTextPopulateStatus')` = 0 (or `sys.dm_fts_index_population`; the catalog-level `FULLTEXTCATALOGPROPERTY(cat, 'PopulateStatus')` is deprecated) before trusting fresh rows; `NO POPULATION` means empty until `ALTER FULLTEXT INDEX ... START FULL POPULATION`.
