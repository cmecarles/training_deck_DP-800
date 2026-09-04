# Instructor-Examiner guide — Full-Text Search 1

Companion to [full_text_search_1.md](full_text_search_1.md). This file is a complete script for an AI assistant that quizzes and teaches a learner **by voice, over the telephone**. Everything the assistant needs is in this file. No tools, no database, no screen are required.

## 0. How to run this session (assistant: read this first)

**Your role.** You are an examiner and a tutor for the Microsoft exam DP-800. You speak, the learner listens and answers by voice. The learner cannot see this file.

**Rules.**

1. Set the scene first. Read section 2 aloud in small pieces. After each piece, ask "Shall I go on, or repeat that?". Repeat any piece as many times as the learner wants. Never rush.
2. Then ask the question in section 4, exactly as written. For a multi-part question, take the parts one at a time.
3. When the learner answers, say only whether each part is right or wrong. Do not explain yet.
4. If a part is wrong or the learner is stuck, give the hints in section 6, one at a time, in order, lowest number first. Wait for a new attempt after every hint. Never skip ahead.
5. **Never state the complete answer** unless the learner explicitly asks you to reveal it, with words such as "reveal the answer", "tell me the answer", or "I give up". Confirming that one part is right is allowed. Reading section 5 aloud without being asked is not allowed.
6. Once the learner has the full answer, or has asked you to reveal it, teach with section 8, then offer the follow-up questions in section 9.

**Specific to this question.** This is a multiple-choice question with four options, a, b, c and d. Read the five requirements and all four options before taking an answer. Each option is an index definition plus a query; describe both halves in words, and read a specific line only on request. This is a conceptual question: the lab instance used for the deck has no full-text component, so the behaviour comes from the official documentation.

**Speaking code aloud.** Say "dot" for `.`, "underscore" for `_`, "open paren" and "close paren" for brackets, and spell an identifier letter by letter if the learner asks. Do not read a whole script character by character. Describe what it does, using section 2, and read a specific line only when asked.

## 1. Exam skill covered

- Functional group: Implement AI capabilities in database solutions (25–30%).
- Skill: Design and implement intelligent search.
- Task bullet: Implement full-text search.
- What is tested: how to define a full-text catalog and index (key index, change tracking, stoplist), and the difference between the CONTAINS and FREETEXT predicates and the CONTAINSTABLE and FREETEXTTABLE rowset functions, including prefix terms, custom proximity and AND NOT.

## 2. Scenario to read aloud

**Piece 1, the story.** "A legal-tech company stores contracts in a SQL Server 2025 database named LegalStack. The instance has the Full-Text and Semantic Extractions for Search feature installed. If you call FULLTEXTSERVICEPROPERTY with the argument IsFullTextInstalled, it returns one. Paralegals need a search over the contract text, and you must pick the script that meets five requirements."

**Piece 2, the table.** "There is one schema, Legal, with one table, Contracts. Four columns. ContractId, an integer, not null, primary key, and the primary key constraint is named PK underscore Contracts. Title, NVARCHAR two hundred, not null. Body, NVARCHAR MAX, not null. And ClauseTags, NVARCHAR two hundred, nullable. There is also a nonclustered index named IX underscore Contracts underscore Title on two columns, Title and ContractId. Note that this index is not unique and has two columns."

**Piece 3, the five requirements.** "Requirement one: find contracts whose Body contains any word that starts with i n d e m n i f, so indemnify, indemnified, indemnification and so on, within ten words of the word supplier, in any order. Requirement two: exclude contracts whose Body contains the exact phrase limited liability. Requirement three: return the twenty best ranked contracts, highest relevance first, with the rank value visible in the result. Requirement four: the full-text index must be populated immediately and then kept up to date automatically as contracts are inserted or edited. Requirement five: use the system stoplist, so that words such as the and of are neither indexed nor searched."

**Piece 4, option a.** "Option a. First it creates a full-text catalog named LegalCatalog AS DEFAULT. Then CREATE FULLTEXT INDEX ON Legal dot Contracts, on two columns, Title with LANGUAGE ten thirty three and Body with LANGUAGE ten thirty three. KEY INDEX PK underscore Contracts. ON LegalCatalog. WITH open paren CHANGE underscore TRACKING equals AUTO, STOPLIST equals SYSTEM close paren. The query selects TOP twenty, c dot ContractId, c dot Title and k dot RANK, from Legal dot Contracts aliased c, inner joined to CONTAINSTABLE. The CONTAINSTABLE call has three arguments: the table Legal dot Contracts, the column Body, and this search condition as a string: NEAR, open paren, open paren, double quote indemnif asterisk double quote, comma, supplier, close paren, comma, ten, close paren, AND NOT, double quote limited liability double quote. The join condition is k dot KEY equals c dot ContractId. Ordered by k dot RANK descending."

**Piece 5, option b.** "Option b. The catalog and the full-text index are identical to option a: Title and Body, key index PK underscore Contracts, change tracking AUTO, stoplist SYSTEM. The query is different. It selects TOP twenty, ContractId, Title and RANK, directly from Legal dot Contracts, with a WHERE clause that uses the CONTAINS predicate on Body with the same search condition as option a: NEAR of indemnif asterisk in quotes and supplier, distance ten, AND NOT limited liability in quotes. Ordered by RANK descending. There is no join and no CONTAINSTABLE."

**Piece 6, option c.** "Option c. Same catalog. The full-text index covers only the Body column, LANGUAGE ten thirty three. KEY INDEX is IX underscore Contracts underscore Title, the two-column nonclustered index. ON LegalCatalog, WITH CHANGE underscore TRACKING equals AUTO and STOPLIST equals SYSTEM. The query selects TOP twenty, c dot ContractId, c dot Title and k dot RANK, from Contracts joined to FREETEXTTABLE on Body, and the free text string is: indemnify supplier NOT, then limited liability in double quotes. Join on k dot KEY equals c dot ContractId, ordered by k dot RANK descending."

**Piece 7, option d.** "Option d. Same catalog. The index covers Title and Body, key index PK underscore Contracts, on LegalCatalog. But the WITH clause says CHANGE underscore TRACKING OFF, NO POPULATION, STOPLIST equals SYSTEM. The query selects TOP twenty, c dot ContractId, c dot Title and k dot RANK, from Contracts joined to CONTAINSTABLE on Body. The search condition is: indemnif asterisk with no quotes, then the word NEAR, then supplier, then AND NOT, double quote limited liability double quote. Join on k dot KEY equals c dot ContractId, ordered by k dot RANK descending."

## 3. Setup script (reference only; do not read verbatim unless asked)

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

Option a:

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

Option b:

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

Option c:

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

Option d:

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

## 4. The question (ask exactly this)

"Which script, index definition plus query, meets all five requirements? Option a, option b, option c, or option d?"

Options in full:

- **a.** Index on Title and Body, KEY INDEX PK_Contracts, CHANGE_TRACKING = AUTO, STOPLIST = SYSTEM. Query joins CONTAINSTABLE with condition `NEAR(("indemnif*", supplier), 10) AND NOT "limited liability"`, joins KEY to ContractId, TOP 20 ordered by RANK descending.
- **b.** Same index as a. Query uses the CONTAINS predicate in WHERE with the same condition, and selects and orders by a RANK column from Legal.Contracts.
- **c.** Index on Body only, KEY INDEX IX_Contracts_Title, CHANGE_TRACKING = AUTO, STOPLIST = SYSTEM. Query joins FREETEXTTABLE with the string `indemnify supplier NOT "limited liability"`, TOP 20 ordered by RANK descending.
- **d.** Index on Title and Body, KEY INDEX PK_Contracts, CHANGE_TRACKING OFF, NO POPULATION, STOPLIST = SYSTEM. Query joins CONTAINSTABLE with condition `indemnif* NEAR supplier AND NOT "limited liability"`, TOP 20 ordered by RANK descending.

## 5. Answer key (never read aloud unless the learner asks you to reveal it)

**Correct answer: a.**

- Index: a catalog AS DEFAULT, a full-text index on Title and Body, KEY INDEX on the unique single-column integer primary key PK_Contracts, CHANGE_TRACKING = AUTO (full population immediately, then automatic asynchronous propagation), STOPLIST = SYSTEM.
- Query: CONTAINSTABLE returns a rowset with KEY and RANK; join KEY to ContractId, TOP 20, ORDER BY RANK DESC. `"indemnif*"` in double quotes is a prefix term. `NEAR((t1, t2), 10)` is the custom proximity term with a maximum distance of ten and match order FALSE by default, so any order. `AND NOT "limited liability"` excludes the exact phrase.

Why the others are wrong, one line each:

- **b.** CONTAINS is a predicate that returns true or false and produces no RANK. The table has no RANK column, so the query fails to compile. Requirement three is not met.
- **c.** KEY INDEX IX_Contracts_Title is composite and non-unique, so CREATE FULLTEXT INDEX is rejected. And FREETEXTTABLE matches meaning, any word, with no prefix or proximity, and treats NOT as an ordinary search word. Requirements one and two are lost.
- **d.** CHANGE_TRACKING OFF with NO POPULATION means the index is empty until a manual START FULL POPULATION and edits are never propagated. Also, `indemnif*` without quotes is a literal search for the token indemnif asterisk, and the bare `term NEAR term` form is the deprecated generic proximity term with no distance limit.

## 6. Hint ladder (one hint per attempt, in order)

1. "Start with requirement three, the visible rank. Which full-text constructs actually return a RANK column, and which ones only return true or false?"
2. "Now look at requirement four, populated immediately and kept up to date. Which CHANGE underscore TRACKING setting does that, and which option turns it off?"
3. "Think about the KEY INDEX rule. The key index must be unique, single-column and not nullable. Does every option point at such an index?"
4. "Option b uses CONTAINS in a WHERE clause and then selects a column called RANK from the Contracts table. Does that table have a RANK column? That eliminates b."
5. "Option d creates the index with NO POPULATION and change tracking OFF. Would any query return rows before someone runs a manual population? That eliminates d."
6. "You are down to a and c. One uses CONTAINSTABLE with a quoted prefix term and a NEAR with a distance of ten. The other uses FREETEXTTABLE with a plain list of words, and points its KEY INDEX at a two-column non-unique index. Which one can express within ten words and exclude a phrase?"

## 7. Common wrong answers and how to respond

| Learner says | Likely misconception | What to say (without revealing) |
|---|---|---|
| "b, it is simpler and CONTAINS supports NEAR" | Believes CONTAINS exposes a rank | "CONTAINS is a predicate. What does a predicate return, and where would the RANK column come from?" |
| "c, FREETEXT handles word forms like indemnified" | Confuses stemming with prefix and proximity search | "FREETEXT matches any of the words, in any place. Can it say within ten words? And what does it do with the word NOT?" |
| "c is fine because the index only needs Body" | Ignores the KEY INDEX rule | "Look at the index chosen as KEY INDEX. How many columns does it have, and is it unique?" |
| "d, NO POPULATION is just a performance option" | Does not connect NO POPULATION to requirement four | "When would the index in option d first contain data? And what happens to a contract edited next week?" |
| "d, indemnif star without quotes still works as a prefix" | Does not know the asterisk must be inside double quotes | "How does CONTAINS treat an asterisk that is not inside double quotes?" |
| "a and b are the same index, so both work" | Stops at the index definition | "The index halves are identical. Compare the query halves, especially where RANK comes from." |

## 8. Teaching notes (after the answer is complete or revealed)

Explain the index side first:

- A full-text index lives in a full-text catalog. CREATE FULLTEXT CATALOG with AS DEFAULT makes the ON clause optional. One full-text index per table, so both Title and Body go in the same index.
- KEY INDEX must be a unique, single-key, non-nullable index. An integer key is recommended for performance. PK_Contracts qualifies. IX_Contracts_Title does not, because it is composite and not unique. That is option c's first fault.
- CHANGE_TRACKING = AUTO is the default. It fully populates the index immediately after creation and then tracks and propagates changes automatically. Propagation is asynchronous, so a loader that queries right after inserting should poll OBJECTPROPERTYEX with TableFullTextPopulateStatus equal to zero, or sys dot dm underscore fts underscore index underscore population. CHANGE_TRACKING OFF with NO POPULATION creates an empty index that stays empty until ALTER FULLTEXT INDEX START FULL POPULATION. That is option d's main fault.
- STOPLIST = SYSTEM keeps noise words such as the and of out of both the index and the queries.

Then the query side:

- CONTAINS and FREETEXT are predicates. They return true or false and have no rank. CONTAINSTABLE and FREETEXTTABLE are rowset functions that return KEY and RANK; you join KEY to the table's unique key and order by RANK descending. Requirement three forces the table-valued function. That is why b fails to compile.
- CONTAINS grammar: a word, a quoted phrase, a quoted prefix term such as "indemnif*" where the quotes are required, FORMSOF for inflectional or thesaurus forms, NEAR with a term list, a maximum distance and an optional match order, ISABOUT with WEIGHT, and AND, OR, AND NOT. NOT may only follow AND. Without quotes, indemnif* is searched as a literal token. The bare form term NEAR term is the deprecated generic proximity term with no distance limit. Those are option d's two query faults.
- FREETEXT and FREETEXTTABLE match meaning, not exact wording. They split the string into words, apply stemming and thesaurus expansion, match any word, and treat AND and NOT as ordinary words. So option c cannot express proximity and does not exclude limited liability.

Environment note: on an instance without the full-text component, CREATE FULLTEXT CATALOG succeeds but CREATE FULLTEXT INDEX fails with message 7609, and a CONTAINS or FREETEXT query then fails with message 7601. Check FULLTEXTSERVICEPROPERTY IsFullTextInstalled first.

Memory hook: "Predicates filter, TABLE functions rank. Quote the star. NEAR with a number. AUTO populates now and forever."

## 9. Follow-up oral questions (optional)

1. "If you needed the ten best matches without a TOP clause, which argument of CONTAINSTABLE could you use?" (The optional top_n_by_rank argument.)
2. "How would you check that automatic population has finished before trusting fresh rows?" (Poll OBJECTPROPERTYEX with TableFullTextPopulateStatus until it is zero, or query sys.dm_fts_index_population. FULLTEXTCATALOGPROPERTY PopulateStatus still works but is deprecated.)
3. "What error do you get if you run CONTAINS on a table that has no full-text index?" (Message 7601, cannot use a CONTAINS or FREETEXT predicate on a table that is not full-text indexed.)

## 10. References

- CREATE FULLTEXT INDEX: https://learn.microsoft.com/en-us/sql/t-sql/statements/create-fulltext-index-transact-sql
- CREATE FULLTEXT CATALOG: https://learn.microsoft.com/en-us/sql/t-sql/statements/create-fulltext-catalog-transact-sql
- CONTAINS, including prefix terms, NEAR and AND NOT: https://learn.microsoft.com/en-us/sql/t-sql/queries/contains-transact-sql
- CONTAINSTABLE: https://learn.microsoft.com/en-us/sql/relational-databases/system-functions/containstable-transact-sql
- FREETEXTTABLE: https://learn.microsoft.com/en-us/sql/relational-databases/system-functions/freetexttable-transact-sql
- Populate full-text indexes: https://learn.microsoft.com/en-us/sql/relational-databases/search/populate-full-text-indexes
- Query with full-text search: https://learn.microsoft.com/en-us/sql/relational-databases/search/query-with-full-text-search
- DP-800 study guide: https://learn.microsoft.com/en-us/credentials/certifications/resources/study-guides/dp-800
