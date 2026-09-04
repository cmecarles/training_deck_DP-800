# Instructor-Examiner guide — Azure SQL Data API builder on Container Apps 1

Companion to [azure_sql_dab_container_app_1.md](azure_sql_dab_container_app_1.md). This file is a complete script for an AI assistant that quizzes and teaches a learner **by voice, over the telephone**. Everything the assistant needs is in this file. No tools, no database, no screen are required.

## 0. How to run this session (assistant: read this first)

**Your role.** You are an examiner and a tutor for the Microsoft exam DP-800. You speak, the learner listens and answers by voice. The learner cannot see this file.

**Rules.**

1. Set the scene first. Read section 2 aloud in small pieces. After each piece, ask "Shall I go on, or repeat that?". Repeat any piece as many times as the learner wants. Never rush.
2. Then ask the question in section 4, exactly as written. For a multi-part question, take the parts one at a time.
3. When the learner answers, say only whether each part is right or wrong. Do not explain yet.
4. If a part is wrong or the learner is stuck, give the hints in section 6, one at a time, in order, lowest number first. Wait for a new attempt after every hint. Never skip ahead.
5. **Never state the complete answer** unless the learner explicitly asks you to reveal it, with words such as "reveal the answer", "tell me the answer", or "I give up". Confirming that one part is right is allowed. Reading section 5 aloud without being asked is not allowed.
6. Once the learner has the full answer, or has asked you to reveal it, teach with section 8, then offer the follow-up questions in section 9.
7. **This is a hands-on Azure lab question.** Before reading the scenario, ask: "Have you already run this lab in your Azure subscription?" If **yes**, do not quiz right away. Go through the requests R1 to R7 and ask what status code the learner observed for each, and whether the three lots came back. Also ask what they did in the Entra admin center portal steps. Then ask the question. If **no**, read section 2 in full, including the CLI, Dockerfile and portal pieces, so the question can be answered from the documented facts alone. The learner does not need Azure to answer.
8. **This is a multiple-choice question.** Read all four options, pieces 12 to 15, before taking an answer. Take one letter as the answer. Do not accept "a or c".

**Speaking code aloud.** Say "dot" for `.`, "underscore" for `_`, "open paren" and "close paren" for brackets, and spell an identifier letter by letter if the learner asks. Do not read a whole script character by character. Describe what it does, using section 2, and read a specific line only when asked.

## 1. Exam skill covered

- Functional group: Design and develop database solutions (35–40%).
- Skill: Implement data access with Data API builder.
- Task bullet: Deploy and secure Data API builder with Microsoft Entra authentication and managed identity.
- What is tested: how DAB decides the effective role from the bearer token and the X-MS-API-ROLE header, when a request is 401 versus 403, and how the health endpoint behaves in production mode.

## 2. Scenario to read aloud

**Piece 1, the story.** "An apple orchard tracks harvest lots in an Azure SQL Database called OrchardStock. They want the lots published through Data API builder, D A B, hosted on Azure Container Apps. Three constraints. First, DAB must reach SQL with a user-assigned managed identity, so there is no connection-string secret with a password. Second, callers must present a Microsoft Entra token. Third, only users holding the app role reader may use the reader role. This is a hands-on lab: you provision, deploy, send requests R1 to R7, and predict their outcomes before you run them."

**Piece 2, provisioning with the Azure CLI.** "A bash script uses the Azure CLI in westeurope with a random suffix. It names a resource group rg dash dp800 dash orchard, a logical server sql dash orchard, a database OrchardStock, a user-assigned identity id dash orchard dash dab, a container registry acrorchard, a Container Apps environment cae dash orchard, and a container app ca dash orchard dash dab. It reads your user principal name, object id, tenant id and public IP. It creates the resource group and the user-assigned identity, and reads the identity's resource id, client id and principal id. It creates the SQL server with Entra-only authentication and you as external admin. Two firewall rules: one for your IP, and one called AllowAzureServices from 0 dot 0 dot 0 dot 0 to 0 dot 0 dot 0 dot 0, which lets Azure services in. Then the database, General Purpose, serverless, one vCore, auto-pause sixty minutes. It reads the server's fully qualified domain name."

**Piece 3, the schema and the database user.** "You connect with sqlcmd using Entra authentication. You create a schema Orchard and a table Orchard dot Lots with four columns: LotId, an integer primary key. Variety, NVARCHAR thirty. Crates, an integer. PickedOn, a date. Three rows: lot 1, Bramley, forty crates, picked first of September 2026. Lot 2, Cox, twenty-five crates, second of September. Lot 3, Egremont Russet, twelve crates, third of September. Then you create a database user named id dash orchard dash dab, in square brackets, FROM EXTERNAL PROVIDER, and add it to db underscore datareader."

**Piece 4, the Entra app registration.** "Back in bash, you create an app registration named orchard dash dab dash api with sign-in audience AzureADMyOrg, single tenant, and capture its app id. You write a roles dot json file containing one app role: allowed member types User, display name Reader, enabled, and value reader, lowercase. You update the app with an identifier URI of api colon slash slash the app id, requested access token version 2, and the app roles from the JSON file. Then you create the service principal for the app, which is the enterprise application, needed for role assignment."

**Piece 5, three portal steps in the Microsoft Entra admin center.** "Step one: App registrations, orchard dash dab dash api, Expose an API, Add a scope. Name Endpoint dot Access, who can consent: Admins and users, any display texts, state Enabled. Step two, same blade: Authorized client applications, Add a client application. Client ID 04b07795 dash 8ddb dash 461a dash bbee dash 02f9e1bf7b46, which is the Microsoft Azure CLI, and tick the scope api colon slash slash app id slash Endpoint dot Access. Step three: Enterprise applications, orchard dash dab dash api, Users and groups, Add user or group. Pick your own user and the role Reader."

**Piece 6, the Dockerfile.** "A Dockerfile with two stages. The build stage starts from the dotnet SDK 8 image. It takes two build arguments, APP underscore ID and TENANT underscore ID. In a folder slash config it creates a tool manifest and installs the Microsoft dot DataApiBuilder tool. Then four dab commands. Dab init with database type mssql, the connection string set to at env open paren DATABASE underscore CONNECTION underscore STRING close paren, and host mode production. Dab add an entity called Lot from source Orchard dot Lots with permissions authenticated colon read. Dab update Lot with permissions reader colon read. And dab configure, setting runtime dot host dot authentication dot provider to EntraID, the JWT audience to api colon slash slash the app id, the JWT issuer to https colon slash slash login dot microsoftonline dot com slash the tenant id slash v2 dot 0, and runtime dot health dot roles to anonymous. The final stage starts from the official image mcr dot microsoft dot com slash azure dash databases slash data dash api dash builder, and copies slash config into slash App."

**Piece 7, registry and image.** "The script creates a Basic container registry with the admin account disabled. It runs az acr build with tag orchard dash dab colon v1, passing the app id and tenant id as build arguments. It reads the registry login server and resource id, and assigns the role AcrPull to the user-assigned identity, scoped to the registry."

**Piece 8, the container app.** "The connection string is built as: Server tcp colon the SQL FQDN comma 1433, Initial Catalog OrchardStock, Authentication equals Active Directory Managed Identity, User Id equals the identity's client id, Encrypt True. There is no password. It creates a Container Apps environment with logs destination none. Then az containerapp create with the image from the registry, registry identity and user-assigned identity both set to the same identity, external ingress on target port 5000, one replica, half a vCPU, one gigabyte. The connection string is stored as a Container Apps secret called conn, and the environment variable DATABASE underscore CONNECTION underscore STRING is set to secretref colon conn. Finally it reads the app's public FQDN."

**Piece 9, the token and requests R1 to R3.** "You get an access token with az account get-access-token for the scope api colon slash slash app id slash Endpoint dot Access, and put it in an Authorization Bearer header. R1 is a GET to slash api slash Lot with no token at all. R2 is a GET to slash api slash Lot with the bearer token and nothing else. R3 is the same GET with the bearer token plus a header X dash MS dash API dash ROLE set to reader."

**Piece 10, requests R4 to R7.** "R4 is the same GET with the token plus X dash MS dash API dash ROLE set to packer, a role that was never defined anywhere. R5 is a POST to slash graphql with the token, content type JSON, and the query: lots, items, LotId, Variety, Crates. R6 is a GET to slash api slash Lot with the header Authorization Bearer not dash a dash token, a made-up string. R7 is a GET to slash health with no token."

**Piece 11, cost and cleanup.** "For the record: Container Apps at half a vCPU, a Basic registry at about fifteen cents a day, and the serverless database add up to well under one euro for the lab. Cleanup deletes the resource group and the app registration."

**Piece 12, option a.** "Option a says: R2 200 with the three lots. R3 200 with the three lots. R4 403. R5 200 with the three lots under data dot lots dot items. R6 401. R7 200."

**Piece 13, option b.** "Option b says: R2 403, because there is no X dash MS dash API dash ROLE header so no role is selected. R3 200. R4 403. R5 200. R6 401. R7 200."

**Piece 14, option c.** "Option c says: R2 200. R3 200. R4 200, because the header is ignored when the role is not in the token and DAB falls back to authenticated. R5 200. R6 401. R7 200."

**Piece 15, option d.** "Option d says: R2 200. R3 200. R4 403. R5 200. R6 403. R7 403, because production mode never exposes slash health anonymously."

## 3. Setup script (reference only; do not read verbatim unless asked)

```bash
LOCATION="westeurope"
SUFFIX=$RANDOM
RG="rg-dp800-orchard-$SUFFIX"
SQL="sql-orchard-$SUFFIX"; DB="OrchardStock"
UAMI="id-orchard-dab"; ACR="acrorchard$SUFFIX"; CAE="cae-orchard"; APP="ca-orchard-dab"
ADMIN_UPN=$(az ad signed-in-user show --query userPrincipalName -o tsv)
ADMIN_OID=$(az ad signed-in-user show --query id -o tsv)
TENANT_ID=$(az account show --query tenantId -o tsv)
MYIP=$(curl -s https://api.ipify.org)

az group create -n $RG -l $LOCATION
az identity create -g $RG -n $UAMI
UAMI_ID=$(az identity show -g $RG -n $UAMI --query id -o tsv)
UAMI_CLIENT_ID=$(az identity show -g $RG -n $UAMI --query clientId -o tsv)
UAMI_PID=$(az identity show -g $RG -n $UAMI --query principalId -o tsv)

az sql server create -g $RG -n $SQL -l $LOCATION --enable-ad-only-auth \
  --external-admin-principal-type User --external-admin-name "$ADMIN_UPN" --external-admin-sid $ADMIN_OID
az sql server firewall-rule create -g $RG -s $SQL -n client --start-ip-address $MYIP --end-ip-address $MYIP
az sql server firewall-rule create -g $RG -s $SQL -n AllowAzureServices --start-ip-address 0.0.0.0 --end-ip-address 0.0.0.0
az sql db create -g $RG -s $SQL -n $DB -e GeneralPurpose -f Gen5 -c 1 --compute-model Serverless \
  --auto-pause-delay 60 --min-capacity 0.5 --backup-storage-redundancy Local
SQL_FQDN=$(az sql server show -g $RG -n $SQL --query fullyQualifiedDomainName -o tsv)
```

```sql
CREATE SCHEMA Orchard;
GO
CREATE TABLE Orchard.Lots (LotId INT NOT NULL PRIMARY KEY, Variety NVARCHAR(30) NOT NULL, Crates INT NOT NULL, PickedOn DATE NOT NULL);
INSERT INTO Orchard.Lots VALUES (1, N'Bramley', 40, '20260901'), (2, N'Cox', 25, '20260902'), (3, N'Egremont Russet', 12, '20260903');
GO
CREATE USER [id-orchard-dab] FROM EXTERNAL PROVIDER;
ALTER ROLE db_datareader ADD MEMBER [id-orchard-dab];
GO
```

```bash
APP_ID=$(az ad app create --display-name "orchard-dab-api" --sign-in-audience AzureADMyOrg --query appId -o tsv)
cat > roles.json <<'EOF'
[{"allowedMemberTypes":["User"],"description":"Read harvest lots","displayName":"Reader","isEnabled":"true","value":"reader"}]
EOF
az ad app update --id $APP_ID --identifier-uris "api://$APP_ID" --requested-access-token-version 2 --app-roles @roles.json
az ad sp create --id $APP_ID          # the enterprise application, needed for role assignment
echo "APP_ID=$APP_ID  TENANT_ID=$TENANT_ID"
```

Portal steps (Microsoft Entra admin center):

1. App registrations > orchard-dab-api > Expose an API > Add a scope: name `Endpoint.Access`, Admins and users, state Enabled.
2. Same blade > Authorized client applications > Add a client application: client ID `04b07795-8ddb-461a-bbee-02f9e1bf7b46` (Microsoft Azure CLI), tick `api://<APP_ID>/Endpoint.Access`.
3. Enterprise applications > orchard-dab-api > Users and groups > Add user/group: your user, role Reader.

```bash
cat > Dockerfile <<'EOF'
FROM mcr.microsoft.com/dotnet/sdk:8.0-cbl-mariner2.0 AS build
ARG APP_ID
ARG TENANT_ID
WORKDIR /config
RUN dotnet new tool-manifest && dotnet tool install Microsoft.DataApiBuilder
RUN dotnet tool run dab -- init --database-type mssql --connection-string "@env('DATABASE_CONNECTION_STRING')" --host-mode production
RUN dotnet tool run dab -- add Lot --source "Orchard.Lots" --permissions "authenticated:read"
RUN dotnet tool run dab -- update Lot --permissions "reader:read"
RUN dotnet tool run dab -- configure --runtime.host.authentication.provider EntraID \
      --runtime.host.authentication.jwt.audience "api://${APP_ID}" \
      --runtime.host.authentication.jwt.issuer "https://login.microsoftonline.com/${TENANT_ID}/v2.0" \
      --runtime.health.roles anonymous
FROM mcr.microsoft.com/azure-databases/data-api-builder:latest
COPY --from=build /config /App
EOF
az acr create -g $RG -n $ACR --sku Basic --admin-enabled false
az acr build -r $ACR -t orchard-dab:v1 -f Dockerfile --build-arg APP_ID=$APP_ID --build-arg TENANT_ID=$TENANT_ID .
ACR_SERVER=$(az acr show -g $RG -n $ACR --query loginServer -o tsv)
ACR_ID=$(az acr show -g $RG -n $ACR --query id -o tsv)
az role assignment create --assignee-object-id $UAMI_PID --assignee-principal-type ServicePrincipal --role AcrPull --scope $ACR_ID

CONN="Server=tcp:$SQL_FQDN,1433;Initial Catalog=$DB;Authentication=Active Directory Managed Identity;User Id=$UAMI_CLIENT_ID;Encrypt=True;"
az containerapp env create -g $RG -n $CAE -l $LOCATION --logs-destination none
az containerapp create -g $RG -n $APP --environment $CAE --image "$ACR_SERVER/orchard-dab:v1" \
  --registry-server $ACR_SERVER --registry-identity $UAMI_ID --user-assigned $UAMI_ID \
  --ingress external --target-port 5000 --min-replicas 1 --max-replicas 1 --cpu 0.5 --memory 1.0Gi \
  --secrets conn="$CONN" --env-vars DATABASE_CONNECTION_STRING=secretref:conn
FQDN=$(az containerapp show -g $RG -n $APP --query properties.configuration.ingress.fqdn -o tsv)
```

```bash
TOKEN=$(az account get-access-token --scope "api://$APP_ID/Endpoint.Access" --query accessToken -o tsv)
H="Authorization: Bearer $TOKEN"
# R1
curl -s -o /dev/null -w "%{http_code}\n" "https://$FQDN/api/Lot"
# R2
curl -s -w "\n%{http_code}\n" -H "$H" "https://$FQDN/api/Lot"
# R3
curl -s -w "\n%{http_code}\n" -H "$H" -H "X-MS-API-ROLE: reader" "https://$FQDN/api/Lot"
# R4
curl -s -w "\n%{http_code}\n" -H "$H" -H "X-MS-API-ROLE: packer" "https://$FQDN/api/Lot"
# R5
curl -s -w "\n%{http_code}\n" -H "$H" -H "Content-Type: application/json" -X POST "https://$FQDN/graphql" \
  -d '{"query":"{ lots { items { LotId Variety Crates } } }"}'
# R6
curl -s -w "\n%{http_code}\n" -H "Authorization: Bearer not-a-token" "https://$FQDN/api/Lot"
# R7
curl -s -o /dev/null -w "%{http_code}\n" "https://$FQDN/health"
```

Cleanup:

```bash
az group delete -n $RG --yes --no-wait
az ad app delete --id $APP_ID
```

## 4. The question (ask exactly this)

"Which row set describes R2 to R7 correctly, giving the HTTP status and whether the three lots are returned? Option a, b, c or d?"

- **a.** R2 200 with the 3 lots · R3 200 with the 3 lots · R4 **403** · R5 200 with the 3 lots under `data.lots.items` · R6 **401** · R7 200
- **b.** R2 **403** (no `X-MS-API-ROLE`, so no role is selected) · R3 200 · R4 403 · R5 200 · R6 401 · R7 200
- **c.** R2 200 · R3 200 · R4 **200** (the header is ignored when the role is not in the token, DAB falls back to `authenticated`) · R5 200 · R6 401 · R7 200
- **d.** R2 200 · R3 200 · R4 403 · R5 200 · R6 **403** · R7 **403** (production mode never exposes `/health` anonymously)

## 5. Answer key (never read aloud unless the learner asks you to reveal it)

**Correct answer: a.**

| Request | Status | Why |
|---|---|---|
| R1 | Rejected (status not asked) | No token, so role Anonymous, which has no permission on Lot. |
| R2 | 200, 3 lots | Valid token, no header, so system role Authenticated; authenticated:read grants read. |
| R3 | 200, 3 lots | Header reader; the token's roles claim contains reader thanks to the Enterprise-application assignment; reader:read. |
| R4 | 403 | Header packer is not in the token's roles claim; rejected before permissions are consulted. |
| R5 | 200, 3 lots in data.lots.items | GraphQL uses the same authentication and permission model; entity Lot is query field lots with items. |
| R6 | 401 | Malformed token fails validation: authentication failure, not authorization. |
| R7 | 200 | runtime.health.roles includes anonymous, so health is displayed in production mode. |

Why the wrong options are wrong:

- **b.** A valid token without X-MS-API-ROLE is not "no role". DAB assigns the system role Authenticated, and the entity grants authenticated:read. The header is needed only to use a role other than Anonymous or Authenticated.
- **c.** DAB evaluates exactly one effective role per request. When the header names a role the token does not carry, the request is rejected with 403. There is no silent downgrade to authenticated.
- **d.** An invalid token is 401, not 403. And /health in production is 403 only when runtime.health.roles is omitted or does not include the current role; listing anonymous makes it public.

## 6. Hint ladder (one hint per attempt, in order)

1. "Start with R2. The token is valid and there is no role header. Does DAB really leave such a request with no role, or does it assign a built-in system role? Recall the two system roles DAB always has."
2. "Now R4. The header says packer, but the token's roles claim only says reader. Does DAB quietly ignore a header it cannot verify, or does it refuse the request? Think about what the header is for: choosing a role you can prove."
3. "R6 sends a token that is not a token at all. Is that a problem of who you are, authentication, or of what you may do, authorization? Which status code goes with each?"
4. "R7 hits slash health with no token. The Dockerfile set runtime dot health dot roles to a specific value. What was that value, and what does it allow?"
5. "You can now drop the option that gives R2 a 403, and the option that gives R6 and R7 a 403. Two options remain, and they differ only on R4."
6. "For R4, the documentation says a request is rejected when the token's roles claim does not contain the role in X dash MS dash API dash ROLE. Is that a 200 or a 403?"

## 7. Common wrong answers and how to respond

| Learner says | Likely misconception | What to say (without revealing) |
|---|---|---|
| "b, you must always send X-MS-API-ROLE" | Does not know the Authenticated system role | "What role does DAB give to a request with a valid token and no header? There is a name for it." |
| "c, the bad header is ignored and you still get authenticated" | Believes in a fallback | "DAB picks one effective role. If the header names a role you cannot prove, does it pick another one for you?" |
| "d, a bad token is forbidden, 403" | Confuses 401 and 403 | "Which code means 'I cannot tell who you are' and which means 'I know who you are and you may not'?" |
| "d, health is never public in production" | Forgets runtime.health.roles | "Look back at the last dab configure option in the Dockerfile. What did it set?" |
| "R5 must fail because GraphQL needs a different token" | Thinks REST and GraphQL are secured differently | "Does DAB have one authentication and permission model for both endpoints, or two?" |
| "R3 fails because the role was never assigned" | Missed portal step three | "Which portal step gave your user the Reader app role? What claim does that add to the token?" |
| "R2 fails because the identity only has db_datareader" | Confuses the caller's identity with DAB's SQL identity | "Whose identity does DAB use to talk to SQL, the caller's or its own? Is read enough for a GET?" |

## 8. Teaching notes (after the answer is complete or revealed)

Explain the two documented decision tables:

- **Token validation for the EntraID provider.** DAB checks the audience, which must equal jwt dot audience, api colon slash slash app id. It checks the issuer, which must equal jwt dot issuer, the v2 issuer https colon slash slash login dot microsoftonline dot com slash tenant slash v2 dot 0. That is why the app registration was updated with requested access token version 2. A v1 token carries the sts dot windows dot net issuer and every call would be 401. It checks expiry and the signature. Failure here is 401.
- **The role matrix.** No token: Anonymous. Valid token and no header: Authenticated. Valid token and a header whose role is not in the token's roles claim: 403. Valid token and a header whose role is in the claim: that role. Invalid token: 401. That single table settles R2, R3, R4 and R6.
- **Permissions.** The entity Lot grants read to authenticated and to reader; nothing to anonymous. That is why R1 is rejected and R2, R3 and R5 succeed. GraphQL and REST share the same model; the entity Lot appears as the query field lots with an items array.
- **The health endpoint.** In production mode runtime dot health dot roles is required. If it includes anonymous, the health page is displayed to anyone, which is the documented pattern for an unauthenticated ingress probe. If it is omitted, production mode returns 403.
- **What the requests do not show.** The container reached SQL as id dash orchard dash dab because the connection string uses Authentication equals Active Directory Managed Identity with the user-assigned identity's client id, that identity was attached with user-assigned, and it exists in the database as a user from external provider in db underscore datareader. The same identity pulled the image with AcrPull. DAB connects to the database with its own identity, separate from the caller; the caller's token is never forwarded to SQL. The connection string lives in a Container Apps secret referenced with secretref colon conn, and the config file only holds at env DATABASE underscore CONNECTION underscore STRING.

Memory hook: "No token, Anonymous. Token, Authenticated. Header must be in the token's roles or it is 403. Bad token is 401. Health needs roles in production."

## 9. Follow-up oral questions (optional)

1. "If step three in the portal, assigning your user the Reader role, had been skipped, what would R3 return?" (403. The header reader would not be in the token's roles claim.)
2. "If the app registration had kept access token version 1, what would R2 return?" (401. The issuer would be sts dot windows dot net, which does not match the configured v2 issuer.)
3. "Does DAB use the caller's Entra token to connect to SQL?" (No. It uses its own identity, here the user-assigned managed identity. Forwarding the caller's identity needs the separate On-Behalf-Of feature.)

## 10. References

- Data API builder: Microsoft Entra ID authentication: https://learn.microsoft.com/en-us/azure/data-api-builder/authentication-entra
- Data API builder: authorization and roles (X-MS-API-ROLE): https://learn.microsoft.com/en-us/azure/data-api-builder/authorization
- Data API builder: configuration reference (runtime.host.authentication, runtime.health): https://learn.microsoft.com/en-us/azure/data-api-builder/configuration/runtime
- Data API builder: health checks: https://learn.microsoft.com/en-us/azure/data-api-builder/health-checks
- Data API builder: deploy to Azure Container Apps: https://learn.microsoft.com/en-us/azure/data-api-builder/deployment/how-to-host-azure-container-apps
- Data API builder: run in a container: https://learn.microsoft.com/en-us/azure/data-api-builder/deployment/how-to-run-container
- Azure SQL: managed identities and CREATE USER FROM EXTERNAL PROVIDER: https://learn.microsoft.com/en-us/azure/azure-sql/database/authentication-aad-configure
- Azure Container Apps: managed identities: https://learn.microsoft.com/en-us/azure/container-apps/managed-identity
- DP-800 study guide: https://learn.microsoft.com/en-us/credentials/certifications/resources/study-guides/dp-800
