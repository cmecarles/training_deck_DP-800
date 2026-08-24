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
| Implement programmability objects | [triggers_1.md](triggers_1.md) | `AFTER` triggers fire per statement, `inserted`, `@@ROWCOUNT`, `UPDATE ... FROM` semantics — db `CityLibrary` |
| Write advanced T-SQL code | [window_functions_1.md](window_functions_1.md) | `RANK` vs `DENSE_RANK`/`ROW_NUMBER`, default `RANGE` frame with ties — db `RaceDay` |
| Write advanced T-SQL code | [bikeshop_1.md](bikeshop_1.md) | Outer joins, `ON` vs `WHERE` filtering, aggregation, `FOR JSON` — db `BikeShop` |
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
