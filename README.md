# DP-800 question repertory

Exam-prep question deck for **Exam DP-800: Developing AI-Enabled Database Solutions** (certification: Microsoft Certified: SQL AI Developer Associate).

- Official study guide: <https://learn.microsoft.com/en-us/credentials/certifications/resources/study-guides/dp-800>
- Skills measured as of March 12, 2026.

Each question is self-contained (own database, schema, and tables), deterministic (exactly one defensible answer or exact reproducible output), and ends with a full worked explanation. One question per epigraph of the skills outline, plus the original seed questions.

## Coverage map

### Design and develop database solutions (35–40%)

| Epigraph | Question file(s) | Topic |
|---|---|---|
| Design and implement database objects | [partitioning_1.md](partitioning_1.md) | Partition function/scheme, `RANGE LEFT` boundaries, `$PARTITION`, `TRUNCATE ... WITH (PARTITIONS ...)` — db `GridSense` |
| Design and implement database objects | [sequences_1.md](sequences_1.md) | Shared `SEQUENCE` objects as column defaults, `FOR XML` output — db `SEQUENCE_EXERCISE` |
| Design and implement database objects | [temporal_tables_1.md](temporal_tables_1.md) | System-versioned temporal tables, history rows, `FOR SYSTEM_TIME` — db `HarborHotel` |
| Design and implement database objects | [temporary_tables_1.md](temporary_tables_1.md) | `#temp` vs `##global` vs table variables, nested-proc scope — db `PayrollHub` |
| Design and implement database objects | [indexes_1.md](indexes_1.md) | Nonclustered index design, key vs `INCLUDE`, covering seeks — db `PageTurner` |
| Design and implement database objects | [clustered_indexes_1.md](clustered_indexes_1.md) | Clustered index vs PK, uniqueifier, row locators, `sys.indexes` — db `AeroPark` |
| Design and implement database objects | [column_store_indexes_1.md](column_store_indexes_1.md) | CCI vs NCCI (HTAP), rowgroups, batch mode — db `MartMetrics` |
| Implement programmability objects | [triggers_1.md](triggers_1.md) | `AFTER` triggers fire per statement, `inserted`, `@@ROWCOUNT`, `UPDATE ... FROM` semantics — db `CityLibrary` |
| Implement programmability objects | [views_1.md](views_1.md) | View updatability, `WITH CHECK OPTION`, `SCHEMABINDING` — db `CampusReg` |
| Implement programmability objects | [indexed_views_1.md](indexed_views_1.md) | Indexed-view requirements, `COUNT_BIG`, `NOEXPAND` — db `FreshWholesale` |
| Write advanced T-SQL code | [window_functions_1.md](window_functions_1.md) | `RANK` vs `DENSE_RANK`/`ROW_NUMBER`, default `RANGE` frame with ties — db `RaceDay` |
| Write advanced T-SQL code | [bikeshop_1.md](bikeshop_1.md) | Outer joins, `ON` vs `WHERE` filtering, aggregation, `FOR JSON` — db `BikeShop` |
| Write advanced T-SQL code | [CTEs_1.md](CTEs_1.md) | Recursive CTEs, `MAXRECURSION`, duplicate paths — db `OrgAtlas` |
| Write advanced T-SQL code | [JSON_OBJECT_1.md](JSON_OBJECT_1.md) | `JSON_OBJECT`, `NULL ON NULL` default, escaping — db `HomeMesh` |
| Write advanced T-SQL code | [JSON_ARRAY_1.md](JSON_ARRAY_1.md) | `JSON_ARRAY`, `ABSENT ON NULL` default, index shifting — db `CineSlate` |
| Write advanced T-SQL code | [JSON_ARRAYAGG_1.md](JSON_ARRAYAGG_1.md) | `JSON_ARRAYAGG`, inner `ORDER BY`, null handling — db `BistroMenu` |
| Write advanced T-SQL code | [JSON_CONTAINS_1.md](JSON_CONTAINS_1.md) | `JSON_CONTAINS` containment semantics, `[*]`, type matching — db `TalentPool` |
| Write advanced T-SQL code | [OPENJSON_1.md](OPENJSON_1.md) | `OPENJSON` default schema, `WITH`, `AS JSON`, type codes — db `SkyWatch` |
| Write advanced T-SQL code | [JSON_VALUE_1.md](JSON_VALUE_1.md) | `JSON_VALUE` lax/strict, 4000-char limit, vs `JSON_QUERY` — db `PolicyVault` |
| Write advanced T-SQL code | [REGEXP_LIKE_1.md](REGEXP_LIKE_1.md) | `REGEXP_LIKE`, anchors, multiline flag — db `FleetReg` |
| Write advanced T-SQL code | [REGEXP_REPLACE_1.md](REGEXP_REPLACE_1.md) | `REGEXP_REPLACE`, capture groups, greedy vs lazy — db `NewsArchive` |
| Write advanced T-SQL code | [REGEXP_SUBSTR_1.md](REGEXP_SUBSTR_1.md) | `REGEXP_SUBSTR`, start/occurrence/group args — db `BioLab` |
| Write advanced T-SQL code | [REGEXP_INSTR_1.md](REGEXP_INSTR_1.md) | `REGEXP_INSTR`, positions, `return_option` — db `HostMon` |
| Write advanced T-SQL code | [REGEXP_COUNT_1.md](REGEXP_COUNT_1.md) | `REGEXP_COUNT`, non-overlapping and empty matches — db `BuzzBoard` |
| Write advanced T-SQL code | [REGEXP_MATCHES_1.md](REGEXP_MATCHES_1.md) | `REGEXP_MATCHES` TVF, capture columns, `CROSS APPLY` — db `ChemStock` |
| Write advanced T-SQL code | [REGEXP_SPLIT_TO_TABLE_1.md](REGEXP_SPLIT_TO_TABLE_1.md) | `REGEXP_SPLIT_TO_TABLE`, ordinals, vs `STRING_SPLIT` — db `LensFolio` |
| Write advanced T-SQL code | [EDIT_DISTANCE_1.md](EDIT_DISTANCE_1.md) | `EDIT_DISTANCE` (Levenshtein), collation independence — db `KinRecords` |
| Write advanced T-SQL code | [EDIT_DISTANCE_SIMILARITY_1.md](EDIT_DISTANCE_SIMILARITY_1.md) | `EDIT_DISTANCE_SIMILARITY` score formula, dedup thresholds — db `ToolBarn` |
| Write advanced T-SQL code | [JARO_WINKLER_DISTANCE_1.md](JARO_WINKLER_DISTANCE_1.md) | `JARO_WINKLER_DISTANCE`, prefix bonus, distance vs similarity — db `BankGuard` |
| Write advanced T-SQL code | [graph_query_MATCH_1.md](graph_query_MATCH_1.md) | SQL Graph node/edge tables, `MATCH` arrow direction — db `ConfGraph` |
| Write advanced T-SQL code | [correlated_queries_1.md](correlated_queries_1.md) | Correlated subqueries, `NOT IN` vs `NOT EXISTS` NULL trap — db `VineYield` |
| Write advanced T-SQL code | [error_handling_1.md](error_handling_1.md) | `TRY...CATCH`, `THROW` vs `RAISERROR`, `XACT_ABORT`, `XACT_STATE()` — db `ParcelFlow` |
| Design and implement SQL solutions by using AI-assisted tools | [copilot_mcp_1.md](copilot_mcp_1.md) | `.vscode/mcp.json`, Copilot instruction files, tool approval, least privilege — scenario Meridian Analytics |

### Secure, optimize, and deploy database solutions (35–40%)

| Epigraph | Question file(s) | Topic |
|---|---|---|
| Implement data security and compliance | [dynamic_data_masking_1.md](dynamic_data_masking_1.md) | DDM mask functions, granular `UNMASK`, inference leak — db `ClinicCare` |
| Implement data security and compliance | [row_level_security_1.md](row_level_security_1.md) | RLS `FILTER` vs `BLOCK` predicates, `AFTER UPDATE` — db with `Sales.Orders` |
| Implement data security and compliance | [row_level_security_2.md](row_level_security_2.md) | RLS + `SESSION_CONTEXT` + database-principal checks — db with `Sales.OrderData` |
| Optimize database performance | [isolation_levels_1.md](isolation_levels_1.md) | `SNAPSHOT` isolation update conflict (Msg 3960), Azure SQL defaults — db `Ticketing` |
| Implement CI/CD by using SQL Database Projects | [sql_database_projects_1.md](sql_database_projects_1.md) | Schema drift, `SqlPackage /Action:DeployReport` vs `DriftReport`, SDK-style projects — project `StoreCatalog` |
| Integrate SQL solutions with Azure services | [data_api_builder_1.md](data_api_builder_1.md) | `dab-config.json` entities: views, stored procedures, REST/GraphQL, permissions, caching — db `MuseumHub` |

### Implement AI capabilities in database solutions (25–30%)

| Epigraph | Question file(s) | Topic |
|---|---|---|
| Design and implement models and embeddings | [embeddings_1.md](embeddings_1.md) | Embedding-maintenance architecture: triggers vs Change Tracking vs CDC vs scheduled — db `RecipeBox` |
| Design and implement intelligent search | [vector_search_1.md](vector_search_1.md) | `VECTOR_DISTANCE` cosine vs euclidean, ENN vs ANN (`VECTOR_SEARCH`) — db `TuneFinder` |
| Design and implement retrieval-augmented generation (RAG) | [rag_1.md](rag_1.md) | `sp_invoke_external_rest_endpoint`, JSON payload construction, `JSON_VALUE` extraction — db `TripDesk` |

## Conventions

- Every question uses a distinct scenario, database, schema, and table set — no reuse across questions.
- Multiple-choice questions have exactly one correct option and at least one deliberately subtle distractor; the explanation refutes every wrong option individually.
- "Write the query" questions include the full DDL/data session and the exact expected result; the explanation recomputes the result step by step and lists equivalent accepted answers.
- `*.txt` files are plain-text versions of the corresponding `*.md` question (legacy format; the MD file is canonical).
- Every pure-T-SQL question was executed and verified against a live SQL Server 2025 RTM (17.0.1000.7) instance; expected outputs, error numbers, and messages are the engine's actual output. Questions using SQL Server 2025 features state their prerequisites (`COMPATIBILITY_LEVEL 170`, and `PREVIEW_FEATURES = ON` for the fuzzy string functions) in their scripts.
