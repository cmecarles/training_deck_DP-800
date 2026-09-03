# DP-800 question repertory

Exam-prep question deck for **Exam DP-800: Developing AI-Enabled Database Solutions** (certification: Microsoft Certified: SQL AI Developer Associate).

- Official study guide: <https://learn.microsoft.com/en-us/credentials/certifications/resources/study-guides/dp-800>
- Skills measured as of March 12, 2026.

Each question is self-contained (own database, schema, and tables), deterministic (exactly one defensible answer or exact reproducible output), and ends with a full worked explanation. At least one question per task-level bullet of the skills outline, plus the original seed questions.

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
| Design and implement database objects | [in_memory_tables_1.md](in_memory_tables_1.md) | Memory-optimized tables: `MEMORY_OPTIMIZED_DATA` filegroup, `SCHEMA_ONLY` vs `SCHEMA_AND_DATA`, hash `BUCKET_COUNT`, natively compiled `ATOMIC` procs, cross-container isolation — db `FlashCart` |
| Design and implement database objects | [ledger_tables_1.md](ledger_tables_1.md) | Updatable vs append-only ledger tables, hidden `GENERATED ALWAYS` columns, ledger view, blocked TRUNCATE, drop = rename — db `GrantLedger` |
| Design and implement database objects | [external_tables_1.md](external_tables_1.md) | Azure SQL elastic query `TYPE = RDBMS` external tables vs three-part names, PolyBase, `SHARD_MAP_MANAGER`, `OPENROWSET(BULK)` — dbs `OutletOrders`/`OutletCatalog` |
| Design and implement database objects | [json_type_and_indexes_1.md](json_type_and_indexes_1.md) | Native `json` type vs `nvarchar` + `ISJSON`, `CREATE JSON INDEX` rules, computed column + B-tree, `JSON_MODIFY` — db `GadgetSpecs` |
| Design and implement database objects | [constraints_1.md](constraints_1.md) | UNIQUE single NULL, CHECK passes UNKNOWN, FK NO ACTION/SET NULL/CASCADE, `WITH NOCHECK` + `is_not_trusted`, errors 515/547/2601/2627 — db `KennelBook` |
| Implement programmability objects | [scalar_functions_1.md](scalar_functions_1.md) | Scalar UDF inlining (`is_inlineable`, `INLINE = OFF`), `RETURNS NULL ON NULL INPUT`, determinism needs SCHEMABINDING, two-part name (195), side effects (443) — db `TollRoad` |
| Implement programmability objects | [table_valued_functions_1.md](table_valued_functions_1.md) | Inline vs multi-statement TVF, ORDER BY rule (1033), CROSS/OUTER APPLY vs JOIN (4104), interleaved execution estimates, updatability (270) — db `ShiftPlanner` |
| Implement programmability objects | [stored_procedures_1.md](stored_procedures_1.md) | OUTPUT at call site, defaults (201), RETURN NULL → 0 (282), `sp_` resolution to system proc, NOCOUNT, EXECUTE AS OWNER, WITH RECOMPILE, `INSERT ... EXEC` — db `WarrantyDesk` |
| Design and implement SQL solutions by using AI-assisted tools | [copilot_fabric_1.md](copilot_fabric_1.md) | Copilot in Fabric tenant/capacity settings, cross-geo processing, what Copilot for SQL database sends to Azure OpenAI, GitHub Copilot + MSSQL `@mssql` agent-mode tools, Fabric Data Warehouse MCP server for the lakehouse — scenario Aurora Retail |

### Secure, optimize, and deploy database solutions (35–40%)

| Epigraph | Question file(s) | Topic |
|---|---|---|
| Implement data security and compliance | [dynamic_data_masking_1.md](dynamic_data_masking_1.md) | DDM mask functions, granular `UNMASK`, inference leak — db `ClinicCare` |
| Implement data security and compliance | [row_level_security_1.md](row_level_security_1.md) | RLS `FILTER` vs `BLOCK` predicates, `AFTER UPDATE` — db with `Sales.Orders` |
| Implement data security and compliance | [row_level_security_2.md](row_level_security_2.md) | RLS + `SESSION_CONTEXT` + database-principal checks — db with `Sales.OrderData` |
| Optimize database performance | [isolation_levels_1.md](isolation_levels_1.md) | `SNAPSHOT` isolation update conflict (Msg 3960), Azure SQL defaults — db `Ticketing` |
| Implement CI/CD by using SQL Database Projects | [sql_database_projects_1.md](sql_database_projects_1.md) | Schema drift, `SqlPackage /Action:DeployReport` vs `DriftReport`, SDK-style projects — project `StoreCatalog` |
| Integrate SQL solutions with Azure services | [data_api_builder_1.md](data_api_builder_1.md) | `dab-config.json` entities: views, stored procedures, REST/GraphQL, permissions, caching — db `MuseumHub` |
| Implement data security and compliance | [always_encrypted_1.md](always_encrypted_1.md) | Always Encrypted: deterministic vs randomized, CMK/CEK, `Column Encryption Setting=Enabled`, secure enclaves (VBS/SGX), in-place encryption — db `LoanLedger` |
| Implement data security and compliance | [column_level_encryption_1.md](column_level_encryption_1.md) | DMK → certificate → symmetric key, `ENCRYPTBYKEY`/`DECRYPTBYKEY` NULL-on-failure, authenticator, `DECRYPTBYKEYAUTOCERT` — db `CourierVault` |
| Implement data security and compliance | [object_level_permissions_1.md](object_level_permissions_1.md) | GRANT/DENY/REVOKE precedence, column-GRANT-over-table-DENY quirk, ownership chaining and how dynamic SQL breaks it, `HAS_PERMS_BY_NAME`, Msg 229/230 — db `HarvestCoop` |
| Implement data security and compliance | [passwordless_access_1.md](passwordless_access_1.md) | Entra auth: `CREATE USER ... FROM EXTERNAL PROVIDER`, system- vs user-assigned managed identity, `Authentication=Active Directory Default/Managed Identity/Interactive`, Entra-only auth — db `TollGrid` |
| Implement data security and compliance | [auditing_1.md](auditing_1.md) | Server audit (FILE target, `QUEUE_DELAY`, `ON_FAILURE`), server vs database audit specification, `sys.fn_get_audit_file`, Azure SQL auditing destinations — db `DockYard` |
| Implement data security and compliance | [secure_model_endpoints_1.md](secure_model_endpoints_1.md) | `DATABASE SCOPED CREDENTIAL` with `IDENTITY = 'Managed Identity'` vs `HTTPEndpointHeaders`, credential-name/URL matching, Cognitive Services OpenAI User role, outbound firewall rules — db `PatentScout` |
| Implement data security and compliance | [secure_api_endpoints_1.md](secure_api_endpoints_1.md) | DAB auth providers (`EntraId`/`AppService`/`Simulator`), `anonymous`/`authenticated` roles, `X-MS-API-ROLE`, `@item`/`@claims` database policies, `runtime.mcp` switches — db `RiverRun` |
| Optimize database performance | [database_configurations_1.md](database_configurations_1.md) | `ALTER DATABASE SCOPED CONFIGURATION` (MAXDOP, LEGACY_CARDINALITY_ESTIMATION, OPTIMIZE_FOR_AD_HOC_WORKLOADS), RCSI vs ALLOW_SNAPSHOT_ISOLATION, compatibility level, automatic tuning — db `LedgerPulse` |
| Optimize database performance | [query_store_1.md](query_store_1.md) | Query Store catalog views, `sp_query_store_force_plan`/unforce, forced-plan failure `NO_INDEX`, capture modes, Query Performance Insight vs Query Store — db `TollGate` |
| Optimize database performance | [execution_plans_dmvs_1.md](execution_plans_dmvs_1.md) | Non-SARGable predicates, `PlanAffectingConvert` warning, seek vs scan + key lookup, `dm_exec_query_stats`/`sql_text`/`query_plan`, missing-index DMVs, `SHOWPLAN_XML` vs `STATISTICS XML` — db `CourierLane` |
| Optimize database performance | [blocking_deadlocks_1.md](blocking_deadlocks_1.md) | Real two-session deadlock (Msg 1205), `blocking_session_id`, `dm_os_waiting_tasks`, `dm_tran_locks`, `DEADLOCK_PRIORITY`, `system_health` xml_deadlock_report, lock escalation — db `WarehouseDock` |
| Implement CI/CD by using SQL Database Projects | [sql_projects_testing_1.md](sql_projects_testing_1.md) | tSQLt unit tests (FakeTable, AssertEqualsTable, rollback per test) vs integration tests on a SQL Server service container in GitHub Actions — project `PayoutEngine` |
| Implement CI/CD by using SQL Database Projects | [reference_data_1.md](reference_data_1.md) | Post-deployment script, `:r` includes, idempotent `MERGE` with `IDENTITY_INSERT`, why plain INSERT / pre-deploy TRUNCATE break re-deploys — db `LoanBook` |
| Implement CI/CD by using SQL Database Projects | [sql_projects_build_1.md](sql_projects_build_1.md) | SDK-style `.sqlproj`, `DSP` target platform, SQL71501 unresolved references, database references, `Extract` to SqlProject, `.gitignore` bin/obj — project `GrantTracker` |
| Implement CI/CD by using SQL Database Projects | [secrets_and_branching_1.md](secrets_and_branching_1.md) | OIDC federated credential + `azure/login`, secrets vs variables vs environment secrets, `azure/sql-action` inputs, feature branch/PR flow, resolving a conflict in a `.sql` object file — project `FareSplit` |
| Implement CI/CD by using SQL Database Projects | [deploy_and_pipeline_controls_1.md](deploy_and_pipeline_controls_1.md) | `Publish` vs `Script` vs `DeployReport`, `BlockOnPossibleDataLoss`/`DropObjectsNotInSource`, branch protection, `CODEOWNERS`, environments with required reviewers, workflow triggers — project `ParkPermit` |
| Integrate SQL solutions with Azure services | [dab_graphql_relationships_1.md](dab_graphql_relationships_1.md) | DAB `relationships`: `cardinality`, `source.fields`/`target.fields`, `linking.object` many-to-many, view `key-fields`, stored-procedure limits, GraphQL query shape — db `TrailGuide` |
| Integrate SQL solutions with Azure services | [dab_deployment_1.md](dab_deployment_1.md) | `dab init/add/validate/start`, `@env()` + `.env`, `runtime.host.mode`, `rest.path`/`graphql.path`, `allow-introspection`, CORS, `/health`, Container Apps image — db `RiverFerry` |
| Integrate SQL solutions with Azure services | [azure_monitor_1.md](azure_monitor_1.md) | Diagnostic-setting categories → Log Analytics vs storage vs Event Hubs, App Insights SQL dependencies, DAB `runtime.telemetry.application-insights`, metric vs log alerts, KQL — db `SkyFare` |
| Integrate SQL solutions with Azure services | [change_event_streaming_1.md](change_event_streaming_1.md) | CES vs CDC vs Change Tracking vs Functions SQL trigger vs Logic Apps: `sp_enable_event_stream`, `CHANGETABLE`, `cdc.<schema>_<table>_CT`, `__$operation`, Agent jobs — db `ClaimStream` |

### Implement AI capabilities in database solutions (25–30%)

| Epigraph | Question file(s) | Topic |
|---|---|---|
| Design and implement models and embeddings | [embeddings_1.md](embeddings_1.md) | Embedding-maintenance architecture: triggers vs Change Tracking vs CDC vs scheduled — db `RecipeBox` |
| Design and implement intelligent search | [vector_search_1.md](vector_search_1.md) | `VECTOR_DISTANCE` cosine vs euclidean, ENN vs ANN (`VECTOR_SEARCH`) — db `TuneFinder` |
| Design and implement retrieval-augmented generation (RAG) | [rag_1.md](rag_1.md) | `sp_invoke_external_rest_endpoint`, JSON payload construction, `JSON_VALUE` extraction — db `TripDesk` |
| Design and implement models and embeddings | [external_models_evaluate_1.md](external_models_evaluate_1.md) | Choosing embedding/chat models: ada-002 vs 3-small vs 3-large + `dimensions`, one model per column, 1998-dim cap, JSON mode vs structured outputs (`json_schema`), multimodal — db `GlobalBazaar` |
| Design and implement models and embeddings | [create_external_model_1.md](create_external_model_1.md) | `CREATE / ALTER ... SET / DROP EXTERNAL MODEL`, `sys.external_models`, `API_FORMAT`/`MODEL_TYPE` validation, `EXECUTE ON EXTERNAL MODEL` + `REFERENCES` on credential — db `ClaimsDesk` |
| Design and implement models and embeddings | [embedding_columns_1.md](embedding_columns_1.md) | Semantic columns vs metadata filters, labelled concatenation, per-chunk granularity, `HASHBYTES` content hash vs `ROWVERSION` — db `GearHub` |
| Design and implement models and embeddings | [chunking_1.md](chunking_1.md) | `AI_GENERATE_CHUNKS` FIXED chunking, overlap as % of chunk_size, output columns, NULL source under CROSS/OUTER APPLY, chunk table with FK + ordinal — db `TurbineDocs` |
| Design and implement models and embeddings | [generate_embeddings_1.md](generate_embeddings_1.md) | `CAST(json array AS vector(n))`, Msg 42204 mismatches, `OPENJSON ... AS JSON` batch parsing of `$.result.data`, `AI_GENERATE_EMBEDDINGS` NULL/disabled/unreachable errors — db `PatentVault` |
| Design and implement intelligent search | [search_strategy_1.md](search_strategy_1.md) | Full-text vs vector vs hybrid selection, recall@k vs exact kNN, latency and index build cost — db `PartsPro` |
| Design and implement intelligent search | [full_text_search_1.md](full_text_search_1.md) | Full-text catalog/index (`KEY INDEX` rules), `CONTAINSTABLE` KEY/RANK, `"prefix*"`, `NEAR`, `AND NOT`, `FREETEXT`, `CHANGE_TRACKING AUTO` — db `LegalStack` |
| Design and implement intelligent search | [vector_index_1.md](vector_index_1.md) | `vector(n)` storage (4n+8), 1998 cap, `VECTORPROPERTY`, `VECTOR_NORMALIZE`, cosine vs dot on unit vectors, `CREATE VECTOR INDEX` requirements and read-only table (42231), metric mismatch (42227) — db `ArtLens` |
| Design and implement intelligent search | [hybrid_search_rrf_1.md](hybrid_search_rrf_1.md) | RRF with k=60 via `ROW_NUMBER` + `FULL OUTER JOIN`, missing-list = 0, integer-division/LEFT JOIN traps, fused top-3 — db `NetHelp` |
| Design and implement retrieval-augmented generation (RAG) | [rag_use_cases_1.md](rag_use_cases_1.md) | RAG vs fine-tuning vs prompt-only vs deterministic SQL: freshness, private data, citations, cost — db `LumenLegal` |
| Design and implement retrieval-augmented generation (RAG) | [rag_structured_to_json_1.md](rag_structured_to_json_1.md) | `FOR JSON PATH`/`JSON_ARRAYAGG`/`STRING_AGG` → `JSON_OBJECT` payload, escaping traps (`JSON_QUERY`, `STRING_ESCAPE`), `sp_invoke_external_rest_endpoint` `@headers`/`@credential`/`@timeout` — db `LoanAdvisor` |
| Design and implement retrieval-augmented generation (RAG) | [rag_extract_responses_1.md](rag_extract_responses_1.md) | Envelope `$.result`/`$.response.status.http.code`, `JSON_VALUE` vs `JSON_QUERY` vs `OPENJSON WITH`, 4000-char limit (Msg 13625), structured output parsed twice — db `RepairPilot` |

## Conventions

- Every question uses a distinct scenario, database, schema, and table set — no reuse across questions.
- Multiple-choice questions have exactly one correct option and at least one deliberately subtle distractor; the explanation refutes every wrong option individually.
- "Write the query" questions include the full DDL/data session and the exact expected result; the explanation recomputes the result step by step and lists equivalent accepted answers.
- `*.txt` files are plain-text versions of the corresponding `*.md` question (legacy format; the MD file is canonical). Every MD question must have its TXT twin.
- Every pure-T-SQL question was executed and verified against a live SQL Server 2025 RTM (17.0.1000.7) instance. Each question's Explanation ends with a line saying whether it was engine-verified or is a conceptual (Azure / tooling, docs-based) question; expected outputs, error numbers, and messages are the engine's actual output. Questions using SQL Server 2025 features state their prerequisites (`COMPATIBILITY_LEVEL 170`, and `PREVIEW_FEATURES = ON` for the fuzzy string functions) in their scripts.

## Skills-outline coverage audit (2026-09-03)

Audit of the 73 task-level bullets in the official study guide (skills measured as of March 12, 2026) against the deck as of commit `9055909`. Status before this batch: **27 covered, 14 partial, 32 missing** (37% fully covered). The files listed in the "Added" column close each gap.

### Design and develop database solutions (35–40%)

| Bullet | Status before | Added |
|---|---|---|
| Tables, data types, columns, indexes, column store indexes | Covered | — |
| Specialized tables: in-memory, temporal, external, ledger, graph | Partial (temporal, graph) | in_memory_tables_1, ledger_tables_1, external_tables_1 |
| JSON columns and indexes | Missing | json_type_and_indexes_1 |
| Constraints: PK, FK, UNIQUE, CHECK, DEFAULT | Missing | constraints_1 |
| SEQUENCES | Covered | — |
| Partitioning for tables and indexes | Covered | — |
| Create views | Covered | — |
| Create scalar functions | Missing | scalar_functions_1 |
| Create table-valued functions | Missing | table_valued_functions_1 |
| Create stored procedures | Missing | stored_procedures_1 |
| Create triggers | Covered | — |
| CTEs | Covered | — |
| Window functions | Covered | — |
| JSON functions | Covered | — |
| Regular expressions | Covered | — |
| Fuzzy string matching | Covered | — |
| Graph MATCH | Covered | — |
| Correlated queries | Covered | — |
| Error handling | Covered | — |
| Security impact of AI-assisted tools | Partial | copilot_fabric_1 |
| Enable GitHub Copilot and Copilot in Fabric | Missing | copilot_fabric_1 |
| Model and MCP tool options in a chat session | Partial | (copilot_mcp_1) |
| GitHub Copilot instruction files | Covered | — |
| Connect to MCP endpoints: SQL Server, Fabric lakehouse | Partial (SQL Server only) | copilot_fabric_1 |

### Secure, optimize, and deploy database solutions (35–40%)

| Bullet | Status before | Added |
|---|---|---|
| Encryption: Always Encrypted, column-level encryption | Missing | always_encrypted_1, column_level_encryption_1 |
| Dynamic Data Masking | Covered | — |
| Row-Level Security | Covered | — |
| Object-level permissions | Missing | object_level_permissions_1 |
| Secure access incl. passwordless | Missing | passwordless_access_1 |
| Auditing | Missing | auditing_1 |
| Secure model endpoints incl. Managed Identity | Missing | secure_model_endpoints_1 |
| Secure GraphQL, REST, MCP endpoints | Missing | secure_api_endpoints_1 |
| Recommend database configurations | Missing | database_configurations_1 |
| Isolation levels and concurrency controls | Covered | — |
| Execution plans, DMVs, Query Store, Query Performance Insight | Missing | execution_plans_dmvs_1, query_store_1 |
| Blocking and deadlocks | Missing | blocking_deadlocks_1 |
| Testing strategy: unit and integration tests | Missing | sql_projects_testing_1 |
| Reference/static data in source control | Missing | reference_data_1 |
| Build and validate SDK-style database models | Partial | sql_projects_build_1 |
| Configure source control for SQL Database Projects | Missing | sql_projects_build_1 |
| Branching, pull requests, conflict resolution | Missing | secrets_and_branching_1 |
| Secrets management | Missing | secrets_and_branching_1 |
| Detect schema drift | Covered | — |
| Update a project and deploy changes | Partial | deploy_and_pipeline_controls_1 |
| Deployment pipeline controls: branch policies, approvals, code owners | Missing | deploy_and_pipeline_controls_1 |
| DAB configuration files | Covered | — |
| Entities for REST/GraphQL: caching, pagination, filtering | Covered | — |
| Configure REST or GraphQL endpoints | Partial | dab_deployment_1 |
| Expose objects, procedures, views incl. GraphQL relationships | Partial (no relationships) | dab_graphql_relationships_1 |
| DAB deployment | Missing | dab_deployment_1 |
| Azure Monitor: Application Insights, Log Analytics | Missing | azure_monitor_1 |
| CES, CDC, Change Tracking, Functions SQL trigger, Logic Apps | Partial (compared in embeddings_1) | change_event_streaming_1 |

### Implement AI capabilities in database solutions (25–30%)

| Bullet | Status before | Added |
|---|---|---|
| Evaluate external models: multimodal, multilanguage, sizes, structured output | Missing | external_models_evaluate_1 |
| Create and manage external models | Partial (used, not taught) | create_external_model_1 |
| Embedding maintenance method | Covered | — |
| Which columns to include in embeddings | Missing | embedding_columns_1 |
| Chunks for embeddings | Missing | chunking_1 |
| Generate embeddings | Partial | generate_embeddings_1 |
| Choose full-text, vector, or hybrid search | Missing | search_strategy_1 |
| Implement full-text search | Missing | full_text_search_1 |
| Vector data type, indexes, size | Partial | vector_index_1 |
| VECTOR_NORMALIZE, VECTOR_DISTANCE, VECTORPROPERTY, VECTOR_SEARCH | Partial | vector_index_1 |
| ANN vs ENN | Covered | — |
| Vector index types and metrics | Partial | vector_index_1 |
| Implement vector search | Covered | — |
| Hybrid search | Missing | hybrid_search_rrf_1 |
| Reciprocal rank fusion | Missing | hybrid_search_rrf_1 |
| Evaluate performance of vector and hybrid search | Missing | search_strategy_1 |
| Use cases for RAG | Missing | rag_use_cases_1 |
| Prompt via sp_invoke_external_rest_endpoint | Covered | — |
| Convert structured data to JSON | Partial | rag_structured_to_json_1 |
| Send results to language model | Covered | — |
| Extract language model responses | Partial | rag_extract_responses_1 |
