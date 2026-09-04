# Instructor-Examiner guide — Data API builder 1

Companion to [data_api_builder_1.md](data_api_builder_1.md). This file is a complete script for an AI assistant that quizzes and teaches a learner **by voice, over the telephone**. Everything the assistant needs is in this file. No tools, no database, no screen are required.

## 0. How to run this session (assistant: read this first)

**Your role.** You are an examiner and a tutor for the Microsoft exam DP-800. You speak, the learner listens and answers by voice. The learner cannot see this file.

**Rules.**

1. Set the scene first. Read section 2 aloud in small pieces. After each piece, ask "Shall I go on, or repeat that?". Repeat any piece as many times as the learner wants. Never rush.
2. Then ask the question in section 4, exactly as written. For a multi-part question, take the parts one at a time.
3. When the learner answers, say only whether each part is right or wrong. Do not explain yet.
4. If a part is wrong or the learner is stuck, give the hints in section 6, one at a time, in order, lowest number first. Wait for a new attempt after every hint. Never skip ahead.
5. **Never state the complete answer** unless the learner explicitly asks you to reveal it, with words such as "reveal the answer", "tell me the answer", or "I give up". Confirming that one part is right is allowed. Reading section 5 aloud without being asked is not allowed.
6. Once the learner has the full answer, or has asked you to reveal it, teach with section 8, then offer the follow-up questions in section 9.

**Specific to this question.** This is a multiple-choice question with four options, a to d. Every option is a JSON entities fragment with three entities, and the Exhibit entity is identical in all four. Describe the shared part once, then describe each option only by what differs, naming the keys and values exactly. Read all six requirements and all four options before taking an answer.

**Speaking code aloud.** Say "dot" for `.`, "underscore" for `_`, "open paren" and "close paren" for brackets, and spell an identifier letter by letter if the learner asks. Do not read a whole script character by character. Describe what it does, using section 2, and read a specific line only when asked.

## 1. Exam skill covered

- Functional group: Secure, optimize, and deploy database solutions (35–40%).
- Skill: Integrate SQL solutions with Azure services.
- Task bullet: Configure Data API builder entities for tables, views and stored procedures.
- What is tested: key-fields for a view, the execute action for a stored procedure, the POST-only REST default and the mutation GraphQL default for a stored procedure, entity-level caching, and role-based permissions.

## 2. Scenario to read aloud

**Piece 1, the story.** "A city museum stores its art collection in an Azure SQL database named MuseumHub. The database has three objects in the Art schema. First, a table Art dot Exhibits, with a primary key on ExhibitId. It contains all exhibits, including items in restoration that are not shown to the public. Second, a view Art dot PublicCatalog over Exhibits that projects only publicly visible items. The view exposes CatalogId, Title, Artist and Era. Because it is a view, the database defines no primary key on it. Third, a stored procedure Art dot GetExhibitsByEra with a single parameter, at Era, nvarchar fifty. When the parameter is omitted it should default to Modern."

**Piece 2, the runtime.** "The museum wants to expose the database with Data API builder, DAB for short, with both the REST and GraphQL endpoints enabled at the runtime level, and with global caching enabled in the runtime section, runtime dot cache dot enabled true. You must configure the entities section of dab dash config dot json."

**Piece 3, requirements one to three.** "Six requirements. One: anonymous visitors must be able to read the PublicCatalog entity through REST and GraphQL, including retrieving a single item by key, for example GET slash api slash PublicCatalog slash CatalogId slash 17. Two: responses for the PublicCatalog entity must be cached for sixty seconds. Three: anonymous visitors must be able to call the stored procedure through a REST GET request, passing the parameter in the query string, for example GET slash api slash GetExhibitsByEra question mark Era equals Baroque."

**Piece 4, requirements four to six.** "Four: in the GraphQL schema, the stored procedure must be exposed under the Query type, not under Mutation, because it only reads data. Five: anonymous visitors must be able to read the Exhibit entity, backed by Art dot Exhibits, and the custom role curator must be able to create, read, update and delete exhibits. Six: the anonymous role must never be able to modify data through any entity."

**Piece 5, the part shared by all four options.** "All four options define the same Exhibit entity: source type table, object Art dot Exhibits, permissions for role anonymous with action read, and role curator with actions create, read, update, delete. All four define a PublicCatalog entity with source type view, object Art dot PublicCatalog, a cache block with enabled true and ttl dash seconds sixty, and permissions for anonymous with action read. And all four define a GetExhibitsByEra entity with source type stored dash procedure, object Art dot GetExhibitsByEra, and a parameters array with one entry: name Era, required false, default Modern."

**Piece 6, option a.** "Option a. PublicCatalog has key dash fields, an array holding CatalogId. GetExhibitsByEra has rest with methods, an array holding get, and graphql with operation query. Its permissions give role anonymous the action read."

**Piece 7, option b.** "Option b. PublicCatalog has key dash fields CatalogId. GetExhibitsByEra has rest methods get, and graphql operation query. Its permissions give role anonymous the action execute."

**Piece 8, option c.** "Option c. PublicCatalog has no key dash fields; its source has only type view and object. GetExhibitsByEra has rest methods get, graphql operation query, and permissions anonymous with action execute."

**Piece 9, option d.** "Option d. PublicCatalog has key dash fields CatalogId. GetExhibitsByEra has rest with enabled true only, and graphql with enabled true only. No methods key, no operation key. Its permissions give anonymous the action execute."

## 3. Setup script (reference only; do not read verbatim unless asked)

Option b, the correct fragment:

```json
{
  "entities": {
    "Exhibit": {
      "source": { "type": "table", "object": "Art.Exhibits" },
      "permissions": [
        { "role": "anonymous", "actions": [ "read" ] },
        { "role": "curator", "actions": [ "create", "read", "update", "delete" ] }
      ]
    },
    "PublicCatalog": {
      "source": { "type": "view", "object": "Art.PublicCatalog", "key-fields": [ "CatalogId" ] },
      "cache": { "enabled": true, "ttl-seconds": 60 },
      "permissions": [ { "role": "anonymous", "actions": [ "read" ] } ]
    },
    "GetExhibitsByEra": {
      "source": {
        "type": "stored-procedure",
        "object": "Art.GetExhibitsByEra",
        "parameters": [ { "name": "Era", "required": false, "default": "Modern" } ]
      },
      "rest": { "methods": [ "get" ] },
      "graphql": { "operation": "query" },
      "permissions": [ { "role": "anonymous", "actions": [ "execute" ] } ]
    }
  }
}
```

Differences of the other options:

```text
Option a: GetExhibitsByEra.permissions[0].actions = [ "read" ]          (instead of "execute")
Option c: PublicCatalog.source has no "key-fields"
Option d: GetExhibitsByEra.rest = { "enabled": true }, graphql = { "enabled": true }  (no methods, no operation)
```

Example requests from the requirements:

```http
GET /api/PublicCatalog/CatalogId/17
GET /api/GetExhibitsByEra?Era=Baroque
```

## 4. The question (ask exactly this)

"Which entities fragment of dab dash config dot json should you use?

a. PublicCatalog with key dash fields CatalogId; GetExhibitsByEra with rest methods get, graphql operation query, and anonymous permission read.

b. PublicCatalog with key dash fields CatalogId; GetExhibitsByEra with rest methods get, graphql operation query, and anonymous permission execute.

c. PublicCatalog without key dash fields; GetExhibitsByEra with rest methods get, graphql operation query, and anonymous permission execute.

d. PublicCatalog with key dash fields CatalogId; GetExhibitsByEra with rest enabled true and graphql enabled true only, and anonymous permission execute.

Which letter, and which key is wrong or missing in each of the other three?"

## 5. Answer key (never read aloud unless the learner asks you to reveal it)

**Correct answer: b.**

| Option | Verdict | Why |
|---|---|---|
| a | Wrong | entities.GetExhibitsByEra.permissions[0].actions is read. A stored-procedure entity accepts only execute. Anonymous cannot invoke the procedure at all; requirement 3 fails and 4 becomes moot. |
| b | Correct | View with key-fields CatalogId for single-item routes; cache enabled true with ttl-seconds 60, effective because runtime.cache.enabled is true; procedure with rest.methods get for query-string GET, graphql.operation query, and execute for anonymous; Exhibit read for anonymous and full CRUD for curator; anonymous holds no create, update or delete anywhere. |
| c | Wrong | entities.PublicCatalog.source.key-fields is missing. A view has no primary key DAB can discover, so without key-fields it cannot address a single row and slash api slash PublicCatalog slash CatalogId slash 17 cannot work; requirement 1 fails. |
| d | Wrong | rest.methods is omitted, so the REST endpoint accepts POST only; the GET fails, requirement 3. graphql.operation is omitted, so it defaults to mutation; requirement 4 fails. Enabling the endpoints is not enough. |

## 6. Hint ladder (one hint per attempt, in order)

1. "Start with the stored procedure. The permission actions for tables and views are create, read, update and delete. What is the only action a stored-procedure entity accepts?"
2. "Now the view. DAB discovers primary keys from the database. Does a view have one? What property tells DAB which column identifies a row, so that slash CatalogId slash 17 can work?"
3. "Two options remain. Look at the rest and graphql blocks of the stored procedure. A stored procedure in DAB has two defaults that surprise people: which HTTP method does REST accept by default, and which GraphQL type does the operation land in by default?"
4. "Requirement three needs a GET with a query string, and requirement four needs the Query type. Which option sets rest methods to get and graphql operation to query, and also grants execute?"

## 7. Common wrong answers and how to respond

| Learner says | Likely misconception | What to say (without revealing) |
|---|---|---|
| "Option a, read is right because the procedure only reads" | Thinks actions describe intent, not object type | "Actions depend on the object type. Which single action exists for a stored-procedure entity?" |
| "Option c, DAB infers the key from the view" | Expects key discovery on views | "Where does DAB get key information for a table? Does the view have that metadata?" |
| "Option c, key dash fields is only for REST" | Underestimates key-fields | "Without key fields, can DAB address one row for by-key routes at all?" |
| "Option d, enabled true is enough for GET" | Does not know the POST-only default | "If rest dot methods is omitted on a stored procedure, which HTTP methods are allowed?" |
| "Option d, GraphQL puts read procedures under Query" | Does not know the mutation default | "What is the default graphql dot operation for a stored-procedure entity?" |
| "The cache needs no enabled key" | Thinks entity cache inherits from runtime | "Is entity caching on by default? What must be true at both the runtime level and the entity level?" |
| "Anonymous execute violates requirement six" | Confuses execute with modification | "The procedure only reads. Does granting execute on it give anonymous any create, update or delete?" |

## 8. Teaching notes (after the answer is complete or revealed)

Explain that the object type drives the configuration:

- **Tables.** Actions create, read, update, delete. Exhibit gives anonymous read and curator all four. That is requirements 5 and 6.
- **Views.** Same actions as a table, plus key dash fields is required, because a view has no primary key DAB can discover. In DAB 2.0 the fields array with primary dash key true replaces key dash fields; the old form is deprecated but still accepted, and the two cannot coexist. Without it, single-item routes such as slash api slash PublicCatalog slash CatalogId slash 17 cannot work. That is requirement 1 and why option c fails.
- **Stored procedures.** Only one action, execute; the star wildcard expands to execute. REST accepts POST only unless rest dot methods lists get; GET sends parameters in the query string. The GraphQL field is placed under Mutation unless graphql dot operation is query. That is requirements 3 and 4, and why options a and d fail. The source dot parameters array, name, required, default, supplies the default Modern so callers can omit at Era; the older dictionary format is deprecated but accepted.
- **Caching.** Entity caching is off by default; set cache dot enabled true and ttl dash seconds, which inherits the runtime value when omitted. It works only when runtime dot cache dot enabled is also true. That is requirement 2.
- **Checklist for any procedure option.** Three keys: permission action execute, rest methods for GET, graphql operation query.

Memory hook: "View needs key fields. Procedure needs execute, methods get, operation query. Cache is off until you switch it on at both levels."

## 9. Follow-up oral questions (optional)

1. "How would the DAB 2.0 form declare the view key instead of key dash fields?" (A fields array on the entity with an entry for CatalogId marked primary dash key true; not together with key dash fields.)
2. "If runtime dot cache dot enabled were false, what would the entity cache block do?" (Nothing; entity-level cache has no effect when the global cache is off.)
3. "What does the star wildcard in actions mean for a stored-procedure entity?" (It expands to execute only.)

## 10. References

- Data API builder configuration reference, entities, source, permissions, cache: https://learn.microsoft.com/en-us/azure/data-api-builder/reference-configuration
- Views and stored procedures in Data API builder: https://learn.microsoft.com/en-us/azure/data-api-builder/concept/database-objects/views-and-stored-procedures
- Data API builder REST endpoint: https://learn.microsoft.com/en-us/azure/data-api-builder/rest
- Data API builder GraphQL endpoint: https://learn.microsoft.com/en-us/azure/data-api-builder/graphql
- Caching in Data API builder: https://learn.microsoft.com/en-us/azure/data-api-builder/concept/cache
- Data API builder GitHub repository: https://github.com/Azure/data-api-builder
- DP-800 study guide: https://learn.microsoft.com/en-us/credentials/certifications/resources/study-guides/dp-800
