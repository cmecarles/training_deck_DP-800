# Instructor-Examiner guide — Row-Level Security 2

Companion to [row_level_security_2.md](row_level_security_2.md). This file is a complete script for an AI assistant that quizzes and teaches a learner **by voice, over the telephone**. Everything the assistant needs is in this file. No tools, no database, no screen are required.

## 0. How to run this session (assistant: read this first)

**Your role.** You are an examiner and a tutor for the Microsoft exam DP-800. You speak, the learner listens and answers by voice. The learner cannot see this file.

**Rules.**

1. Set the scene first. Read section 2 aloud in small pieces. After each piece, ask "Shall I go on, or repeat that?". Repeat any piece as many times as the learner wants. Never rush.
2. Then ask the question in section 4, exactly as written. For a multi-part question, take the parts one at a time.
3. When the learner answers, say only whether each part is right or wrong. Do not explain yet.
4. If a part is wrong or the learner is stuck, give the hints in section 6, one at a time, in order, lowest number first. Wait for a new attempt after every hint. Never skip ahead.
5. **Never state the complete answer** unless the learner explicitly asks you to reveal it, with words such as "reveal the answer", "tell me the answer", or "I give up". Confirming that one part is right is allowed. Reading section 5 aloud without being asked is not allowed.
6. Once the learner has the full answer, or has asked you to reveal it, teach with section 8, then offer the follow-up questions in section 9.

**Specific to this question.** This is a multiple-choice question with four options, a to d. All four options use the same security policy; only the WHERE clause of the predicate function differs, and the difference is in the AND and OR logic. Read all four option pieces slowly, and offer to repeat the logic of any option before taking an answer. Ask the learner to test their chosen option against all three principals.

**Speaking code aloud.** Say "dot" for `.`, "underscore" for `_`, "open paren" and "close paren" for brackets, and spell an identifier letter by letter if the learner asks. Do not read a whole script character by character. Describe what it does, using section 2, and read a specific line only when asked.

## 1. Exam skill covered

- Functional group: Secure, optimize, and deploy database solutions (35–40%).
- Skill: Implement data security and compliance.
- Task bullet: Implement row-level security.
- What is tested: combining a SESSION_CONTEXT check with a DATABASE_PRINCIPAL_ID check inside an RLS predicate, so that only the intended principal can use the application identity.

## 2. Scenario to read aloud

**Piece 1, the story.** "An organization runs a multi-tenant application on Azure SQL Database. Application users never connect to the database directly. The middle tier connects with one database principal, AppUser. At the start of each request the application stores the current application user's number in session context, calling sys dot sp underscore set underscore session underscore context with the key UserId and a value such as forty-two."

**Piece 2, the table.** "There is one table, Sales dot OrderData. Three columns. OrderId, a bigint, the primary key. AppUserId, an integer, not null. And Amount, a decimal with twelve digits and two decimals, not null."

**Piece 3, the three principals.** "Three database principals already exist. AppUser, used only by the middle-tier application. SupportAdmin, used by support administrators. And ReportingUser, used by a separate reporting workload. Assume all three have already been granted SELECT and INSERT on Sales dot OrderData."

**Piece 4, the requirements.** "Five requirements. One: AppUser can SELECT only rows whose AppUserId matches the UserId in session context. Two: AppUser cannot insert a row whose AppUserId does not match the session context. Three: SupportAdmin can SELECT all rows and INSERT rows for any AppUserId. Four: ReportingUser must see no rows and must be unable to insert any rows, even if ReportingUser calls sp underscore set underscore session underscore context itself and supplies a valid UserId. Five: the database enforces all of this; no application WHERE clauses."

**Piece 5, what is common to all options.** "A schema called Security is created. Every option defines an inline table-valued function, Security dot fn underscore OrderAccess, with one integer parameter, AppUserId, created WITH SCHEMABINDING, returning SELECT 1 AS AccessResult with a WHERE clause. Every option then creates the same security policy, Security dot OrderPolicy, with a FILTER PREDICATE on Sales dot OrderData and a BLOCK PREDICATE AFTER INSERT on Sales dot OrderData, both using that function on the AppUserId column, STATE equals ON. So the only thing that changes from option to option is the WHERE clause of the function. I will read each one carefully."

**Piece 6, option a.** "Option a's WHERE clause is: DATABASE underscore PRINCIPAL underscore ID open paren close paren is not equal to DATABASE underscore PRINCIPAL underscore ID of AppUser, OR the parameter AppUserId equals the session context UserId cast to int. In words: either you are anyone other than AppUser, or the row matches the session context."

**Piece 7, option b.** "Option b's WHERE clause is: open paren, the current principal equals AppUser, AND the parameter AppUserId equals the session context UserId, close paren, OR the current principal equals SupportAdmin. In words: either you are AppUser and the row matches the session context, or you are SupportAdmin."

**Piece 8, option c.** "Option c's WHERE clause is: the parameter AppUserId equals the session context UserId, OR the current principal equals SupportAdmin. In words: either the row matches the session context, whoever you are, or you are SupportAdmin."

**Piece 9, option d.** "Option d's WHERE clause is: open paren, the current principal equals AppUser, OR the parameter AppUserId equals the session context UserId, close paren, AND the current principal is not equal to ReportingUser. In words: you are AppUser or the row matches the session context, and in either case you are not ReportingUser. Notice that inside the parentheses the connector is OR, not AND."

## 3. Setup script (reference only; do not read verbatim unless asked)

```sql
EXEC sys.sp_set_session_context
    @key = N'UserId',
    @value = 42;

CREATE TABLE Sales.OrderData
(
    OrderId    bigint NOT NULL PRIMARY KEY,
    AppUserId  int NOT NULL,
    Amount     decimal(12,2) NOT NULL
);
GO

CREATE SCHEMA Security;
GO

-- Policy, identical in all four options
CREATE SECURITY POLICY Security.OrderPolicy
ADD FILTER PREDICATE
    Security.fn_OrderAccess(AppUserId)
    ON Sales.OrderData,
ADD BLOCK PREDICATE
    Security.fn_OrderAccess(AppUserId)
    ON Sales.OrderData AFTER INSERT
WITH (STATE = ON);
GO

-- Option a: WHERE clause of Security.fn_OrderAccess
    WHERE
        DATABASE_PRINCIPAL_ID() <>
            DATABASE_PRINCIPAL_ID(N'AppUser')
        OR
        @AppUserId =
            CAST(SESSION_CONTEXT(N'UserId') AS int)

-- Option b: WHERE clause of Security.fn_OrderAccess
    WHERE
        (
            DATABASE_PRINCIPAL_ID() =
                DATABASE_PRINCIPAL_ID(N'AppUser')
            AND
            @AppUserId =
                CAST(SESSION_CONTEXT(N'UserId') AS int)
        )
        OR
        DATABASE_PRINCIPAL_ID() =
            DATABASE_PRINCIPAL_ID(N'SupportAdmin')

-- Option c: WHERE clause of Security.fn_OrderAccess
    WHERE
        @AppUserId =
            CAST(SESSION_CONTEXT(N'UserId') AS int)
        OR
        DATABASE_PRINCIPAL_ID() =
            DATABASE_PRINCIPAL_ID(N'SupportAdmin')

-- Option d: WHERE clause of Security.fn_OrderAccess
    WHERE
        (
            DATABASE_PRINCIPAL_ID() =
                DATABASE_PRINCIPAL_ID(N'AppUser')
            OR
            @AppUserId =
                CAST(SESSION_CONTEXT(N'UserId') AS int)
        )
        AND
        DATABASE_PRINCIPAL_ID() <>
            DATABASE_PRINCIPAL_ID(N'ReportingUser')

-- Full function shape (same in every option, only the WHERE differs)
CREATE FUNCTION Security.fn_OrderAccess
(
    @AppUserId int
)
RETURNS TABLE
WITH SCHEMABINDING
AS
RETURN
(
    SELECT 1 AS AccessResult
    WHERE <option-specific clause>
);
GO
```

## 4. The question (ask exactly this)

"Which implementation should you use? Option a: not AppUser, or row matches session context. Option b: AppUser and row matches session context, or SupportAdmin. Option c: row matches session context, or SupportAdmin. Option d: AppUser or row matches session context, and not ReportingUser. Which one, and why?"

## 5. Answer key (never read aloud unless the learner asks you to reveal it)

**Correct answer: b.**

Test each principal against option b:

- AppUser: principal check passes, so the row must also match the session context. The filter limits SELECT, the AFTER INSERT block rejects inserts for other application users.
- SupportAdmin: the second branch is true regardless of AppUserId. All rows visible, any insert allowed.
- ReportingUser: neither AppUser nor SupportAdmin. Even after setting UserId to 42 in session context, the AppUser check fails. No rows visible, no inserts allowed.

Why each wrong option is wrong:

- **a**: "not AppUser" is true for SupportAdmin and for ReportingUser. ReportingUser, who already has SELECT and INSERT permission, sees and inserts everything. Violates requirement four.
- **c**: the session-context branch is not tied to any principal. ReportingUser sets UserId to 42 and gains access to the rows of application user 42. Violates requirement four.
- **d**: inside the parentheses the connector is OR. For AppUser the first part is true simply because the principal is AppUser, so AppUser sees every row and can insert for any application user. Violates requirements one and two.

## 6. Hint ladder (one hint per attempt, in order)

1. "Do not judge the options by how they look. Take one option and run three test cases through it: AppUser, SupportAdmin, ReportingUser. What does each one see?"
2. "Requirement four is the sharp one: ReportingUser can set the session context to any value it likes. Which options let a session-context match, on its own, open the door?"
3. "Session context is not an authorization boundary. Any session can set it. So a strong predicate must ask two questions of AppUser: are you really AppUser, and does the row match. Which connector joins those two questions, AND or OR?"
4. "Option a says 'not AppUser'. Who else is 'not AppUser'? Is ReportingUser one of them?"
5. "Two options remain. In one of them, the parentheses contain an OR between the AppUser check and the session-context check. Evaluate that for AppUser looking at a row of application user 99."

## 7. Common wrong answers and how to respond

| Learner says | Likely misconception | What to say (without revealing) |
|---|---|---|
| "a, because SupportAdmin is not AppUser and gets everything" | Only tests SupportAdmin, forgets ReportingUser | "SupportAdmin is not the only principal that is 'not AppUser'. Try the third one." |
| "c, it is the simplest that lets SupportAdmin through" | Trusts SESSION_CONTEXT as an identity | "Who can call sp underscore set underscore session underscore context? Only AppUser?" |
| "d, it explicitly excludes ReportingUser, so requirement four is safe" | Reads the exclusion and stops before checking AppUser | "Requirement four is fine in d. Now evaluate the parentheses for AppUser and a row of user 99." |
| "b and d are the same" | Misses AND versus OR inside the parentheses | "Read the connector inside the parentheses of each. One says AND, one says OR. Which matters for AppUser?" |
| "ReportingUser cannot set session context, it is an app thing" | Does not know any session can set its own context | "The requirement text tells you ReportingUser might call the procedure. Assume it does." |

## 8. Teaching notes (after the answer is complete or revealed)

The question tests two independent identities:

- The **database principal** executing the statement, reported by DATABASE underscore PRINCIPAL underscore ID.
- The **application identity** stored in SESSION_CONTEXT, which the database trusts only because the predicate chooses to.

Any session can write to its own session context. So a predicate that says "row matches session context" and nothing else is only as safe as the set of principals holding permission on the table. Here three principals have SELECT and INSERT, so the predicate must bind the session-context check to the principal it is meant for.

The authorization test for a tenant-style principal is therefore an AND: is this the expected principal, AND does the row belong to the application identity. Administrative bypass is a separate OR branch naming the administrative principal explicitly. Anyone else falls through and gets nothing, which is exactly what requirement four asks for ReportingUser, with no need to name it.

Why the distractors fail:

- Option a uses "not AppUser" as if it meant "SupportAdmin". It also matches ReportingUser.
- Option c leaves the session-context branch unguarded, so ReportingUser can impersonate application user 42 by setting the context.
- Option d gets the exclusion right but the parentheses wrong: OR lets AppUser through on identity alone, without the row check, destroying tenant isolation.

Also note what is shared by all options: a FILTER predicate for reads plus a BLOCK AFTER INSERT for writes, the same pattern as the previous row-level security question. The policy is not where this question is decided.

Memory hook: "Principal AND session context for the app. OR the admin by name. Everyone else gets nothing."

## 9. Follow-up oral questions (optional)

1. "Why is the session-context value cast to int in every option?" (SESSION_CONTEXT returns sql underscore variant; the cast makes the comparison with the integer parameter explicit.)
2. "If a new principal, AuditUser, is later granted SELECT on the table, what does option b let it see?" (Nothing. It is neither AppUser nor SupportAdmin, so the predicate returns no row.)
3. "Option b protects reads and inserts. What would you add so AppUser cannot update a row to another user's AppUserId?" (A BLOCK PREDICATE AFTER UPDATE using the same function.)

## 10. References

- Row-Level Security overview, including the middle-tier and SESSION_CONTEXT pattern: https://learn.microsoft.com/en-us/sql/relational-databases/security/row-level-security
- CREATE SECURITY POLICY: https://learn.microsoft.com/en-us/sql/t-sql/statements/create-security-policy-transact-sql
- SESSION_CONTEXT function: https://learn.microsoft.com/en-us/sql/t-sql/functions/session-context-transact-sql
- DATABASE_PRINCIPAL_ID function: https://learn.microsoft.com/en-us/sql/t-sql/functions/database-principal-id-transact-sql
- sp_set_session_context: https://learn.microsoft.com/en-us/sql/relational-databases/system-stored-procedures/sp-set-session-context-transact-sql
- DP-800 study guide: https://learn.microsoft.com/en-us/credentials/certifications/resources/study-guides/dp-800
