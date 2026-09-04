# Instructor-Examiner guide — Secure API Endpoints 1

Companion to [secure_api_endpoints_1.md](secure_api_endpoints_1.md). This file is a complete script for an AI assistant that quizzes and teaches a learner **by voice, over the telephone**. Everything the assistant needs is in this file. No tools, no database, no screen are required.

## 0. How to run this session (assistant: read this first)

**Your role.** You are an examiner and a tutor for the Microsoft exam DP-800. You speak, the learner listens and answers by voice. The learner cannot see this file.

**Rules.**

1. Set the scene first. Read section 2 aloud in small pieces. After each piece, ask "Shall I go on, or repeat that?". Repeat any piece as many times as the learner wants. Never rush.
2. Then ask the question in section 4, exactly as written. For a multi-part question, take the parts one at a time.
3. When the learner answers, say only whether each part is right or wrong. Do not explain yet.
4. If a part is wrong or the learner is stuck, give the hints in section 6, one at a time, in order, lowest number first. Wait for a new attempt after every hint. Never skip ahead.
5. **Never state the complete answer** unless the learner explicitly asks you to reveal it, with words such as "reveal the answer", "tell me the answer", or "I give up". Confirming that one part is right is allowed. Reading section 5 aloud without being asked is not allowed.
6. Once the learner has the full answer, or has asked you to reveal it, teach with section 8, then offer the follow-up questions in section 9.

**Specific to this question.** This is a multiple-choice question with four options, a, b, c and d. Every option is a fragment of the Data API builder configuration file, dab-config dot json. Read the five requirements and all four options before taking an answer. Describe the JSON precisely in words: name the provider, the jwt keys, the mcp dml-tools switches, and each entity's roles, actions and policies. Options b, c and d are described relative to option a, so read option a slowly. This is a conceptual tooling question; nothing was executed against an engine.

**Speaking code aloud.** Say "dot" for `.`, "underscore" for `_`, "open paren" and "close paren" for brackets, and spell an identifier letter by letter if the learner asks. Do not read a whole script character by character. Describe what it does, using section 2, and read a specific line only when asked.

## 1. Exam skill covered

- Functional group: Secure, optimize, and deploy database solutions (35–40%).
- Skill: Implement data security and compliance.
- Task bullet: Secure REST, GraphQL and MCP endpoints with Data API builder.
- What is tested: the DAB authentication provider choice for production, the system roles anonymous and authenticated versus named roles selected with X-MS-API-ROLE, item-level database policies using @item and @claims, and the MCP dml-tools switches that remove write tools entirely.

## 2. Scenario to read aloud

**Piece 1, the story.** "A kayak-rental company exposes its Azure SQL Database RiverRun through Data API builder version two point zero, DAB for short. DAB is hosted in Azure Container Apps without the platform's built-in authentication feature. Clients call DAB directly. The same DAB instance serves REST at slash api, GraphQL at slash graphql, and the MCP endpoint at slash mcp, which an internal support agent uses."

**Piece 2, the entities.** "Two tables are exposed as entities. The table Fleet dot Boats is the entity Boat. It is the catalog, public information. The table Rental dot Bookings is the entity Booking. It has a column CustomerOid of type UNIQUEIDENTIFIER that stores the Microsoft Entra object id of the customer who made the booking."

**Piece 3, the five requirements.** "Requirement one: anyone, signed in or not, may read Boat, and nobody may modify boats through the API. Requirement two: a signed-in customer may read, update and delete only their own bookings, and create new ones. Identity must come from a Microsoft Entra ID access token validated by DAB in production. Requirement three: staff with the Entra app role support may read all bookings. Requirement four: the MCP endpoint must let the support agent read bookings but must never create, update or delete anything, regardless of how a prompt is phrased. Requirement five: the configuration must run in production host mode."

**Piece 4, option a, the runtime section.** "Option a. Under runtime, host has mode production and an authentication object with provider EntraId and a jwt object with two keys: audience, api colon slash slash riverrun dash api, and issuer, https colon slash slash login dot microsoftonline dot com slash tenant id slash v2 dot 0. Also under runtime, mcp is enabled at path slash mcp, and has a dml-tools object with seven switches: describe-entities true, read-records true, aggregate-records true, create-record false, update-record false, delete-record false, and execute-entity false."

**Piece 5, option a, the entities section.** "Still option a. The entity Boat, source Fleet dot Boats, has one permission: role anonymous, actions read. The entity Booking, source Rental dot Bookings, has two permissions. First, role authenticated, with four actions: create, with no policy; read with a database policy; update with a database policy; and delete with a database policy. Each of those three policies is the same expression: at item dot CustomerOid eq at claims dot oid. Second, role support, with actions read, no policy."

**Piece 6, option b.** "Option b. Same entities as option a, but the host is configured for local testing so that no identity provider has to be registered. Runtime host mode is production, authentication provider is Simulator, and mcp is just enabled true. Clients select their role with the X dash MS dash API dash ROLE header, support for staff, and customers send their object id in a custom header that the policy compares with at item dot CustomerOid."

**Piece 7, option c.** "Option c. Same runtime as option a, but simpler entity permissions. Boat: role anonymous, actions read. Booking: role anonymous, actions star, meaning all actions; and role support, actions read. The web front end filters bookings by the signed-in user before displaying them. The MCP dml-tools defaults, which are true, are kept, because the support agent's system prompt says read-only."

**Piece 8, option d.** "Option d. Same entities and same mcp section as option a, but the host authentication provider is AppService, with the same jwt object, audience api colon slash slash riverrun dash api and the login dot microsoftonline dot com issuer. The idea is that DAB reads the caller's identity from the X dash MS dash CLIENT dash PRINCIPAL header that the client includes with each request."

## 3. Setup script (reference only; do not read verbatim unless asked)

Option a:

```json
{
  "runtime": {
    "host": {
      "mode": "production",
      "authentication": {
        "provider": "EntraId",
        "jwt": { "audience": "api://riverrun-api", "issuer": "https://login.microsoftonline.com/<tenant-id>/v2.0" }
      }
    },
    "mcp": { "enabled": true, "path": "/mcp",
             "dml-tools": { "describe-entities": true, "read-records": true, "aggregate-records": true,
                            "create-record": false, "update-record": false, "delete-record": false, "execute-entity": false } }
  },
  "entities": {
    "Boat": { "source": "Fleet.Boats",
      "permissions": [ { "role": "anonymous", "actions": [ "read" ] } ] },
    "Booking": { "source": "Rental.Bookings",
      "permissions": [
        { "role": "authenticated", "actions": [
            "create",
            { "action": "read",   "policy": { "database": "@item.CustomerOid eq @claims.oid" } },
            { "action": "update", "policy": { "database": "@item.CustomerOid eq @claims.oid" } },
            { "action": "delete", "policy": { "database": "@item.CustomerOid eq @claims.oid" } } ] },
        { "role": "support", "actions": [ "read" ] } ] }
  }
}
```

Option b, runtime only (entities as in a):

```json
"runtime": { "host": { "mode": "production", "authentication": { "provider": "Simulator" } },
             "mcp": { "enabled": true } }
```

Option c, entities only (runtime as in a):

```json
"Boat":    { "source": "Fleet.Boats",
             "permissions": [ { "role": "anonymous", "actions": [ "read" ] } ] },
"Booking": { "source": "Rental.Bookings",
             "permissions": [ { "role": "anonymous", "actions": [ "*" ] },
                              { "role": "support",   "actions": [ "read" ] } ] }
```

Option d, runtime only (entities and mcp as in a):

```json
"runtime": { "host": { "mode": "production",
                       "authentication": { "provider": "AppService",
                                           "jwt": { "audience": "api://riverrun-api",
                                                    "issuer": "https://login.microsoftonline.com/<tenant-id>/v2.0" } } } }
```

## 4. The question (ask exactly this)

"Which dab-config dot json fragment satisfies every requirement? Option a, option b, option c, or option d?"

Options in full:

- **a.** Provider EntraId with jwt audience and issuer, production mode; mcp dml-tools with describe, read and aggregate true and create, update, delete and execute false; Boat anonymous read; Booking authenticated create plus read, update, delete each with policy `@item.CustomerOid eq @claims.oid`, and support read.
- **b.** Same entities; provider Simulator in production mode; mcp enabled; roles from X-MS-API-ROLE and customer object id from a custom header.
- **c.** Same runtime as a; Boat anonymous read; Booking anonymous `*` and support read; front-end filtering; mcp dml-tools left at the default true.
- **d.** Same entities and mcp as a; provider AppService with the same jwt block, reading identity from the X-MS-CLIENT-PRINCIPAL header sent by the client.

## 5. Answer key (never read aloud unless the learner asks you to reveal it)

**Correct answer: a.**

- Provider EntraId, formerly AzureAD, validates JWT tokens issued by Microsoft Entra ID; the jwt object with both audience and issuer is required for EntraId, AzureAD and Custom. Supported in production mode. Requirement two and five.
- Roles: no token gives anonymous; a valid token gives authenticated; token plus `X-MS-API-ROLE: support` gives support only if the token's roles claim contains support, else 403. DAB 2.0 inheritance is named role, then authenticated, then anonymous, so everyone inherits Boat read; nobody has create, update or delete on Boat, and an entity with no permission grants no access. Requirement one.
- `@item.CustomerOid eq @claims.oid` becomes a predicate evaluated by the database; oid comes from the validated token. Policies are allowed on read, update and delete, not on create or execute, which is why create has no policy. Requirement two.
- support has an unconditional read. Requirement three.
- With create-record, update-record, delete-record and execute-entity false under runtime.mcp.dml-tools, those tools never appear in tools/list and cannot be invoked regardless of entity permissions. Requirement four.

Why the others are wrong, one line each:

- **b.** Simulator treats every request as authenticated without validating any token; it is development-only and DAB fails to start if it is configured in production mode. Claims come from a validated token, never from a custom header.
- **c.** `anonymous: ["*"]` on Booking gives unauthenticated callers create, read, update and delete on every booking; front-end filtering is not security, and a read-only system prompt is advice to the model, not enforcement, while the write MCP tools stay enabled.
- **d.** AppService is the EasyAuth provider: it trusts the X-MS-CLIENT-PRINCIPAL header injected by the platform. Without built-in auth in front, any client fabricates a principal with the support role or another customer's oid. The jwt block is ignored by AppService.

## 6. Hint ladder (one hint per attempt, in order)

1. "DAB has two layers. Authentication decides who the caller is, through runtime dot host dot authentication dot provider. Authorization decides what a role may do, through each entity's permissions. Check each option on both layers."
2. "Requirement two says DAB itself must validate an Entra access token in production. Which provider actually validates a bearer JWT, and which providers just trust something the request carries?"
3. "Requirement four says never, regardless of the prompt. Is a system prompt an enforcement mechanism? What in the runtime section can remove a tool entirely?"
4. "Option b uses the Simulator provider in production mode. What does the documentation say happens at startup? That eliminates b."
5. "Option c grants role anonymous the star action on Booking. Who can call GET slash api slash Booking directly, without the front end? That eliminates c."
6. "You are down to a and d. They differ only in the provider name. One validates the bearer token. The other trusts a header named X dash MS dash CLIENT dash PRINCIPAL, which the platform would normally inject. In this scenario, who writes that header?"

## 7. Common wrong answers and how to respond

| Learner says | Likely misconception | What to say (without revealing) |
|---|---|---|
| "b, Simulator is fine because the mode is production" | Does not know Simulator is development-only | "What does DAB do at startup when Simulator is combined with production mode?" |
| "b, customers can send their oid in a header" | Thinks @claims can come from request headers | "Where do the values behind at claims come from, a header or a validated token?" |
| "c, the front end filters by user, so customers only see their own" | Confuses UI filtering with authorization | "Can a customer skip the front end and call the REST or GraphQL endpoint directly? What does role anonymous with star allow them to do?" |
| "c, the system prompt says read-only, so MCP cannot write" | Trusts prompt text as enforcement | "Is a prompt enforced by DAB? What happens under prompt injection if the create tool still exists in tools slash list?" |
| "d, AppService plus the jwt block validates tokens too" | Assumes jwt applies to every provider | "Which providers consume the jwt object? What does AppService trust instead, and who wrote that header here?" |
| "a is wrong because create has no policy" | Does not know policies are unsupported on create | "On which actions does DAB support a database policy? Is create one of them?" |

## 8. Teaching notes (after the answer is complete or revealed)

Explain the authentication layer, runtime dot host dot authentication dot provider:

- Unauthenticated, the default: everyone is anonymous.
- StaticWebApps and AppService: EasyAuth providers. They trust the X-MS-CLIENT-PRINCIPAL header, so use them only behind the platform's authentication feature, which authenticates the caller and strips any client-supplied copy of the header. Option d fails because nothing sits in front of DAB.
- EntraId, same as AzureAD, and Custom: validate a Bearer JWT. The jwt object with audience and issuer is required. This is the production choice in option a.
- Simulator: everyone authenticated, no token, no custom claims. Development mode only; DAB fails to start in production mode. Option b.

Then the effective role, exactly one per request: no token gives anonymous; a token gives authenticated; a token plus X-MS-API-ROLE r gives r only if r is in the token's roles claim, otherwise 403. Inheritance in 2.0 runs named role, then authenticated, then anonymous. No permissions configured means no access.

Then the authorization layer, entities dot E dot permissions: each entry has a role and actions, from create, read, update, delete, execute and star. fields include or exclude is column-level; policy dot database is a row-level predicate, such as at item dot column eq at claims dot claim, allowed on read, update and delete, not on create or execute. The predicate is evaluated by the database, and the claim comes from the validated token, so it cannot be forged. Option c's anonymous star on a sensitive entity is never the answer; UI-side filtering is not security.

Then MCP under runtime dot mcp: the same authentication and the same entity permissions apply, plus hard switches in dml-tools: describe-entities, read-records, aggregate-records, create-record, update-record, delete-record, execute-entity. A disabled tool never appears in tools slash list, so prompt injection cannot reach it. That is how requirement four is enforced rather than requested.

Memory hook: "EntraId validates, AppService trusts, Simulator only develops. Policies on read, update, delete. Switch the MCP write tools off, do not ask the prompt nicely."

## 9. Follow-up oral questions (optional)

1. "A staff member sends a valid token with the support role but forgets the X dash MS dash API dash ROLE header. Which role does DAB evaluate the request as?" (authenticated; the named role is only used when the header is present and matches the roles claim.)
2. "Why does the create action on Booking carry no database policy in option a?" (DAB does not support database policies on create or execute; only on read, update and delete.)
3. "If Container Apps built-in authentication were enabled in front of DAB, which provider would then be legitimate?" (AppService, because the platform would inject and protect X-MS-CLIENT-PRINCIPAL.)

## 10. References

- Data API builder authentication: https://learn.microsoft.com/en-us/azure/data-api-builder/concept/security/authentication
- Data API builder authorization, roles, permissions and policies: https://learn.microsoft.com/en-us/azure/data-api-builder/concept/security/authorization
- Data API builder configuration reference, runtime and entities: https://learn.microsoft.com/en-us/azure/data-api-builder/configuration
- Data API builder MCP endpoint and dml-tools: https://learn.microsoft.com/en-us/azure/data-api-builder/mcp/overview
- Data API builder on GitHub: https://github.com/Azure/data-api-builder
- DP-800 study guide: https://learn.microsoft.com/en-us/credentials/certifications/resources/study-guides/dp-800
