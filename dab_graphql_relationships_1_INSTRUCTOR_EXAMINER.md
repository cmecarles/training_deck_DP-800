# Instructor-Examiner guide — DAB GraphQL relationships 1

Companion to [dab_graphql_relationships_1.md](dab_graphql_relationships_1.md). This file is a complete script for an AI assistant that quizzes and teaches a learner **by voice, over the telephone**. Everything the assistant needs is in this file. No tools, no database, no screen are required.

## 0. How to run this session (assistant: read this first)

**Your role.** You are an examiner and a tutor for the Microsoft exam DP-800. You speak, the learner listens and answers by voice. The learner cannot see this file.

**Rules.**

1. Set the scene first. Read section 2 aloud in small pieces. After each piece, ask "Shall I go on, or repeat that?". Repeat any piece as many times as the learner wants. Never rush.
2. Then ask the question in section 4, exactly as written. For a multi-part question, take the parts one at a time.
3. When the learner answers, say only whether each part is right or wrong. Do not explain yet.
4. If a part is wrong or the learner is stuck, give the hints in section 6, one at a time, in order, lowest number first. Wait for a new attempt after every hint. Never skip ahead.
5. **Never state the complete answer** unless the learner explicitly asks you to reveal it, with words such as "reveal the answer", "tell me the answer", or "I give up". Confirming that one part is right is allowed. Reading section 5 aloud without being asked is not allowed.
6. Once the learner has the full answer, or has asked you to reveal it, teach with section 8, then offer the follow-up questions in section 9.

**Specific to this question.** This is a multiple-choice question with four options, a to d. Every option is a JSON relationships block. The four blocks differ in a handful of values, so read each one slowly, naming each relationship, its cardinality, its target entity, its source and target fields, and its linking keys if any. Read the GraphQL query and the expected response before the options. Read all four options before taking an answer.

**Speaking code aloud.** Say "dot" for `.`, "underscore" for `_`, "open paren" and "close paren" for brackets, and spell an identifier letter by letter if the learner asks. Do not read a whole script character by character. Describe what it does, using section 2, and read a specific line only when asked.

## 1. Exam skill covered

- Functional group: Design and develop database solutions (35–40%).
- Skill: Expose data through APIs.
- Task bullet: Configure Data API builder entities and relationships for GraphQL.
- What is tested: how cardinality shapes the GraphQL field, how source and target fields mirror a foreign key, how a many-to-many relationship is declared with a linking object and linking fields, and the limits of views and stored procedures in DAB.

## 2. Scenario to read aloud

**Piece 1, the story.** "A hiking-app company stores its trail catalog in an Azure SQL database named TrailGuide. There is a schema called Outdoors with four tables, a view and a stored procedure. The team exposes the database with Data API builder, DAB for short."

**Piece 2, the tables Region and Trail.** "Outdoors dot Region has two columns: RegionId, an integer, the primary key, and Name, up to sixty characters. Outdoors dot Trail has five columns: TrailId, an integer, the primary key. Name, up to eighty characters. LengthKm, a decimal five comma one. IsOpen, a bit. And RegionId, an integer with a foreign key called FK underscore Trail underscore Region to Region dot RegionId."

**Piece 3, the tables Tag and TrailTag.** "Outdoors dot Tag has TagId, an integer, the primary key, and Label, up to thirty characters. Outdoors dot TrailTag is a linking table for a many-to-many relationship. It has TrailId, with a foreign key to Trail, and TagId, with a foreign key to Tag, and a composite primary key on TrailId and TagId. There is no foreign key between Trail and Tag directly. They are related only through TrailTag."

**Piece 4, the data.** "Two regions: 1, Pyrenees, and 2, Dolomites. Three trails: 10, Ordesa Canyon, seventeen point five kilometres, open, region 1. 11, Aneto Summit, twenty-two kilometres, not open, region 1. 12, Tre Cime Loop, nine point eight, open, region 2. Three tags: 1 family, 2 alpine, 3 waterfalls. Five linking rows: trail 10 with tags 1 and 3; trail 11 with tag 2; trail 12 with tags 1 and 2."

**Piece 5, the view and the procedure.** "A view, Outdoors dot OpenTrails, selects TrailId, Name, LengthKm and the region name as RegionName, joining Trail to Region, where IsOpen equals 1. A stored procedure, Outdoors dot GetTrailsByRegion, takes at RegionId and returns TrailId, Name and LengthKm for that region, ordered by TrailId. The DAB config already has entities Region, Trail and Tag for the tables, OpenTrails for the view and GetTrailsByRegion for the procedure, all with anonymous read or execute permission."

**Piece 6, the GraphQL query.** "The mobile app must run one GraphQL query. It calls region underscore by underscore pk with RegionId 1, and asks for Name, then trails, then inside trails, items, and inside items, TrailId, Name, LengthKm, and tags, and inside tags, items, and Label. So trails and tags are both navigated through an items list."

**Piece 7, the expected response.** "The response has region Name Pyrenees, and trails items with two entries. Trail 10, Ordesa Canyon, seventeen point five, with tags items family and waterfalls. Trail 11, Aneto Summit, twenty-two, with tags items alpine. Additionally, the query trail underscore by underscore pk with TrailId 10, then region, then Name, must return the trail's single region, without an items list."

**Piece 8, option a.** "Option a. On entity Region, a relationship named trails: cardinality many, target entity Trail, source fields RegionId, target fields RegionId. On entity Trail, two relationships. Region: cardinality one, target entity Region, source fields RegionId, target fields RegionId. Tags: cardinality many, target entity Tag, source fields TrailId, target fields TagId, linking object Outdoors dot TrailTag, linking source fields TrailId, linking target fields TagId."

**Piece 9, option b.** "Option b is identical to option a except in the tags relationship's linking fields: linking source fields is TagId and linking target fields is TrailId. The two linking fields are swapped."

**Piece 10, option c.** "Option c is identical to option a except that the tags relationship has no linking object and no linking fields at all. It only has cardinality many, target entity Tag, source fields TrailId and target fields TagId."

**Piece 11, option d.** "Option d has the same tags relationship as option a, with the correct linking keys. But the cardinalities of the other two are reversed: Region dot trails has cardinality one, and Trail dot region has cardinality many."

## 3. Setup script (reference only; do not read verbatim unless asked)

```sql
CREATE DATABASE TrailGuide;
GO
USE TrailGuide;
GO
CREATE SCHEMA Outdoors;
GO
CREATE TABLE Outdoors.Region
(
    RegionId INT          NOT NULL PRIMARY KEY,
    Name     NVARCHAR(60) NOT NULL
);
CREATE TABLE Outdoors.Trail
(
    TrailId   INT          NOT NULL PRIMARY KEY,
    Name      NVARCHAR(80) NOT NULL,
    LengthKm  DECIMAL(5,1) NOT NULL,
    IsOpen    BIT          NOT NULL,
    RegionId  INT          NOT NULL
        CONSTRAINT FK_Trail_Region REFERENCES Outdoors.Region (RegionId)
);
CREATE TABLE Outdoors.Tag
(
    TagId INT          NOT NULL PRIMARY KEY,
    Label NVARCHAR(30) NOT NULL
);
CREATE TABLE Outdoors.TrailTag           -- linking table (many-to-many)
(
    TrailId INT NOT NULL CONSTRAINT FK_TrailTag_Trail REFERENCES Outdoors.Trail (TrailId),
    TagId   INT NOT NULL CONSTRAINT FK_TrailTag_Tag   REFERENCES Outdoors.Tag (TagId),
    CONSTRAINT PK_TrailTag PRIMARY KEY (TrailId, TagId)
);
GO
INSERT INTO Outdoors.Region VALUES (1, N'Pyrenees'), (2, N'Dolomites');
INSERT INTO Outdoors.Trail VALUES
  (10, N'Ordesa Canyon', 17.5, 1, 1),
  (11, N'Aneto Summit',  22.0, 0, 1),
  (12, N'Tre Cime Loop',  9.8, 1, 2);
INSERT INTO Outdoors.Tag VALUES (1, N'family'), (2, N'alpine'), (3, N'waterfalls');
INSERT INTO Outdoors.TrailTag VALUES (10,1), (10,3), (11,2), (12,1), (12,2);
GO
CREATE VIEW Outdoors.OpenTrails AS
SELECT t.TrailId, t.Name, t.LengthKm, r.Name AS RegionName
FROM Outdoors.Trail AS t JOIN Outdoors.Region AS r ON r.RegionId = t.RegionId
WHERE t.IsOpen = 1;
GO
CREATE PROCEDURE Outdoors.GetTrailsByRegion @RegionId INT AS
BEGIN
    SET NOCOUNT ON;
    SELECT TrailId, Name, LengthKm FROM Outdoors.Trail WHERE RegionId = @RegionId ORDER BY TrailId;
END;
GO
```

The GraphQL query and expected response:

```graphql
{
  region_by_pk(RegionId: 1) {
    Name
    trails {
      items {
        TrailId Name LengthKm
        tags { items { Label } }
      }
    }
  }
}
```

```json
{ "data": { "region_by_pk": { "Name": "Pyrenees", "trails": { "items": [
  { "TrailId": 10, "Name": "Ordesa Canyon", "LengthKm": 17.5,
    "tags": { "items": [ { "Label": "family" }, { "Label": "waterfalls" } ] } },
  { "TrailId": 11, "Name": "Aneto Summit",  "LengthKm": 22.0,
    "tags": { "items": [ { "Label": "alpine" } ] } } ] } } } }
```

Option a, the correct block:

```json
"Region": { "relationships": { "trails": {
    "cardinality": "many", "target.entity": "Trail",
    "source.fields": [ "RegionId" ], "target.fields": [ "RegionId" ] } } },
"Trail": { "relationships": {
    "region": { "cardinality": "one", "target.entity": "Region",
                "source.fields": [ "RegionId" ], "target.fields": [ "RegionId" ] },
    "tags":   { "cardinality": "many", "target.entity": "Tag",
                "source.fields": [ "TrailId" ], "target.fields": [ "TagId" ],
                "linking.object": "Outdoors.TrailTag",
                "linking.source.fields": [ "TrailId" ], "linking.target.fields": [ "TagId" ] } } }
```

Option b: as a, but `"linking.source.fields": [ "TagId" ], "linking.target.fields": [ "TrailId" ]`.
Option c: as a, but the `tags` relationship has no `linking.object`, `linking.source.fields` or `linking.target.fields`.
Option d: as a, but `Region.trails` has `"cardinality": "one"` and `Trail.region` has `"cardinality": "many"`.

CLI equivalent of the many-to-many entry:

```bash
dab update Trail --relationship tags --target.entity Tag --cardinality many --relationship.fields "TrailId:TagId" --linking.object Outdoors.TrailTag --linking.source.fields TrailId --linking.target.fields TagId
```

## 4. The question (ask exactly this)

"Which relationships configuration should you add?

a. Region dot trails: cardinality many, target entity Trail, source fields RegionId, target fields RegionId. Trail dot region: cardinality one, target entity Region, source fields RegionId, target fields RegionId. Trail dot tags: cardinality many, target entity Tag, source fields TrailId, target fields TagId, linking object Outdoors dot TrailTag, linking source fields TrailId, linking target fields TagId.

b. Same as a, but Trail dot tags has linking source fields TagId and linking target fields TrailId.

c. Same as a, but Trail dot tags has no linking object and no linking fields.

d. Same as a, but Region dot trails has cardinality one and Trail dot region has cardinality many.

Which letter, and what is wrong with each of the other three?"

## 5. Answer key (never read aloud unless the learner asks you to reveal it)

**Correct answer: a.**

| Option | Verdict | Why |
|---|---|---|
| a | Correct | Region.trails many gives trails as a connection with items. Trail.region one gives a single object. Trail.tags many with linking.object Outdoors.TrailTag, linking.source.fields TrailId pointing to the source Trail, linking.target.fields TagId pointing to the target Tag. Produces exactly the expected response, items ordered by primary key. |
| b | Wrong | Linking fields swapped. DAB would join Trail.TrailId to TrailTag.TagId and TrailTag.TrailId to Tag.TagId. Trail 10 would look for linking rows with TagId 10; none exist, so tags.items comes back empty for every trail, silently. |
| c | Wrong | No linking.object. DAB treats tags as a direct relationship and joins Trail.TrailId to Tag.TagId, which is meaningless; trail 10 would pair with a non-existent tag 10. Validation says linking.object must exist for many-to-many. |
| d | Wrong | Cardinalities reversed. Region.trails one generates a single Trail object, so trails with items is a GraphQL validation error. Trail.region many generates a RegionConnection, so region with Name and no items fails too. |

Engine-verified equivalent of the correct join: region Pyrenees, trail 10 Ordesa Canyon with tags family and waterfalls, trail 11 Aneto Summit with tag alpine.

## 6. Hint ladder (one hint per attempt, in order)

1. "Look at the query shape. Trails is read through items, and region is read directly. Which cardinality gives a connection with items, and which gives a single object? Now check the cardinalities in each option."
2. "One option reverses the cardinalities of the region and trail pair. Cardinality describes the target side: how many regions per trail, how many trails per region. Eliminate it."
3. "Trail and Tag have no direct foreign key. Which key in a relationship tells DAB to go through a join table? One option lacks it. What would DAB join then?"
4. "Two options remain, differing only in the linking fields. Linking source fields is the column of the join table that points back to the source entity, Trail. Which column of TrailTag points to Trail?"
5. "With the linking fields swapped, trail 10 would search TrailTag for TagId equal to 10. Do any such rows exist? Which option avoids that?"

## 7. Common wrong answers and how to respond

| Learner says | Likely misconception | What to say (without revealing) |
|---|---|---|
| "Option d, a region has one trails field so cardinality one" | Reads cardinality as the source side | "Cardinality describes the target. How many trails does one region have? And does the query read trails through items?" |
| "Option c, DAB finds the join table from the foreign keys" | Expects automatic many-to-many discovery | "Is there any foreign key between Trail and Tag directly? Without a linking object, which two columns would DAB compare?" |
| "Option b, the linking fields do not matter" | Thinks the two linking keys are symmetric | "Trace it: linking source fields TagId means Trail dot TrailId equals TrailTag dot TagId. For trail 10, what rows match?" |
| "Option b, source fields is TrailId so it is fine" | Confuses source fields with linking source fields | "Source fields is the column on Trail. Linking source fields is the column on TrailTag. Which TrailTag column references Trail?" |
| "The target entity should be Outdoors dot Tag" | Confuses entity name with table name | "Does target dot entity name a table or an entity in the config?" |
| "Tags should hang off the stored procedure" | Does not know procedure limits | "Can a stored procedure entity be the source or target of a relationship in DAB?" |

## 8. Teaching notes (after the answer is complete or revealed)

Explain how DAB relationships work:

- **Where they live.** Relationships exist only in GraphQL; REST has no nested navigation. They are declared per entity under entities dot name dot relationships dot relationship dash name, and the relationship name becomes the GraphQL field: trails, region, tags.
- **Cardinality shapes the field.** One gives a single object, so Trail dot region is queried as region then Name. Many gives a paginated connection, so Region dot trails is queried as trails then items. Cardinality describes the target side. Reversing it, as in option d, makes both queries fail GraphQL validation.
- **One-to-many and many-to-one.** target dot entity names the other entity, not the table. source dot fields and target dot fields name the join columns and mirror the foreign key, RegionId to RegionId here.
- **Many-to-many.** Add linking dot object, the join table, which need not be an entity. Add linking dot source dot fields, the join table column that references the source entity, TrailId, and linking dot target dot fields, the join table column that references the target entity, TagId. Without linking dot object, DAB joins source dot fields to target dot fields directly, which is option c's error; validation says linking dot object must exist for many-to-many. Swapping the two linking keys, option b, joins the wrong columns and returns empty items silently.
- **Ordering.** Items come back ordered by primary key by default, which is why trail 10 precedes 11 and tags family precede waterfalls.
- **Views.** OpenTrails has no discoverable primary key, so the entity needs source dot key dash fields, TrailId, or in DAB 2.0 the fields entry with primary dash key true; the two cannot coexist. Only then can openTrails underscore by underscore pk or REST slash api slash OpenTrails slash TrailId slash 10 work.
- **Stored procedures.** GetTrailsByRegion appears as executeGetTrailsByRegion with RegionId, under Mutation unless graphql dot operation is set to query. Only the first result set is returned. No pagination, filtering, ordering or relationships, and no fetch by key. You cannot hang tags off it.

Memory hook: "One is an object, many is a connection with items. Mirror the foreign key with source and target fields. Many-to-many adds the linking object, and linking source points back to the source."

## 9. Follow-up oral questions (optional)

1. "What must you add to the OpenTrails entity before openTrails underscore by underscore pk works?" (source dot key dash fields with TrailId, or fields with primary dash key true in DAB 2.0.)
2. "By default, is executeGetTrailsByRegion a query or a mutation, and how do you change it?" (A mutation; set graphql dot operation to query on the entity.)
3. "What CLI command adds the tags relationship?" (dab update Trail with relationship tags, target dot entity Tag, cardinality many, relationship dot fields TrailId colon TagId, linking dot object Outdoors dot TrailTag, linking dot source dot fields TrailId, linking dot target dot fields TagId.)

## 10. References

- Data API builder relationships: https://learn.microsoft.com/en-us/azure/data-api-builder/concept/database-objects/relationships
- Data API builder configuration reference, entities and relationships: https://learn.microsoft.com/en-us/azure/data-api-builder/reference-configuration
- Data API builder CLI reference, dab update: https://learn.microsoft.com/en-us/azure/data-api-builder/reference-command-line-interface
- Views and stored procedures in Data API builder: https://learn.microsoft.com/en-us/azure/data-api-builder/concept/database-objects/views-and-stored-procedures
- Data API builder GraphQL endpoint: https://learn.microsoft.com/en-us/azure/data-api-builder/graphql
- Data API builder GitHub repository: https://github.com/Azure/data-api-builder
- DP-800 study guide: https://learn.microsoft.com/en-us/credentials/certifications/resources/study-guides/dp-800
