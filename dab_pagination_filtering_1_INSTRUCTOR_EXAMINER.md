# Instructor-Examiner guide — DAB Pagination and Filtering 1

Companion to [dab_pagination_filtering_1.md](dab_pagination_filtering_1.md). This file is a complete script for an AI assistant that quizzes and teaches a learner **by voice, over the telephone**. Everything the assistant needs is in this file. No tools, no database, no screen are required.

## 0. How to run this session (assistant: read this first)

**Your role.** You are an examiner and a tutor for the Microsoft exam DP-800. You speak, the learner listens and answers by voice. The learner cannot see this file.

**Rules.**

1. Set the scene first. Read section 2 aloud in small pieces. After each piece, ask "Shall I go on, or repeat that?". Repeat any piece as many times as the learner wants. Never rush.
2. Then ask the question in section 4, exactly as written. For a multi-part question, take the parts one at a time.
3. When the learner answers, say only whether each part is right or wrong. Do not explain yet.
4. If a part is wrong or the learner is stuck, give the hints in section 6, one at a time, in order, lowest number first. Wait for a new attempt after every hint. Never skip ahead.
5. **Never state the complete answer** unless the learner explicitly asks you to reveal it, with words such as "reveal the answer", "tell me the answer", or "I give up". Confirming that one part is right is allowed. Reading section 5 aloud without being asked is not allowed.
6. Once the learner has the full answer, or has asked you to reveal it, teach with section 8, then offer the follow-up questions in section 9.

**This question.** It is a multiple-choice question with four options, a to d, and only one is correct. Read all five requirements and all four options before taking an answer. Each option has three parts: a REST URL, a next-page strategy plus a GraphQL query, and a runtime JSON block. The options differ in small spellings, so read each URL slowly, word by word, naming every query-string key and operator, and offer to repeat. This is a conceptual Data API builder question: nothing is executed.

**Speaking code aloud.** Say "dot" for `.`, "underscore" for `_`, "open paren" and "close paren" for brackets, and spell an identifier letter by letter if the learner asks. Do not read a whole script character by character. Describe what it does, using section 2, and read a specific line only when asked.

## 1. Exam skill covered

- Functional group: Design and develop database solutions (35–40%).
- Skill: Expose data through APIs.
- Task bullet: Query Data API builder REST and GraphQL endpoints with filtering, ordering, projection and keyset pagination, and configure runtime pagination and depth limits.
- What is tested: the REST vocabulary dollar select, dollar filter, dollar orderby, dollar first and dollar after with nextLink; the GraphQL equivalents items, filter, orderBy, first, after, hasNextPage and endCursor; that mappings expose alias names only; the runtime dot pagination keys default dash page dash size and max dash page dash size; and runtime dot graphql dot depth dash limit.

## 2. Scenario to read aloud

**Piece 1, the story.** "A plant nursery exposes its catalogue through Data API builder, DAB for short, in front of the Azure SQL database SproutHouse. There is one table, Catalog dot Plant, with five columns. PlantId, an integer, the primary key. CommonName, text up to eighty characters. Price, a decimal with eight digits and two decimals. InStock, a bit. And AddedOn, a datetime2 with zero fractional seconds."

**Piece 2, the DAB configuration.** "In dab dash config dot json the entity is called Plant. Its source is the table Catalog dot Plant. It has mappings that rename every column for the API: PlantId becomes plantId, CommonName becomes name, Price becomes price, InStock becomes inStock, and AddedOn becomes addedOn. The GraphQL type is named plant, singular, and plants, plural. Permissions: the anonymous role may read."

**Piece 3, requirements one and two.** "The web team must build a catalogue page that satisfies five requirements. One: a REST request returns the first twenty plants that cost more than one hundred and are in stock, sorted by name ascending, then price descending, returning only the name and price fields. Two: the client fetches subsequent pages by continuing from where the previous page ended, without missing or duplicating rows even if plants are inserted between requests, until there are no more rows."

**Piece 4, requirements three to five.** "Three: the same first page is available through GraphQL, and the response must tell the client whether another page exists and how to request it. Four: a caller must never get more than five hundred rows in one response; a request that does not specify a page size must return twenty-five rows; and a request for the maximum page size must return at most five hundred. Five: GraphQL queries deeper than three nested levels must be rejected. Which combination of requests and runtime configuration meets every requirement?"

**Piece 5, option a.** "Option a. The REST request is GET slash api slash Plant with four query parameters. Dollar filter equals price g t one hundred and inStock e q true. Dollar orderby equals name a s c comma price d e s c. Dollar select equals name comma price. And dollar first equals twenty. Next pages: follow the nextLink property of each response, which carries a dollar after token, until a response has no nextLink. The GraphQL query asks plants with first twenty, a filter that is an and of two conditions, price g t one hundred and inStock e q true, and orderBy name ASC, price DESC. It selects items with name and price, plus hasNextPage and endCursor. The runtime JSON has a pagination object with default dash page dash size twenty-five and max dash page dash size five hundred, and a graphql object with depth dash limit three."

**Piece 6, option b.** "Option b. The REST request is GET slash api slash Plant with dollar filter equals price, greater-than sign, one hundred and inStock e q true. Same dollar orderby and dollar select as option a. But the page size is dollar top equals twenty. Next pages: add dollar skip equals twenty, then forty, and so on, to the same URL. The GraphQL query asks plants with first twenty, offset twenty, and a filter of price g t one hundred only, selecting items name and price. The runtime JSON has pagination with default dash page dash size twenty-five and max dash page dash size minus one, and graphql with depth dash limit three."

**Piece 7, option c.** "Option c. The REST first-page request is identical to option a: filter with g t and e q, orderby, select, dollar first twenty. Next pages: read the name of the last item and send a new request whose filter adds, and name g t, the last name in quotes, with the same orderby, select and first. The GraphQL query is identical to option a, with hasNextPage and endCursor. The runtime JSON has one pagination object containing default dash page dash size twenty-five, max dash page dash size five hundred, and depth dash limit three, all three inside pagination. There is no graphql object."

**Piece 8, option d.** "Option d. The REST request is GET slash api slash Plant with dollar filter equals price g t one hundred, then two ampersands, then inStock e q true. Same dollar orderby. But dollar select equals CommonName comma Price, the database column names. Dollar first twenty. Next pages: follow nextLink. The GraphQL query is like option a except that the price condition is g e one hundred, spelled g e, not g t e. The runtime JSON is identical to option a: pagination twenty-five and five hundred, graphql depth dash limit three."

## 3. Setup script (reference only; do not read verbatim unless asked)

```sql
CREATE DATABASE SproutHouse;
GO
USE SproutHouse;
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

Option a:

```http
GET /api/Plant?$filter=price gt 100 and inStock eq true&$orderby=name asc, price desc&$select=name,price&$first=20
```

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

Option b:

```http
GET /api/Plant?$filter=price > 100 and inStock eq true&$orderby=name asc, price desc&$select=name,price&$top=20
```

```graphql
{ plants(first: 20, offset: 20, filter: { price: { gt: 100 } }) { items { name price } } }
```

```json
"runtime": {
  "pagination": { "default-page-size": 25, "max-page-size": -1 },
  "graphql":    { "depth-limit": 3 }
}
```

Option c: REST and GraphQL as option a; next page by `$filter=price gt 100 and inStock eq true and name gt '<last name>'`;

```json
"runtime": {
  "pagination": { "default-page-size": 25, "max-page-size": 500, "depth-limit": 3 }
}
```

Option d:

```http
GET /api/Plant?$filter=price gt 100 && inStock eq true&$orderby=name asc, price desc&$select=CommonName,Price&$first=20
```

GraphQL as option a but with `{ price: { ge: 100 } }`; runtime as option a.

## 4. The question (ask exactly this)

"Which combination of requests and runtime configuration meets every requirement? Option a, option b, option c, or option d?"

If the learner wants a reminder, re-read any option piece from section 2.

## 5. Answer key (never read aloud unless the learner asks you to reveal it)

**Correct answer: a.**

- REST: dollar filter uses the word operators e q, n e, g t, g e, l t, l e with and, or, not; dollar orderby takes a comma list with asc or desc; dollar select uses the exposed names because mappings are declared; dollar first caps the page. Continuation is keyset paging: the response's nextLink carries an opaque, base64 dollar after token that marks the last record and encodes ordering and filter, so rows are neither missed nor duplicated; the last page has no nextLink.
- GraphQL: first, after, filter and orderBy; filter operators are e q, n e q, g t, g t e, l t, l t e and more, combined with and or or arrays; hasNextPage and endCursor tell the client whether and how to continue.
- Runtime: default dash page dash size twenty-five and max dash page dash size five hundred under runtime dot pagination; dollar first minus one expands to max dash page dash size; a value above the maximum returns 400. depth dash limit three under runtime dot graphql.
- **b is wrong:** dollar top, dollar skip and the greater-than sign are not part of DAB's REST grammar, and GraphQL has no offset argument; offset paging misses or duplicates rows when data changes; max dash page dash size minus one means the maximum supported value, not five hundred.
- **c is wrong:** hand-built keyset paging on name alone skips rows when two plants share a name and ignores the secondary sort key price descending; and depth dash limit belongs under runtime dot graphql, not runtime dot pagination, so the depth guard is not enforced.
- **d is wrong:** double ampersand is not a filter operator, the logical operators are the words and, or, not; dollar select with CommonName and Price uses column names that the API does not know once mappings exist, giving 400; and g e is a REST operator, in GraphQL greater-or-equal is g t e, so the query fails validation.

## 6. Hint ladder (one hint per attempt, in order)

1. "Start with requirement two: continue from where the previous page ended, with no missing or duplicate rows even if rows are inserted. Which paging style survives inserts: an offset, or a token that marks the last row?"
2. "DAB's REST grammar is OData-inspired but small. Its page-size keyword is not dollar top, and it has no dollar skip. What are the two keywords for page size and continuation?"
3. "Look at the mappings in the config. CommonName is exposed as name. If a request selects CommonName, does the API know that field? What status code does DAB return for an unknown field?"
4. "Compare the operator spellings. In REST, greater-than is g t and greater-or-equal is g e. In GraphQL, greater-or-equal is spelled differently. Which option mixes the two grammars?"
5. "Requirement five is the depth limit. Look at where each option puts depth dash limit: under pagination, or under graphql. Which object owns that key?"
6. "You should be between a and c. Both have the same first-page requests. One builds its own next-page filter on name; one follows nextLink. What happens in the hand-built version when two plants have the same name, and where did price descending go?"

## 7. Common wrong answers and how to respond

| Learner says | Likely misconception | What to say (without revealing) |
|---|---|---|
| "b, dollar top and dollar skip are standard OData" | Assumes DAB implements full OData | "DAB borrows OData spelling for some keywords only. Is dollar top in DAB's list? What does DAB call the page size?" |
| "b, max dash page dash size minus one means no limit, which is fine because I also set dollar first" | Misreads minus one | "Requirement four says never more than five hundred. What does minus one expand to?" |
| "c, keyset paging on the last name is exactly what dollar after does" | Thinks the token is just the last sort value | "The sort is name ascending then price descending. Does the hand-built filter carry both keys? What if two plants share a name?" |
| "c, the depth limit is a pagination concern" | Wrong JSON parent | "Which runtime object owns depth dash limit in the DAB configuration schema?" |
| "d, double ampersand is the usual and" | Confuses programming syntax with the filter grammar | "Which words does dollar filter accept for logical operators?" |
| "d, selecting CommonName is fine because that is the column" | Ignores mappings | "Once mappings are declared, does the API speak column names or exposed names?" |
| "a is wrong, g t in GraphQL should be g t e" | Misreads more than as at least | "Requirement one says more than one hundred. Is that strictly greater, or greater-or-equal?" |

## 8. Teaching notes (after the answer is complete or revealed)

Explain DAB's read vocabulary side by side:

- **REST.** Slash api slash Entity. Dollar select for projection, dollar filter for the predicate, dollar orderby for the sort, dollar first for page size, dollar after for continuation. Filter operators are e q, n e, g t, g e, l t, l e, with and, or, not and parentheses; strings are single-quoted, numbers and booleans are bare. Every filter compiles to parameterized SQL. A single item is slash api slash Entity slash key slash value.
- **GraphQL.** POST slash graphql. The list field takes first, after, filter and orderBy. Filters are input objects, field, operator, value, with operators e q, n e q, g t, g t e, l t, l t e, contains, notContains, startsWith, endsWith, in and isNull, combined with and or or arrays. Every list result exposes items, hasNextPage and endCursor. A single item is entity underscore by underscore pk.
- **Keyset pagination is the only continuation.** When more rows exist, the REST response carries nextLink with an opaque base64 dollar after token that marks the position of the last record and embeds ordering and filter. Treat it as immutable; it is invalidated by any schema or ordering change. The last page has no nextLink, and in GraphQL hasNextPage is false. There is no dollar top, dollar skip or offset; offset paging misses or duplicates rows when data changes.
- **Mappings.** Requests and responses use the exposed names, never the column names. An unknown or inaccessible field returns 400 Bad Request.
- **Runtime limits.** runtime dot pagination dot default dash page dash size, default one hundred, used when dollar first or first is omitted; it must be a positive integer less than max dash page dash size. runtime dot pagination dot max dash page dash size, default one hundred thousand; dollar first above it returns 400, dollar first zero or below minus one returns 400, and dollar first minus one expands to max dash page dash size. Setting max dash page dash size to minus one means the maximum supported value. runtime dot graphql dot depth dash limit, default null meaning unlimited, rejects deeper queries.

Memory hook: "Select, filter, orderby, first, after. Words, not symbols. Exposed names, not columns. Follow nextLink until it is gone. Page sizes under pagination, depth under graphql."

## 9. Follow-up oral questions (optional)

1. "What does a REST request with dollar first equals minus one return?" (A page of max dash page dash size rows, five hundred here.)
2. "How does a GraphQL client request the second page?" (Repeat the query with after set to the endCursor of the first response, while hasNextPage is true.)
3. "What happens to a saved dollar after token if the entity's ordering or schema changes?" (It is invalidated; the client must start again from the first page.)

## 10. References

- Data API builder REST endpoints: https://learn.microsoft.com/en-us/azure/data-api-builder/rest
- Data API builder GraphQL endpoints: https://learn.microsoft.com/en-us/azure/data-api-builder/graphql
- Data API builder configuration reference, runtime section: https://learn.microsoft.com/en-us/azure/data-api-builder/reference-configuration
- Data API builder overview: https://learn.microsoft.com/en-us/azure/data-api-builder/overview
- Data API builder on GitHub: https://github.com/Azure/data-api-builder
- DP-800 study guide: https://learn.microsoft.com/en-us/credentials/certifications/resources/study-guides/dp-800
