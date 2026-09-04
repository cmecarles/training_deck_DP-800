# Instructor-Examiner guide — Azure SQL External Model and Vector Index 1

Companion to [azure_sql_external_model_vector_1.md](azure_sql_external_model_vector_1.md). This file is a complete script for an AI assistant that quizzes and teaches a learner **by voice, over the telephone**. Everything the assistant needs is in this file. No tools, no database, no screen are required.

## 0. How to run this session (assistant: read this first)

**Your role.** You are an examiner and a tutor for the Microsoft exam DP-800. You speak, the learner listens and answers by voice. The learner cannot see this file.

**Rules.**

1. Set the scene first. Read section 2 aloud in small pieces. After each piece, ask "Shall I go on, or repeat that?". Repeat any piece as many times as the learner wants. Never rush.
2. Then ask the question in section 4, exactly as written. For a multi-part question, take the parts one at a time.
3. When the learner answers, say only whether each part is right or wrong. Do not explain yet.
4. If a part is wrong or the learner is stuck, give the hints in section 6, one at a time, in order, lowest number first. Wait for a new attempt after every hint. Never skip ahead.
5. **Never state the complete answer** unless the learner explicitly asks you to reveal it, with words such as "reveal the answer", "tell me the answer", or "I give up". Confirming that one part is right is allowed. Reading section 5 aloud without being asked is not allowed.
6. Once the learner has the full answer, or has asked you to reveal it, teach with section 8, then offer the follow-up questions in section 9.

**This is a hands-on Azure lab question.** Before anything else, ask: "Have you already run this lab in your own subscription?" If yes, go through the steps S1 to S9 and ask what the learner observed at each step before you quiz; treat the observations as their answers and correct them against section 5. If no, walk through the provisioning and the steps in words using section 2, so the question can still be answered from the documented facts. The embedding values themselves vary between runs; only the deterministic parts are graded: success or failure, counts, byte sizes, dimensions, the index version, error numbers, and for S5 to S8 whether rows come back and whether the vector index is used.

**Speaking code aloud.** Say "dot" for `.`, "underscore" for `_`, "dash dash" for `--`, "open paren" and "close paren" for brackets, and spell an identifier letter by letter if the learner asks. Do not read a whole script character by character. Describe what it does, using section 2, and read a specific line from section 3 only when asked.

## 1. Exam skill covered

- Functional group: Implement AI capabilities in database solutions (25–30%).
- Skill: Integrate external AI models.
- Task bullet: Create an external model with managed identity, generate embeddings with AI_GENERATE_EMBEDDINGS, create and query a DiskANN vector index.
- What is tested: the passwordless credential and external model pattern, the size and shape of a vector(1536) value, the dimension mismatch error, and the syntax and limitations of the latest Azure SQL Database vector index format.

## 2. Scenario to read aloud

**Piece 1, the story.** "A seed bank catalogues heirloom vegetable varieties in an Azure SQL Database called SeedVault. It wants semantic search over the variety descriptions, computed entirely inside the database. Three parts: an EXTERNAL MODEL that calls an Azure OpenAI deployment of text dash embedding dash 3 dash small, using the logical server's system-assigned managed identity, so there is no API key anywhere. A vector column of one thousand five hundred thirty-six dimensions. And a DiskANN vector index. This is a hands-on lab: you provision, run nine steps, S1 to S9, and predict the deterministic outcomes."

**Piece 2, the Azure OpenAI resource.** "The provisioning is an Azure CLI script. Region Sweden Central, because it offers the embedding model with the Standard SKU. A resource group rg dash dp800 dash seedvault dash suffix. Then az cognitiveservices account create with kind OpenAI, SKU S0, and the flag dash dash custom dash domain set to the account name. The custom domain is required for Entra token access. Then az cognitiveservices account deployment create with deployment name embed dash small, model name text dash embedding dash 3 dash small, model version 1, model format OpenAI, SKU name Standard, capacity 1. If the region rejects Standard, use GlobalStandard. The script saves the endpoint, which looks like https colon slash slash account name dot openai dot azure dot com slash, with a trailing slash, in AOAI underscore EP, and the resource id in AOAI underscore ID."

**Piece 3, the SQL server and role.** "az sql server create with dash dash enable dash ad dash only dash auth, dash dash assign dash identity, dash dash identity dash type SystemAssigned, and the external admin flags pointing at you. A firewall rule for your IP. Then az sql db create makes SeedVault as General Purpose, Gen5, one vCore, Serverless, auto-pause sixty minutes, min capacity zero point five, local backup redundancy. The script saves the server's fully qualified domain name and its identity principal id, in SQL underscore MI. Finally, az role assignment create gives that principal id the role Cognitive Services OpenAI User, scoped to the Azure OpenAI resource. That is the least-privilege inference role."

**Piece 4, the credential and the model.** "You connect with sqlcmd Go using ActiveDirectoryDefault and the dash I flag for quoted identifiers. First, a database master key if none exists. Then CREATE DATABASE SCOPED CREDENTIAL whose name is the endpoint URL itself, in square brackets, WITH IDENTITY equals the string Managed Identity, and SECRET equals a JSON string with one key, resourceid, whose value is https colon slash slash cognitiveservices dot azure dot com. Then CREATE EXTERNAL MODEL SeedEmbedder WITH LOCATION equal to the endpoint followed by openai slash deployments slash embed dash small slash embeddings, question mark api dash version equals 2024 dash 02 dash 01. API underscore FORMAT is Azure OpenAI. MODEL underscore TYPE is EMBEDDINGS. MODEL is text dash embedding dash 3 dash small. And CREDENTIAL is the credential just created."

**Piece 5, the table and data.** "A schema Catalog and a table Catalog dot Varieties with five columns. VarietyId, an integer, the clustered primary key. Crop, varchar twenty. Name, nvarchar sixty. Description, nvarchar four hundred. And Embedding, VECTOR of one thousand five hundred thirty-six, nullable. An INSERT SELECT from GENERATE underscore SERIES one to one hundred twenty loads one hundred twenty varieties. Crop cycles through tomato, bean, squash, pepper. Name is Variety followed by the number. Description is a generated sentence such as: A sweet tomato that ripens early and tolerates drought. Embedding is left NULL. The comment says the index needs at least one hundred rows with non-NULL vectors."

**Piece 6, steps S1 and S2.** "S1 updates every row, setting Embedding to AI underscore GENERATE underscore EMBEDDINGS of the Name, a colon, and the Description, USE MODEL SeedEmbedder. Then it selects, over rows with a non-NULL Embedding: COUNT star as Embedded, the minimum DATALENGTH of Embedding as Bytes, the minimum VECTORPROPERTY Dimensions as Dims, and the minimum VECTORPROPERTY BaseType as BaseType. S2 declares a JSON variable at p containing dimensions colon five hundred twelve, and inserts row 121, tomato, Shortcut, A test row, with the embedding generated USE MODEL SeedEmbedder PARAMETERS at p."

**Piece 7, steps S3 and S4.** "S3 runs CREATE VECTOR INDEX VX underscore Varieties ON Catalog dot Varieties, on the Embedding column, WITH METRIC equals cosine and TYPE equals diskann. Then it selects, from sys dot vector underscore indexes joined to sys dot indexes, the JSON value dollar dot Version out of build underscore parameters as index underscore version, and distance underscore metric, for the index VX underscore Varieties. S4 inserts row 121, pepper, Late Ember, description: A smoky pepper that ripens late and tolerates heat, with a freshly generated embedding, no PARAMETERS."

**Piece 8, steps S5 to S8, four searches.** "Each of the four declares a query vector at q of type VECTOR 1536 from the phrase late smoky pepper for hot climates, through SeedEmbedder. S5: SELECT TOP 3 VarietyId, Name and distance FROM VECTOR underscore SEARCH with TABLE equals Catalog dot Varieties aliased t, COLUMN equals Embedding, SIMILAR underscore TO equals at q, METRIC equals cosine, and TOP underscore N equals 3, ORDER BY distance. S6: the same, but SELECT TOP 3 WITH APPROXIMATE, and no TOP underscore N inside VECTOR underscore SEARCH, ORDER BY distance. S7: identical to S6 except the ORDER BY is distance, then t dot Name. S8: no WITH APPROXIMATE, no TOP underscore N, METRIC equals euclidean instead of cosine, ORDER BY distance."

**Piece 9, step S9 and cleanup.** "S9 is TRUNCATE TABLE Catalog dot Varieties. For cleanup, az group delete, then az cognitiveservices account purge to release the soft-deleted OpenAI account name. The one hundred twenty-two embedding calls cost well under one cent."

## 3. Setup script (reference only; do not read verbatim unless asked)

```bash
LOCATION="swedencentral"            # a region that offers text-embedding-3-small with the Standard SKU
SUFFIX=$RANDOM
RG="rg-dp800-seedvault-$SUFFIX"
SQL="sql-seedvault-$SUFFIX"
DB="SeedVault"
AOAI="aoai-seedvault-$SUFFIX"       # globally unique; also becomes the custom domain
ADMIN_UPN=$(az ad signed-in-user show --query userPrincipalName -o tsv)
ADMIN_OID=$(az ad signed-in-user show --query id -o tsv)
MYIP=$(curl -s https://api.ipify.org)

az group create -n $RG -l $LOCATION
# Azure OpenAI resource (custom domain is required for Entra-token access) + embedding deployment
az cognitiveservices account create -g $RG -n $AOAI -l $LOCATION --kind OpenAI --sku S0 --custom-domain $AOAI --yes
az cognitiveservices account deployment create -g $RG -n $AOAI --deployment-name embed-small \
  --model-name text-embedding-3-small --model-version "1" --model-format OpenAI --sku-name Standard --sku-capacity 1
  # if your region rejects the Standard SKU for this model, repeat with --sku-name GlobalStandard
AOAI_EP=$(az cognitiveservices account show -g $RG -n $AOAI --query properties.endpoint -o tsv)   # https://<AOAI>.openai.azure.com/
AOAI_ID=$(az cognitiveservices account show -g $RG -n $AOAI --query id -o tsv)

# Logical server with a system-assigned managed identity, Entra admin = you; serverless 1 vCore database
az sql server create -g $RG -n $SQL -l $LOCATION --enable-ad-only-auth --assign-identity --identity-type SystemAssigned \
  --external-admin-principal-type User --external-admin-name "$ADMIN_UPN" --external-admin-sid $ADMIN_OID
az sql server firewall-rule create -g $RG -s $SQL -n client --start-ip-address $MYIP --end-ip-address $MYIP
az sql db create -g $RG -s $SQL -n $DB -e GeneralPurpose -f Gen5 -c 1 --compute-model Serverless \
  --auto-pause-delay 60 --min-capacity 0.5 --backup-storage-redundancy Local
SQL_FQDN=$(az sql server show -g $RG -n $SQL --query fullyQualifiedDomainName -o tsv)
SQL_MI=$(az sql server show -g $RG -n $SQL --query identity.principalId -o tsv)

# Least-privilege inference role for the server identity on the Azure OpenAI resource
az role assignment create --assignee-object-id $SQL_MI --assignee-principal-type ServicePrincipal \
  --role "Cognitive Services OpenAI User" --scope $AOAI_ID
echo "Endpoint: $AOAI_EP"
```

Connect: `sqlcmd -S $SQL_FQDN -d $DB --authentication-method ActiveDirectoryDefault -I` (ODBC sqlcmd: `-G -U "$ADMIN_UPN" -I`). Replace `<AOAI_EP>` with the printed endpoint (keep the trailing slash).

```sql
IF NOT EXISTS (SELECT 1 FROM sys.symmetric_keys WHERE name = '##MS_DatabaseMasterKey##')
    CREATE MASTER KEY ENCRYPTION BY PASSWORD = 'Str0ng!Passw0rd#2026';
GO
CREATE DATABASE SCOPED CREDENTIAL [<AOAI_EP>]
    WITH IDENTITY = 'Managed Identity', SECRET = '{"resourceid":"https://cognitiveservices.azure.com"}';
GO
CREATE EXTERNAL MODEL SeedEmbedder
WITH (LOCATION = '<AOAI_EP>openai/deployments/embed-small/embeddings?api-version=2024-02-01',
      API_FORMAT = 'Azure OpenAI', MODEL_TYPE = EMBEDDINGS, MODEL = 'text-embedding-3-small',
      CREDENTIAL = [<AOAI_EP>]);
GO
CREATE SCHEMA Catalog;
GO
CREATE TABLE Catalog.Varieties
(
    VarietyId   INT           NOT NULL PRIMARY KEY CLUSTERED,
    Crop        VARCHAR(20)   NOT NULL,
    Name        NVARCHAR(60)  NOT NULL,
    Description NVARCHAR(400) NOT NULL,
    Embedding   VECTOR(1536)  NULL
);
-- 120 varieties (the index needs at least 100 rows with non-NULL vectors)
INSERT INTO Catalog.Varieties (VarietyId, Crop, Name, Description)
SELECT value,
       CHOOSE(value % 4 + 1, 'tomato', 'bean', 'squash', 'pepper'),
       CONCAT(N'Variety ', value),
       CONCAT(N'A ', CHOOSE(value % 3 + 1, N'sweet', N'tart', N'smoky'), N' ', CHOOSE(value % 4 + 1, 'tomato', 'bean', 'squash', 'pepper'),
              N' that ripens ', CHOOSE(value % 2 + 1, N'early', N'late'), N' and tolerates ', CHOOSE(value % 5 + 1, N'drought', N'frost', N'heat', N'shade', N'wind'), N'.')
FROM GENERATE_SERIES(1, 120);
GO
```

```sql
-- S1
UPDATE Catalog.Varieties SET Embedding = AI_GENERATE_EMBEDDINGS(CONCAT(Name, N': ', Description) USE MODEL SeedEmbedder);
SELECT COUNT(*) AS Embedded, MIN(DATALENGTH(Embedding)) AS Bytes,
       MIN(VECTORPROPERTY(Embedding, 'Dimensions')) AS Dims, MIN(VECTORPROPERTY(Embedding, 'BaseType')) AS BaseType
FROM Catalog.Varieties WHERE Embedding IS NOT NULL;

-- S2
DECLARE @p JSON = N'{"dimensions":512}';
INSERT INTO Catalog.Varieties (VarietyId, Crop, Name, Description, Embedding)
VALUES (121, 'tomato', N'Shortcut', N'A test row', AI_GENERATE_EMBEDDINGS(N'A test row' USE MODEL SeedEmbedder PARAMETERS @p));

-- S3
CREATE VECTOR INDEX VX_Varieties ON Catalog.Varieties (Embedding) WITH (METRIC = 'cosine', TYPE = 'diskann');
SELECT JSON_VALUE(v.build_parameters, '$.Version') AS index_version, v.distance_metric
FROM sys.vector_indexes AS v JOIN sys.indexes AS i ON i.object_id = v.object_id AND i.index_id = v.index_id
WHERE i.name = 'VX_Varieties';

-- S4
INSERT INTO Catalog.Varieties (VarietyId, Crop, Name, Description, Embedding)
VALUES (121, 'pepper', N'Late Ember', N'A smoky pepper that ripens late and tolerates heat.',
        AI_GENERATE_EMBEDDINGS(N'Late Ember: A smoky pepper that ripens late and tolerates heat.' USE MODEL SeedEmbedder));

-- S5
DECLARE @q VECTOR(1536) = AI_GENERATE_EMBEDDINGS(N'late smoky pepper for hot climates' USE MODEL SeedEmbedder);
SELECT TOP (3) t.VarietyId, t.Name, r.distance
FROM VECTOR_SEARCH(TABLE = Catalog.Varieties AS t, COLUMN = Embedding, SIMILAR_TO = @q, METRIC = 'cosine', TOP_N = 3) AS r
ORDER BY r.distance;

-- S6
DECLARE @q VECTOR(1536) = AI_GENERATE_EMBEDDINGS(N'late smoky pepper for hot climates' USE MODEL SeedEmbedder);
SELECT TOP (3) WITH APPROXIMATE t.VarietyId, t.Name, r.distance
FROM VECTOR_SEARCH(TABLE = Catalog.Varieties AS t, COLUMN = Embedding, SIMILAR_TO = @q, METRIC = 'cosine') AS r
ORDER BY r.distance;

-- S7
DECLARE @q VECTOR(1536) = AI_GENERATE_EMBEDDINGS(N'late smoky pepper for hot climates' USE MODEL SeedEmbedder);
SELECT TOP (3) WITH APPROXIMATE t.VarietyId, t.Name, r.distance
FROM VECTOR_SEARCH(TABLE = Catalog.Varieties AS t, COLUMN = Embedding, SIMILAR_TO = @q, METRIC = 'cosine') AS r
ORDER BY r.distance, t.Name;

-- S8
DECLARE @q VECTOR(1536) = AI_GENERATE_EMBEDDINGS(N'late smoky pepper for hot climates' USE MODEL SeedEmbedder);
SELECT TOP (3) t.VarietyId, t.Name, r.distance
FROM VECTOR_SEARCH(TABLE = Catalog.Varieties AS t, COLUMN = Embedding, SIMILAR_TO = @q, METRIC = 'euclidean') AS r
ORDER BY r.distance;

-- S9
TRUNCATE TABLE Catalog.Varieties;
```

Cleanup: `az group delete -n $RG --yes` then `az cognitiveservices account purge -g $RG -n $AOAI -l $LOCATION`

## 4. The question (ask exactly this)

"For each step, S1 to S9, tell me whether the batch succeeds or fails. For the deterministic parts, tell me what it returns: counts, byte sizes, dimension counts, the index version, error numbers. For S5 to S8 I only ask whether rows come back and whether the vector index is used, not which varieties. Let's go one at a time, starting with S1."

## 5. Answer key (never read aloud unless the learner asks you to reveal it)

| Step | Outcome | Detail |
|---|---|---|
| S1 | Succeeds | Embedded = 120, Bytes = 6152, Dims = 1536, BaseType = float32 |
| S2 | Fails, Msg 42204 | "The vector dimensions 1536 and 512 do not match." The model returned 512 values because of PARAMETERS |
| S3 | Succeeds | Index created; index_version = 3, distance_metric = COSINE. Latest DiskANN format; no PREVIEW_FEATURES needed on Azure SQL Database |
| S4 | Succeeds | 1 row affected. With a latest-version index the table is not read-only |
| S5 | Fails, Msg 42274 | "Vector search with version 3 index does not support explicit TOP_N parameter." |
| S6 | Succeeds | 3 rows, approximate search through VX_Varieties |
| S7 | Fails, Msg 42271 | "TOP WITH APPROXIMATE and VECTOR_SEARCH requires ORDER BY on distance column ascending, and no other columns" |
| S8 | Succeeds | 3 rows. No cosine/euclidean index match, so a warning is raised and an exact kNN scan runs instead; the index is not used |
| S9 | Fails | TRUNCATE TABLE is not allowed on a table with a vector index. Drop the index, truncate, reload at least 100 rows, recreate |

Extra numbers behind the key: 1536 dimensions times 4 bytes plus an 8-byte header equals 6152. Fewer than 100 non-NULL vectors at index creation gives Msg 42266. Omitting ORDER BY with WITH APPROXIMATE gives Msg 42248.

## 6. Hint ladder (one hint per attempt, in order)

**S1**
1. "Three things to settle: does the passwordless call work at all, how many rows get a vector, and how big is each vector."
2. "The server identity holds Cognitive Services OpenAI User on the resource, and the credential says Managed Identity. So the call succeeds. How many rows were loaded by GENERATE underscore SERIES?"
3. "text dash embedding dash 3 dash small returns one thousand five hundred thirty-six numbers by default. Each is a four-byte float32. Add an eight-byte header."
4. "One thousand five hundred thirty-six times four, plus eight. Do the arithmetic."

**S2**
1. "Look at the PARAMETERS JSON. What does dimensions colon five hundred twelve tell Azure OpenAI to do?"
2. "The model obeys and returns five hundred twelve values. What is the declared size of the Embedding column? Can a vector column stretch?"
3. "It is a conversion error about mismatched vector dimensions. The message names both numbers. The error number starts with four two two."

**S3**
1. "Check the preconditions of a vector index: a clustered primary key, and a minimum number of non-NULL vectors. Are both satisfied after S1?"
2. "On Azure SQL Database, does CREATE VECTOR INDEX require the PREVIEW underscore FEATURES database option, as SQL Server 2025 does?"
3. "The catalog reports the index format as a version number inside build underscore parameters. New indexes on Azure get the latest format. It is a small integer, larger than one."

**S4**
1. "This is the classic trap from SQL Server 2025 RTM: the first-format index makes the table read-only. Does the latest format keep that restriction?"
2. "The documentation for the latest format says changes are visible to vector search after the transaction commits. So what happens to a plain INSERT?"

**S5**
1. "Compare S5 with S6. What is the one parameter S5 has inside VECTOR underscore SEARCH that S6 does not?"
2. "With a version-3 index, how do you ask for approximate search: inside the function, or in the SELECT clause?"
3. "TOP underscore N belongs to the older syntax. With a version-3 index it is rejected with an error number starting with four two two."

**S6**
1. "SELECT TOP 3 WITH APPROXIMATE, metric cosine, ORDER BY distance only. Does that match the index you created in S3?"
2. "Every rule is satisfied: matching metric, matching column, ORDER BY on distance ascending alone. So does it run, and through which index?"

**S7**
1. "S7 differs from S6 in exactly one place. Find it in the ORDER BY."
2. "WITH APPROXIMATE has a strict rule about the ORDER BY clause: which columns may appear in it, and in which direction?"
3. "Adding t dot Name breaks that rule. It is an error, not a warning. The number starts with four two two."

**S8**
1. "The index was built with metric cosine. What metric does S8 ask for?"
2. "On Azure SQL Database, when no index matches the requested metric, is that an error or a fallback?"
3. "S8 has no WITH APPROXIMATE. Without it, TOP 3 ORDER BY distance is an exact search anyway. So rows come back, but through what: the index, or a scan?"

**S9**
1. "The table now has a vector index. Is TRUNCATE TABLE allowed on such a table, even though INSERT and DELETE are?"
2. "The documentation lists a specific limitation for TRUNCATE. What is the documented workaround, in three or four steps?"

## 7. Common wrong answers and how to respond

| Learner says | Likely misconception | What to say (without revealing) |
|---|---|---|
| "S1 fails because there is no API key" | Does not know the managed identity credential pattern | "Read the credential again: IDENTITY equals Managed Identity, SECRET has a resourceid. What token does the engine fetch, and with which identity?" |
| "S1 fails because sp_configure external rest endpoint is off" | Applies the SQL Server 2025 switch to Azure | "On Azure SQL Database, is external rest endpoint enabled by default?" |
| "S1 bytes are 6144" | Forgets the 8-byte header | "Four bytes per dimension is right. Is there anything else stored with a vector value?" |
| "S2 succeeds and pads or truncates the vector" | Thinks vector columns are flexible | "A vector column has exactly one dimensionality. What happens when the value has a different one?" |
| "S3 fails because PREVIEW_FEATURES is not on" | Confuses SQL Server 2025 with Azure SQL Database | "That switch exists on SQL Server 2025. Is it required on Azure SQL Database?" |
| "S3 index version is 1" | Assumes the original format | "New indexes on Azure are created in the latest format. Is the latest format the first one?" |
| "S4 fails with 42231, the table is read-only" | Applies the SQL Server 2025 RTM restriction | "That is true for the first-format index. Which format did S3 create, and what does it allow?" |
| "S5 succeeds; TOP_N is how you ask for the top three" | Knows the older syntax only | "Which index version do you have, and does that version accept TOP underscore N?" |
| "S7 succeeds, ordering by name is a tie-breaker" | Does not know the ORDER BY rule for WITH APPROXIMATE | "The rule says ORDER BY on the distance column ascending, and what else?" |
| "S8 fails with 42227, no euclidean index" | Applies the SQL Server 2025 RTM behaviour | "That is the SQL Server 2025 RTM message. On Azure SQL Database, is a metric mismatch an error or a fallback?" |
| "S9 succeeds because DML is fully supported" | Confuses DML with TRUNCATE | "TRUNCATE is not DML in the same sense. Is it listed as a limitation for tables with vector indexes?" |

## 8. Teaching notes (after the answer is complete or revealed)

Start with the passwordless model call:

- The database scoped credential is named after the endpoint URL, `https://<aoai>.openai.azure.com/`. Its name must be a valid URL whose scheme and host match the model LOCATION, whose path is a prefix of it, and with no query string. IDENTITY is the literal string `Managed Identity`, SECRET is `{"resourceid":"https://cognitiveservices.azure.com"}`. The engine obtains a bearer token for the Azure AI services audience with the logical server's system-assigned identity.
- The identity needs Cognitive Services OpenAI User on the Azure OpenAI resource. That is the minimum role that can make inference calls with Entra. Cognitive Services Contributor explicitly cannot.
- `CREATE EXTERNAL MODEL` with LOCATION pointing at `.../openai/deployments/<deployment>/embeddings?api-version=...`, API_FORMAT Azure OpenAI, MODEL_TYPE EMBEDDINGS. `AI_GENERATE_EMBEDDINGS(text USE MODEL m)` returns the model's array. No sp_configure is needed on Azure SQL Database.
- text-embedding-3-small emits 1536 dimensions. A vector(1536) is 4 bytes per dimension plus an 8-byte header, 6152 bytes, BaseType float32. That is S1.
- PARAMETERS JSON is appended to the request body, so `"dimensions":512` makes the model return 512 values. Assigning them to vector(1536) fails with Msg 42204. To shorten, declare the column as vector(512) or set PARAMETERS on the external model. That is S2.

Then the vector index, latest Azure SQL Database format:

- `CREATE VECTOR INDEX ... WITH (METRIC = 'cosine', TYPE = 'diskann')` needs a clustered primary key and at least 100 rows with non-NULL vectors, otherwise Msg 42266. No PREVIEW_FEATURES on Azure. New indexes report `$.Version` = 3 in `sys.vector_indexes.build_parameters`; earlier formats are deprecated and must be dropped and recreated. That is S3.
- The latest format supports full DML: INSERT, UPDATE, DELETE, MERGE. Changes are visible to vector search after commit. Contrast: SQL Server 2025 RTM first-format index makes the table read-only, Msg 42231. That is S4.
- Query syntax follows the index version. Version 3 rejects TOP_N inside VECTOR_SEARCH with Msg 42274. Approximate search is `SELECT TOP (n) WITH APPROXIMATE ... ORDER BY r.distance`. That ORDER BY is mandatory and must be only the distance column, ascending. An extra column gives Msg 42271, no ORDER BY gives Msg 42248. Sort by other columns in an outer query. WHERE predicates are fine and are applied during the search. That is S5, S6, S7.
- Metric mismatch: an ANN index is used only when metric and column match. Otherwise Azure raises a warning and runs exact kNN. Without WITH APPROXIMATE the query is exact anyway, so S8 returns the exact three euclidean neighbours after scanning 121 vectors. On SQL Server 2025 RTM the same query fails with Msg 42227.
- TRUNCATE TABLE is blocked on tables with vector indexes. Workflow: drop index, truncate, reload at least 100 rows, recreate. The same limitation is why vector indexes cannot be deployed through DacPac or BACPAC imports. That is S9.

Memory hook: "Identity plus OpenAI User role, no key. 1536 dims equals 6152 bytes. Version 3: TOP WITH APPROXIMATE, order by distance only, DML yes, TRUNCATE no."

## 9. Follow-up oral questions (optional)

1. "How would you make S2 succeed while keeping the 512-dimension request?" (Declare the column as vector(512), or put PARAMETERS on the EXTERNAL MODEL itself; a vector column has one fixed dimensionality.)
2. "You want the three nearest results sorted by Name. How do you write it with WITH APPROXIMATE?" (Put the approximate TOP 3 ORDER BY distance in a subquery or CTE, and sort by Name in the outer query.)
3. "What would S4 do on SQL Server 2025 RTM with a first-format index?" (Fail with Msg 42231; the table is read-only until the index is dropped.)

## 10. References

- CREATE EXTERNAL MODEL: https://learn.microsoft.com/en-us/sql/t-sql/statements/create-external-model-transact-sql
- AI_GENERATE_EMBEDDINGS: https://learn.microsoft.com/en-us/sql/t-sql/functions/ai-generate-embeddings-transact-sql
- CREATE DATABASE SCOPED CREDENTIAL: https://learn.microsoft.com/en-us/sql/t-sql/statements/create-database-scoped-credential-transact-sql
- Vector data type: https://learn.microsoft.com/en-us/sql/t-sql/data-types/vector-data-type
- Vector indexes: https://learn.microsoft.com/en-us/sql/relational-databases/vectors/vectors-vector-indexes
- CREATE VECTOR INDEX: https://learn.microsoft.com/en-us/sql/t-sql/statements/create-vector-index-transact-sql
- VECTOR_SEARCH: https://learn.microsoft.com/en-us/sql/t-sql/functions/vector-search-transact-sql
- Role-based access control for Azure OpenAI: https://learn.microsoft.com/en-us/azure/ai-services/openai/how-to/role-based-access-control
- DP-800 study guide: https://learn.microsoft.com/en-us/credentials/certifications/resources/study-guides/dp-800
