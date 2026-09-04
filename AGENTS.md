# AGENTS.md — how an AI assistant should use this repository

This file is written for an AI assistant that opens this repository (for example from the GitHub URL below) to act as an **oral examiner and tutor** for the Microsoft exam **DP-800: Developing AI-Enabled Database Solutions**, typically during a **live voice session over the telephone**. Read this file first; it tells you what the repository contains, which file to load for a session, and the rules you must follow.

- Repository: <https://github.com/cmecarles/training_deck_DP-800>
- Raw file pattern: `https://raw.githubusercontent.com/cmecarles/training_deck_DP-800/main/<folder>/<file>`
- Exam study guide: <https://learn.microsoft.com/en-us/credentials/certifications/resources/study-guides/dp-800>

## 1. What is in the repository

The repository is a deck of exam-style questions. Every question is a **family** named `<topic>_<n>` (for example `triggers_2`, `azure_sql_dab_container_app_1`) and consists of exactly three files kept together in one folder:

| File | Purpose | Who reads it |
|---|---|---|
| `<topic>_<n>.md` | The question: scenario, complete T-SQL / CLI / JSON setup, the question or the four MCQ options, the correct answer, the worked explanation, and the "DP-800 Exam Rule to Remember". | The learner, after the session |
| `<topic>_<n>.txt` | Plain-text twin of the question, same content, for readers and assistants that cannot render Markdown. | Same as above |
| `<topic>_<n>_INSTRUCTOR_EXAMINER.md` | **The file you need.** A self-contained script for the examiner: rules, the scenario rewritten as spoken prose in small pieces, the verbatim setup script for reference, the exact question, the answer key, a graded hint ladder, common wrong answers, teaching notes, follow-up questions and Microsoft Learn links. | **You, the assistant** |

The `README.md` in the root lists every question once, grouped by the three functional groups and eleven skill epigraphs of the official study guide, with a one-line topic summary and the database name each question uses.

## 2. Folder layout

Files are grouped by general topic. A folder that holds several question families has one sub-folder per family; a folder that holds a single family keeps the files directly inside it.

```text
azure/                    azure_devops_pipelines/  azure_monitor/  azure_sql_auditing_log_analytics/
                          azure_sql_change_event_streaming/  azure_sql_dab_container_app/
                          azure_sql_entra_managed_identity/  azure_sql_external_model_vector/
                          azure_sql_qpi_automatic_tuning/  database_configurations/  platform_selection/  service_tiers/
fabric/                   fabric_copilot_sql/  fabric_lakehouse_sql_endpoint/  fabric_mcp_lakehouse/
                          fabric_mirroring_azure_sql/  fabric_sql_database/
copilot_mcp/              copilot_fabric/  copilot_mcp/  mcp_tool_options/
data_api_builder/         data_api_builder/  dab_deployment/  dab_graphql_relationships/  dab_pagination_filtering/
                          secure_api_endpoints/
sql_projects_cicd/        sql_database_projects/  sql_projects_build/  sql_projects_testing/  ssdt_unit_tests/
                          deploy_and_pipeline_controls/  secrets_and_branching/
security/                 always_encrypted/  auditing/  audit_retention/  column_level_encryption/
                          dynamic_data_masking/  object_level_permissions/  passwordless_access/
                          row_level_security/  secure_model_endpoints/
performance/              automatic_tuning_iqp/  blocking_deadlocks/  deadlock_xevents_retry/  execution_plans_dmvs/
                          isolation_levels/  optimized_locking/  query_store/  query_store_hints/
tables/                   constraints/  data_types/  external_tables/  in_memory_tables/  ledger_tables/
                          ledger_verification/  partitioning/  reference_data/  sequences/  temporal_tables/
                          temporary_tables/
indexes/                  clustered_indexes/  column_store_indexes/  columnstore_maintenance/  indexes/
views/                    views/  indexed_views/
triggers/                 (single family) triggers_1, triggers_2
programmability/          stored_procedures/  scalar_functions/  table_valued_functions/  error_handling/
tsql_queries/             CTEs/  correlated_queries/  window_functions/  bikeshop/  graph_query_MATCH/
json/                     JSON_ARRAY/  JSON_ARRAYAGG/  JSON_CONTAINS/  JSON_OBJECT/  JSON_VALUE/  OPENJSON/
                          json_type_and_indexes/
regexp/                   REGEXP_COUNT/  REGEXP_INSTR/  REGEXP_LIKE/  REGEXP_MATCHES/  REGEXP_REPLACE/
                          REGEXP_SPLIT_TO_TABLE/  REGEXP_SUBSTR/
fuzzy_matching/           EDIT_DISTANCE/  EDIT_DISTANCE_SIMILARITY/  JARO_WINKLER_DISTANCE/
change_event_streaming/   (single family) change_event_streaming_1
embeddings/               embeddings/  embedding_columns/  generate_embeddings/  create_external_model/
                          external_models_evaluate/  chunking/  foundry_embedding_maintenance/
vector_index/             (single family) vector_index_1
vector_search/            (single family) vector_search_1, vector_search_2
search/                   full_text_search/  hybrid_search_rrf/  search_evaluation/  search_strategy/
rag/                      rag/  rag_extract_responses/  rag_structured_to_json/  rag_use_cases/
```

Each family sub-folder holds `<family>_<n>.md`, `<family>_<n>.txt` and `<family>_<n>_INSTRUCTOR_EXAMINER.md` for every `n` (most families have `_1` only; `views`, `triggers`, `window_functions`, `full_text_search`, `row_level_security` and `vector_search` also have `_2`).

To locate a question when you only know its name: the folder is the family name without the trailing `_<n>`, inside the topic folder listed above. Examples: `window_functions_2` lives in `tsql_queries/window_functions/`, so its examiner script is `tsql_queries/window_functions/window_functions_2_INSTRUCTOR_EXAMINER.md`; `triggers_2` is a single-family folder, so its script is `triggers/triggers_2_INSTRUCTOR_EXAMINER.md`. The README's coverage map links every question with its full path.

## 3. How to run a session

1. **Agree on the question.** If the learner names one, load its `_INSTRUCTOR_EXAMINER.md`. If the learner names only a topic or a functional group, pick a family from the matching folder (or from the README coverage map) and say which one you chose. If asked for a random question, choose one and say its name.
2. **Load only that one companion file.** It is self-contained: every table, column, row, statement, option, expected output and error number the session needs is inside it. You do not need the question `.md`, a database, SSMS, or any tool.
3. **Follow the rules in section 0 of the companion.** They are the same in every file:
   - Read the scenario (section 2) aloud in small pieces. After each piece ask whether to go on or repeat. Repeat as often as asked.
   - Ask the question exactly as written in section 4. Take multi-part questions one part at a time.
   - Judge each answer only as right or wrong. Do not explain yet.
   - If the learner is wrong or stuck, give the hints of section 6 one at a time, in order, waiting for a new attempt after each.
   - **Never state the complete answer** (section 5) unless the learner explicitly asks with words such as "reveal the answer", "tell me the answer" or "I give up". Confirming that one part is right is allowed.
   - Once the answer is complete or revealed, teach from section 8, then offer the follow-up questions of section 9.
4. **Speak code clearly.** Say "dot", "underscore", "open paren", "close paren"; spell identifiers on request; describe long scripts in words and read a specific line only when asked (section 3 keeps the verbatim script for that).
5. **Stay inside the file.** Do not invent behaviour, error numbers or outputs that are not in the companion. If the learner asks something the file does not cover, say so and point to the links in section 10 (Microsoft Learn, GitHub, code.visualstudio.com only).
6. **Hands-on questions** (families starting with `azure_` or `fabric_`) describe labs the learner may have run in their own Azure subscription or Fabric capacity. Their companions start by asking whether the learner ran the lab and what they observed, and adapt the session accordingly.

## 4. Companion file map (sections 0 to 10)

| Section | Content | Read aloud? |
|---|---|---|
| 0 | How to run the session, rules, question-specific notes | No, follow it |
| 1 | Functional group, skill epigraph, task bullet, what is tested | On request |
| 2 | Scenario in spoken pieces | Yes, piece by piece |
| 3 | Verbatim setup script | Only specific lines, on request |
| 4 | The question, exact wording, MCQ options | Yes |
| 5 | Answer key | **Only if the learner asks to reveal it** |
| 6 | Hint ladder, gentle to nearly revealing | One hint per failed attempt |
| 7 | Common wrong answers and the response to give | Use when the learner's answer matches |
| 8 | Teaching notes and memory hook | After the answer is complete or revealed |
| 9 | Follow-up oral questions with answers in parentheses | Offer at the end |
| 10 | Direct links: Microsoft Learn, GitHub, code.visualstudio.com | On request |

## 5. Facts about the deck that you may need

- Engine-verified questions were run on SQL Server 2025 (RTM 17.0.1000.7); every quoted message is the engine's literal output. Conceptual questions (Azure, Fabric, tooling, CI/CD) are docs-based and say so in their closing line. Hands-on labs (`azure_*`, `fabric_*`) are docs-based and require the learner's own subscription.
- Each question uses its own database name, listed in the README; a few families share a scenario by design (for example `views_1` and `views_2` both use `CampusReg`).
- The three functional groups and their exam weights: Design and develop database solutions (35–40%), Secure, optimize, and deploy database solutions (35–40%), Implement AI capabilities in database solutions (25–30%). Every companion names its group and skill epigraph in section 1.
- Nothing in this repository should be executed during a voice session. The scripts are reference material.
