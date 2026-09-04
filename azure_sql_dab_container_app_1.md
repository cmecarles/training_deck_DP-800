# SQL Server question — Azure SQL Data API builder on Container Apps 1

## Statement

An apple orchard tracks harvest lots in an **Azure SQL Database** named `OrchardStock` and wants them published through **Data API builder (DAB)** hosted on **Azure Container Apps**. Constraints: DAB must reach SQL with a **user-assigned managed identity** (no connection-string secret), callers must present a **Microsoft Entra** token, and only users holding the app role `reader` may use the `reader` role.

This is a **hands-on** question: provision, deploy, send requests R1–R7, and predict their outcomes before you run them.

### Provisioning (Azure CLI, bash; `az login` first)

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

Schema and the identity's database user (`sqlcmd -S $SQL_FQDN -d $DB --authentication-method ActiveDirectoryDefault`, or `-G -U "$ADMIN_UPN"` with the ODBC sqlcmd):

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

### The Entra app registration that represents the API

```bash
APP_ID=$(az ad app create --display-name "orchard-dab-api" --sign-in-audience AzureADMyOrg --query appId -o tsv)
cat > roles.json <<'EOF'
[{"allowedMemberTypes":["User"],"description":"Read harvest lots","displayName":"Reader","isEnabled":"true","value":"reader"}]
EOF
az ad app update --id $APP_ID --identifier-uris "api://$APP_ID" --requested-access-token-version 2 --app-roles @roles.json
az ad sp create --id $APP_ID          # the enterprise application, needed for role assignment
echo "APP_ID=$APP_ID  TENANT_ID=$TENANT_ID"
```

Three steps in the Microsoft Entra admin center (portal), exactly as the DAB documentation describes them:

1. **App registrations › orchard-dab-api › Expose an API › Add a scope**: name `Endpoint.Access`, *Admins and users*, any display texts, state *Enabled*.
2. Same blade, **Authorized client applications › Add a client application**: client ID `04b07795-8ddb-461a-bbee-02f9e1bf7b46` (Microsoft Azure CLI), tick the `api://<APP_ID>/Endpoint.Access` scope.
3. **Enterprise applications › orchard-dab-api › Users and groups › Add user/group**: your user, role **Reader**.

### Build the DAB image and deploy it

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

### The requests

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

Which row set describes R2–R7 correctly (HTTP status; whether the three lots are returned)?

### a.

R2 200 with the 3 lots · R3 200 with the 3 lots · R4 **403** · R5 200 with the 3 lots under `data.lots.items` · R6 **401** · R7 200

### b.

R2 **403** (no `X-MS-API-ROLE`, so no role is selected) · R3 200 · R4 403 · R5 200 · R6 401 · R7 200

### c.

R2 200 · R3 200 · R4 **200** (the header is ignored when the role is not in the token, DAB falls back to `authenticated`) · R5 200 · R6 401 · R7 200

### d.

R2 200 · R3 200 · R4 403 · R5 200 · R6 **403** · R7 **403** (production mode never exposes `/health` anonymously)

**Cost and cleanup.** Container Apps (0.5 vCPU, one replica), a Basic registry (~€0.15/day) and the serverless database add up to well under one euro for the lab. Delete the resource group and the app registration:

```bash
az group delete -n $RG --yes --no-wait
az ad app delete --id $APP_ID
```

## Correct Answer

**a**

## Explanation

### Why option a is correct

Every request is decided by two documented tables. Token validation for the `EntraID` provider checks `aud` (must equal `jwt.audience`, `api://<APP_ID>`), `iss` (must equal `jwt.issuer`, the **v2.0** issuer `https://login.microsoftonline.com/<tenant>/v2.0` — which is why the registration was updated with `--requested-access-token-version 2`; a v1 token carries `https://sts.windows.net/<tenant>/` and every call would be 401), `exp` and the signature. Then the role matrix:

| Bearer token | `X-MS-API-ROLE` | role in token `roles` claim | effective role |
|---|---|---|---|
| none | — | — | `Anonymous` |
| valid | none | — | `Authenticated` |
| valid | present | no | **403 Forbidden** |
| valid | present | yes | the header value |
| invalid | any | — | **401 Unauthorized** |

- **R1** — no token: `Anonymous`, which has no permission on `Lot` (permissions were configured only for `authenticated` and `reader`), so the request is rejected. (Its exact status is not part of the question.)
- **R2** — valid token, no header → `Authenticated`; `authenticated:read` grants `read` → **200** with `{"value":[...3 lots...]}`.
- **R3** — header `reader`; your token carries `roles: ["reader"]` because of the Enterprise-application role assignment; `reader:read` → **200**.
- **R4** — header `packer` is not in the token → rejected with **403** *before* permissions are even consulted ("A client app's request is rejected when the supplied access token's `roles` claim doesn't contain the role listed in the `X-MS-API-ROLE` header"). Without the role assignment in step 3 of the portal work, R3 would fail the same way.
- **R5** — GraphQL shares the same authentication and permission model; the entity `Lot` is exposed as the query field `lots` with `items`, so the POST returns **200** and `data.lots.items` holds the three rows.
- **R6** — a malformed token fails validation → **401** (the troubleshooting table: `401 Unauthorized` — token expired or malformed / audience mismatch / issuer mismatch).
- **R7** — `runtime.health.roles` was set to `["anonymous"]`; in production mode `roles` is *required* and "roles includes anonymous → Health displayed", hence **200**. Had it been omitted, production mode returns 403.

Two things the request phase does not show but the deployment relied on: the container reached SQL as **`id-orchard-dab`** because the connection string `Authentication=Active Directory Managed Identity;User Id=<client id>` names the user-assigned identity attached with `--user-assigned`, and that identity is a `db_datareader` created with `CREATE USER ... FROM EXTERNAL PROVIDER`; the same identity pulled the image (`--registry-identity`, AcrPull). DAB "connects to the database using its own identity, separate from the authenticated user" — the caller's token is never forwarded to SQL (that would require the separate On-Behalf-Of feature). The connection string is a Container Apps **secret** referenced with `secretref:conn`, and the config only holds `@env('DATABASE_CONNECTION_STRING')`.

### Why option b is wrong

A valid token without `X-MS-API-ROLE` is not "no role": DAB assigns the system role `Authenticated`, and the entity grants `authenticated:read`. The header is required only "to use a role other than `Anonymous` or `Authenticated`".

### Why option c is wrong

This is the subtle distractor. DAB evaluates exactly one effective role per request; when the header names a role the token does not carry, the request is **rejected (403)** — it does not silently downgrade to `authenticated`. (Role *inheritance* in DAB 2.0 — `named-role → authenticated → anonymous` — is about permission blocks you did not configure, not about headers you cannot prove.)

### Why option d is wrong

An invalid token is an authentication failure (**401**), not an authorization failure (403). And `/health` in production is 403 only when `runtime.health.roles` is omitted or does not contain the current role; listing `anonymous` makes it public, which is the documented pattern for an unauthenticated ingress probe.

Hands-on question (Azure subscription required); the T-SQL fragments that do not depend on Azure were checked on SQL Server 2025 RTM 17.0.1000.7; Azure-side behaviour is taken from the official documentation.

## DP-800 Exam Rule to Remember

```text
DAB on Azure Container Apps (documented path): Dockerfile = dotnet sdk stage (dab init/add/update/configure)
   -> COPY /config into mcr.microsoft.com/azure-databases/data-api-builder (reads /App/dab-config.json, port 5000)
   az acr build -> az containerapp create --registry-identity <UAMI> --user-assigned <UAMI>
                   --secrets conn=... --env-vars DATABASE_CONNECTION_STRING=secretref:conn
SQL side: "Authentication=Active Directory Managed Identity;User Id=<UAMI client id>" + CREATE USER [<uami>] FROM EXTERNAL PROVIDER
Caller side (runtime.host.authentication): provider EntraID, jwt.audience api://<app-id>,
   jwt.issuer https://login.microsoftonline.com/<tenant>/v2.0  (app manifest accessTokenAcceptedVersion = 2)
   token: az account get-access-token --scope api://<app-id>/Endpoint.Access  (Azure CLI must be an authorized client app)
Roles: no token -> Anonymous | token -> Authenticated | X-MS-API-ROLE must be in the token's roles claim, else 403
       bad/expired/wrong-aud/wrong-iss token -> 401 | app roles are assigned in Enterprise applications, not granted by existing
/health in production needs runtime.health.roles (include anonymous for probes); DAB never passes the caller's token to SQL
```
