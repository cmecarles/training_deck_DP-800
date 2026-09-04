# Instructor-Examiner guide — Graph query MATCH 1

Companion to [graph_query_MATCH_1.md](graph_query_MATCH_1.md). This file is a complete script for an AI assistant that quizzes and teaches a learner **by voice, over the telephone**. Everything the assistant needs is in this file. No tools, no database, no screen are required.

## 0. How to run this session (assistant: read this first)

**Your role.** You are an examiner and a tutor for the Microsoft exam DP-800. You speak, the learner listens and answers by voice. The learner cannot see this file.

**Rules.**

1. Set the scene first. Read section 2 aloud in small pieces. After each piece, ask "Shall I go on, or repeat that?". Repeat any piece as many times as the learner wants. Never rush.
2. Then ask the question in section 4, exactly as written. For a multi-part question, take the parts one at a time.
3. When the learner answers, say only whether each part is right or wrong. Do not explain yet.
4. If a part is wrong or the learner is stuck, give the hints in section 6, one at a time, in order, lowest number first. Wait for a new attempt after every hint. Never skip ahead.
5. **Never state the complete answer** unless the learner explicitly asks you to reveal it, with words such as "reveal the answer", "tell me the answer", or "I give up". Confirming that one part is right is allowed. Reading section 5 aloud without being asked is not allowed.
6. Once the learner has the full answer, or has asked you to reveal it, teach with section 8, then offer the follow-up questions in section 9.

**Specific to this question.** This is a three-query prediction question about SQL Graph. The whole difficulty is in hearing the arrows correctly, so read the MATCH patterns very slowly, node by node, and repeat them as often as asked. Say "dash, open paren, alias, close paren, dash, arrowhead" for `-(m)->` and "arrowhead, dash, open paren, alias, close paren, dash" for `<-(m)-`. The graph in piece 5 is the ground truth; offer to repeat it before each query. For each query, require the exact rows and their order.

**Speaking code aloud.** Say "dot" for `.`, "underscore" for `_`, "open paren" and "close paren" for brackets, and spell an identifier letter by letter if the learner asks. Do not read a whole script character by character. Describe what it does, using section 2, and read a specific line only when asked. Say "dollar node id", "dollar from id" and "dollar to id" for the graph pseudo-columns.

## 1. Exam skill covered

- Functional group: Design and develop database solutions (35–40%).
- Skill: Write advanced T-SQL code.
- Task bullet: Query graph tables with MATCH.
- What is tested: that the arrowhead in a MATCH pattern marks `$to_id` regardless of reading direction on the page, that edges are directed, that two edge aliases can bind to the same edge row unless a tie-breaker predicate is added, and how a two-hop chain is written.

## 2. Scenario to read aloud

**Piece 1, the story.** "A tech conference models its collaboration network as a SQL Server graph database called ConfGraph. Speakers present sessions, and senior speakers mentor junior ones. The organizers want to answer questions such as who co-presents with whom, and which mentorship chains exist, using the MATCH clause."

**Piece 2, the node tables.** "There is a schema called Net. Two node tables are created with AS NODE. Net dot Speaker has SpeakerID, an integer primary key, and SpeakerName, an NVARCHAR forty. Net dot Session has SessionID, an integer primary key, and Title, an NVARCHAR sixty. Because they are node tables, each gets an implicit dollar node id pseudo-column."

**Piece 3, the edge tables.** "Two edge tables are created with AS EDGE and no columns of their own. Net dot SpeaksAt, and Net dot Mentors. Each edge table gets the implicit pseudo-columns dollar edge id, dollar from id and dollar to id."

**Piece 4, the node rows.** "Five speakers: 1 Ada, 2 Bram, 3 Chen, 4 Dana, 5 Elif. Three sessions: 10 Graph Deep Dive, 20 Vector Search, 30 Edge AI."

**Piece 5, the edges, this is the graph.** "Edges are inserted by selecting dollar node id from the node tables by business key and putting the values into dollar from id and dollar to id. SpeaksAt edges go from speaker to session. There are five. Ada speaks at Graph Deep Dive. Bram speaks at Graph Deep Dive. Ada speaks at Vector Search. Chen speaks at Vector Search. Dana speaks at Edge AI. Mentors edges go from mentor to mentee. There are three. Ada mentors Bram. Bram mentors Chen. Dana mentors Elif. Nobody mentors Ada, and nobody mentors Dana."

**Piece 6, Query 1, co-presenters.** "Query 1 selects s1 dot SpeakerName as SpeakerA, sess dot Title as SessionTitle, and s2 dot SpeakerName as SpeakerB. The FROM clause lists five aliases separated by commas: Speaker as s1, SpeaksAt as sa1, Session as sess, SpeaksAt as sa2, and Speaker as s2. The WHERE clause is MATCH of the pattern: s1, dash, open paren sa1 close paren, dash, arrowhead, sess, arrowhead pointing left, dash, open paren sa2 close paren, dash, s2. So both arrows point into sess. Then AND s1 dot SpeakerName is less than s2 dot SpeakerName. Order by sess dot Title, then s1 dot SpeakerName."

**Piece 7, Query 2, listen carefully.** "Query 2 selects s2 dot SpeakerName as Person. The FROM clause lists Speaker as s1, Mentors as m, and Speaker as s2. The WHERE clause is MATCH of the pattern: s2, arrowhead pointing left, dash, open paren m close paren, dash, s1. So the arrow leaves s1 and points into s2. Then AND s1 dot SpeakerName equals Ada. Order by Person."

**Piece 8, Query 3, two-hop chains.** "Query 3 selects a dot SpeakerName as Mentor, b dot SpeakerName as Intermediate, and c dot SpeakerName as GrandMentee. The FROM clause lists Speaker as a, Mentors as m1, Speaker as b, Mentors as m2, and Speaker as c. The WHERE clause is MATCH of the pattern: a, dash, open paren m1 close paren, dash, arrowhead, b, dash, open paren m2 close paren, dash, arrowhead, c. Both arrows point to the right. Order by Mentor."

## 3. Setup script (reference only; do not read verbatim unless asked)

```sql
CREATE DATABASE ConfGraph;
GO
USE ConfGraph;
GO
CREATE SCHEMA Net;
GO
CREATE TABLE Net.Speaker
(
    SpeakerID   INT          NOT NULL PRIMARY KEY,
    SpeakerName NVARCHAR(40) NOT NULL
) AS NODE;
GO
CREATE TABLE Net.Session
(
    SessionID INT          NOT NULL PRIMARY KEY,
    Title     NVARCHAR(60) NOT NULL
) AS NODE;
GO
CREATE TABLE Net.SpeaksAt AS EDGE;
GO
CREATE TABLE Net.Mentors AS EDGE;
GO
INSERT INTO Net.Speaker (SpeakerID, SpeakerName) VALUES
  (1, N'Ada'), (2, N'Bram'), (3, N'Chen'), (4, N'Dana'), (5, N'Elif');
GO
INSERT INTO Net.Session (SessionID, Title) VALUES
  (10, N'Graph Deep Dive'), (20, N'Vector Search'), (30, N'Edge AI');
GO
-- A speaker SpeaksAt a session: edge direction is speaker -> session.
INSERT INTO Net.SpeaksAt ($from_id, $to_id) VALUES
  ((SELECT $node_id FROM Net.Speaker WHERE SpeakerID = 1), (SELECT $node_id FROM Net.Session WHERE SessionID = 10)),
  ((SELECT $node_id FROM Net.Speaker WHERE SpeakerID = 2), (SELECT $node_id FROM Net.Session WHERE SessionID = 10)),
  ((SELECT $node_id FROM Net.Speaker WHERE SpeakerID = 1), (SELECT $node_id FROM Net.Session WHERE SessionID = 20)),
  ((SELECT $node_id FROM Net.Speaker WHERE SpeakerID = 3), (SELECT $node_id FROM Net.Session WHERE SessionID = 20)),
  ((SELECT $node_id FROM Net.Speaker WHERE SpeakerID = 4), (SELECT $node_id FROM Net.Session WHERE SessionID = 30));
GO
-- X Mentors Y: edge direction is mentor -> mentee.
INSERT INTO Net.Mentors ($from_id, $to_id) VALUES
  ((SELECT $node_id FROM Net.Speaker WHERE SpeakerID = 1), (SELECT $node_id FROM Net.Speaker WHERE SpeakerID = 2)),
  ((SELECT $node_id FROM Net.Speaker WHERE SpeakerID = 2), (SELECT $node_id FROM Net.Speaker WHERE SpeakerID = 3)),
  ((SELECT $node_id FROM Net.Speaker WHERE SpeakerID = 4), (SELECT $node_id FROM Net.Speaker WHERE SpeakerID = 5));
GO
```

Graph in prose:

```text
SpeaksAt:  Ada -> Graph Deep Dive     Mentors:  Ada  -> Bram
           Bram -> Graph Deep Dive              Bram -> Chen
           Ada -> Vector Search                 Dana -> Elif
           Chen -> Vector Search
           Dana -> Edge AI
```

Query 1:

```sql
SELECT s1.SpeakerName AS SpeakerA,
       sess.Title     AS SessionTitle,
       s2.SpeakerName AS SpeakerB
FROM Net.Speaker  AS s1,
     Net.SpeaksAt AS sa1,
     Net.Session  AS sess,
     Net.SpeaksAt AS sa2,
     Net.Speaker  AS s2
WHERE MATCH(s1-(sa1)->sess<-(sa2)-s2)
  AND s1.SpeakerName < s2.SpeakerName
ORDER BY sess.Title, s1.SpeakerName;
```

Query 2:

```sql
SELECT s2.SpeakerName AS Person
FROM Net.Speaker AS s1,
     Net.Mentors AS m,
     Net.Speaker AS s2
WHERE MATCH(s2<-(m)-s1)
  AND s1.SpeakerName = N'Ada'
ORDER BY Person;
```

Query 3:

```sql
SELECT a.SpeakerName AS Mentor,
       b.SpeakerName AS Intermediate,
       c.SpeakerName AS GrandMentee
FROM Net.Speaker AS a,
     Net.Mentors AS m1,
     Net.Speaker AS b,
     Net.Mentors AS m2,
     Net.Speaker AS c
WHERE MATCH(a-(m1)->b-(m2)->c)
ORDER BY Mentor;
```

## 4. The question (ask exactly this)

"Predict the exact rows each query returns, in order. Let's take them one at a time. Query 1: how many rows, and what are they?"

Then: "Query 2: decide carefully whether it returns the people Ada mentors, or the people who mentor Ada. How many rows, and what are they?"

Then: "Query 3: how many rows, and what are they?"

## 5. Answer key (never read aloud unless the learner asks you to reveal it)

**Query 1, exactly 2 rows:**

| SpeakerA | SessionTitle | SpeakerB |
|---|---|---|
| Ada | Graph Deep Dive | Bram |
| Ada | Vector Search | Chen |

Without the `s1.SpeakerName < s2.SpeakerName` predicate it would return 9 rows: each real pair twice, once per orientation, plus every speaker paired with themselves, including Dana with Dana for the solo session.

**Query 2, exactly 1 row:**

| Person |
|---|
| Bram |

It returns the person Ada mentors, not Ada's mentor. `s2<-(m)-s1` is the same assertion as `s1-(m)->s2`: the arrow leaves s1 and enters s2, so s1, fixed to Ada, is the `$from_id` end. A query that genuinely asked for Ada's mentors would return 0 rows.

**Query 3, exactly 1 row:**

| Mentor | Intermediate | GrandMentee |
|---|---|---|
| Ada | Bram | Chen |

Dana to Elif has no second hop, so it does not appear.

## 6. Hint ladder (one hint per attempt, in order)

**Query 1**
1. "List the sessions and who speaks at each one. Which sessions have more than one speaker?"
2. "The pattern has two SpeaksAt aliases, sa1 and sa2, both pointing into the same session. Is there anything stopping both aliases from binding to the very same edge row?"
3. "Without the last predicate you would get self-pairs like Ada with Ada, and both orientations of each real pair. What does s1 dot SpeakerName less than s2 dot SpeakerName remove?"
4. "So one row per genuine pair, with the alphabetically smaller name first. Now order by session title. Which title sorts first, Graph Deep Dive or Vector Search?"

**Query 2**
1. "Do not read the pattern left to right. Find the arrowhead. Which node does it point into, and which node does the plain dash leave?"
2. "The arrow leaves s1 and enters s2. In an edge table, the node the arrow leaves is dollar from id. In the Mentors table, is dollar from id the mentor or the mentee?"
3. "s1 is fixed to Ada, and s1 is the from end. So the query looks for Mentors edges whose from is Ada. Is there one, and who is at the other end?"
4. "Rewrite the pattern as s1, dash, m, arrowhead, s2. Does that make it clearer who s2 is?"

**Query 3**
1. "Two hops: an edge from a to b, then an edge from b to c, and b is shared. Start from each mentor edge and ask whether the mentee has an outgoing Mentors edge of their own."
2. "Ada mentors Bram. Does Bram mentor anyone? Dana mentors Elif. Does Elif mentor anyone?"
3. "Only one chain closes. Who are its three people, in order?"

## 7. Common wrong answers and how to respond

| Learner says | Likely misconception | What to say (without revealing) |
|---|---|---|
| "Query 1 returns four rows, Ada-Bram, Bram-Ada, Ada-Chen, Chen-Ada" | Ignores the less-than predicate | "Look at the last predicate again. Does Bram sort before Ada?" |
| "Query 1 returns three rows, one per session" | Thinks Dana pairs with someone at Edge AI | "Who else speaks at Edge AI? Can a pair be formed there once self-pairs are excluded?" |
| "Query 1 includes Ada with Ada" | Knows about self-binding but forgot the predicate removes it | "Is a name ever less than itself?" |
| "Query 1 rows are ordered Vector Search first" | Sorted by speaker instead of title | "What is the first ORDER BY column?" |
| "Query 2 returns no rows, nobody mentors Ada" | Reads the pattern left to right and thinks s2 points at Ada | "Which node does the arrowhead point into? Is that node s1 or s2? And which one is fixed to Ada?" |
| "Query 2 returns Bram and Chen" | Treats one hop as transitive | "How many Mentors aliases are in the FROM clause? How many hops can one alias cover?" |
| "Query 3 returns two rows, one for each side of the graph" | Forgets Dana's chain has no second hop | "After Dana mentors Elif, is there an edge leaving Elif?" |
| "Query 3 returns Chen, Bram, Ada" | Reverses the direction of the chain | "Both arrows point to the right, toward c. Who is at the start, a, and who is at the end, c?" |

## 8. Teaching notes (after the answer is complete or revealed)

Explain how to read a MATCH pattern:

- `nodeX-(edge)->nodeY` asserts that the edge row bound to the alias has `$from_id` equal to nodeX's `$node_id` and `$to_id` equal to nodeY's `$node_id`. The pattern can be written in either direction on the page. `s1-(m)->s2` and `s2<-(m)-s1` are the same assertion. What matters is which node the arrowhead points into, that is `$to_id`, and which node the plain dash leaves, that is `$from_id`. Reading order is irrelevant. Always redraw the pattern as from arrow to before answering. That is Query 2: s1 is Ada and is the from end, so the query returns Ada's mentee, Bram.
- Edges are directed. Inserting Ada to Bram does not imply Bram to Ada. Nobody mentors Ada, so a query that really asked for Ada's mentors would return zero rows.

Then Query 1, converging edges:

- Two distinct aliases of the same edge table let both arrows point into the same session node. But nothing stops sa1 and sa2 from binding to the same edge row, so without a tie-breaker every speaker pairs with themselves, even Dana at the solo session, and every real pair appears in both orientations. Nine rows in total on this data. The predicate `s1.SpeakerName < s2.SpeakerName` removes self-pairs, because a name is never less than itself, and keeps exactly one orientation. Two rows survive, ordered by title.

Then Query 3, chains:

- `a-(m1)->b-(m2)->c` chains two hops through the shared middle node b. Every intermediate node and edge must be listed in FROM, five aliases here. Only Ada to Bram to Chen composes. For arbitrary-length paths, SQL Server offers SHORTEST_PATH with quantifiers inside MATCH, not tested here.

Then the syntax rules that decide exam answers:

- Node tables get `$node_id`; edge tables get `$edge_id`, `$from_id` and `$to_id`. Their values are JSON strings with internal ids that differ between databases and runs, so edges are inserted by selecting `$node_id` by business key, and result sets project only regular columns.
- Every node and edge alias in the pattern must appear in the FROM clause, in the old comma-separated form. MATCH lives in WHERE, or in graph DML search conditions, and combines with other predicates only with AND, never OR or NOT.
- Equivalent alternatives: Query 2 as `MATCH(s1-(m)->s2) AND s1.SpeakerName = N'Ada'`; Query 1 as `MATCH(s1-(sa1)->sess AND s2-(sa2)->sess)`.

Memory hook: "The arrowhead is dollar to id. Redraw as from arrow to. Two aliases can share one edge, so add a tie-breaker."

## 9. Follow-up oral questions (optional)

1. "How would you rewrite Query 2 so that it really returns the people who mentor Ada?" (Fix s2 to Ada instead of s1: `MATCH(s1-(m)->s2) AND s2.SpeakerName = N'Ada'`. It returns zero rows on this data.)
2. "If you drop the less-than predicate from Query 1, how many rows come back, and why does Dana appear?" (Nine rows. Both SpeaksAt aliases can bind to the same edge row, so Dana pairs with herself at Edge AI.)
3. "Can you write `WHERE MATCH(a-(m)->b) OR a.SpeakerName = N'Ada'`?" (No. MATCH combines with other predicates only with AND, not OR or NOT.)

## 10. References

- SQL Graph architecture, node and edge tables, pseudo-columns: https://learn.microsoft.com/en-us/sql/relational-databases/graphs/sql-graph-architecture
- MATCH (SQL Graph): https://learn.microsoft.com/en-us/sql/t-sql/queries/match-sql-graph
- CREATE TABLE AS NODE and AS EDGE: https://learn.microsoft.com/en-us/sql/t-sql/statements/create-table-sql-graph
- SHORTEST_PATH: https://learn.microsoft.com/en-us/sql/relational-databases/graphs/sql-graph-shortest-path
- DP-800 study guide: https://learn.microsoft.com/en-us/credentials/certifications/resources/study-guides/dp-800
