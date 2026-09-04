# SQL Server question — Azure SQL Entra Managed Identity 1

## Statement

A marina publishes its berth availability through a small worker that runs on demand in Azure and reads an **Azure SQL Database** named `BerthBook`. The security baseline is: *no SQL logins can ever connect, the worker authenticates as a user-assigned managed identity, and nothing that looks like a password exists anywhere*.

This is a **hands-on** question: you provision the resources in your own subscription, run the steps R1–R7 below, and predict what each one does before you run it.

### Provisioning (Azure CLI, bash; `az login` first)

```bash
LOCATION="westeurope"                       # any region with Azure SQL and Container Instances
SUFFIX=$RANDOM
RG="rg-dp800-berthbook-$SUFFIX"
SQL="sql-berthbook-$SUFFIX"                  # globally unique, lowercase
DB="BerthBook"
UAMI="id-berthbook-worker"
ADMIN_UPN=$(az ad signed-in-user show --query userPrincipalName -o tsv)
ADMIN_OID=$(az ad signed-in-user show --query id -o tsv)
MYIP=$(curl -s https://api.ipify.org)

az group create -n $RG -l $LOCATION
# Logical server with NO SQL admin: Entra admin = you, Entra-only authentication ON from birth
az sql server create -g $RG -n $SQL -l $LOCATION --enable-ad-only-auth \
  --external-admin-principal-type User --external-admin-name "$ADMIN_UPN" --external-admin-sid $ADMIN_OID
az sql server firewall-rule create -g $RG -s $SQL -n client --start-ip-address $MYIP --end-ip-address $MYIP
az sql server firewall-rule create -g $RG -s $SQL -n AllowAzureServices --start-ip-address 0.0.0.0 --end-ip-address 0.0.0.0
# Cheapest vCore option: serverless GP Gen5 1 vCore, auto-pause after 60 idle minutes
az sql db create -g $RG -s $SQL -n $DB -e GeneralPurpose -f Gen5 -c 1 --compute-model Serverless \
  --auto-pause-delay 60 --min-capacity 0.5 --backup-storage-redundancy Local
SQL_FQDN=$(az sql server show -g $RG -n $SQL --query fullyQualifiedDomainName -o tsv)

# User-assigned managed identity for the worker
az identity create -g $RG -n $UAMI
UAMI_ID=$(az identity show -g $RG -n $UAMI --query id -o tsv)
UAMI_CLIENT_ID=$(az identity show -g $RG -n $UAMI --query clientId -o tsv)
```

Connect to the database with **sqlcmd (Go)** (`winget install sqlcmd`), which reuses your `az login` session: `sqlcmd -S $SQL_FQDN -d $DB --authentication-method ActiveDirectoryDefault`. (With the ODBC sqlcmd use `-G -U "$ADMIN_UPN"` for an interactive browser sign-in instead.) Create the schema:

```sql
CREATE SCHEMA Marina;
GO
CREATE TABLE Marina.Berths (BerthId INT NOT NULL PRIMARY KEY, Pontoon CHAR(1) NOT NULL, LengthM DECIMAL(4,1) NOT NULL, IsFree BIT NOT NULL);
INSERT INTO Marina.Berths VALUES (1, 'A', 8.0, 1), (2, 'A', 10.0, 0), (3, 'B', 12.5, 1);
GO
SELECT SERVERPROPERTY('IsExternalAuthenticationOnly') AS EntraOnly;
```

The worker is an **Azure Container Instance** (the cheapest compute you can create with one CLI command) that installs sqlcmd (Go) and connects **as the managed identity**:

```bash
cat > worker.sh <<'EOF'
set -e
apt-get update -qq && apt-get install -y -qq curl >/dev/null
curl -fsSL https://packages.microsoft.com/keys/microsoft.asc -o /etc/apt/trusted.gpg.d/microsoft.asc
curl -fsSL https://packages.microsoft.com/config/ubuntu/22.04/prod.list -o /etc/apt/sources.list.d/mssql-release.list
apt-get update -qq && apt-get install -y -qq sqlcmd >/dev/null
sqlcmd -S "$SQL_FQDN" -d BerthBook --authentication-method ActiveDirectoryManagedIdentity -U "$UAMI_CLIENT_ID" -W \
  -Q "SELECT USER_NAME() AS db_user, (SELECT type_desc FROM sys.database_principals WHERE name = USER_NAME()) AS principal_type, COUNT(*) AS berths FROM Marina.Berths;"
EOF
SCRIPT_B64=$(base64 worker.sh | tr -d '\n')

run_worker () {   # creates the container, waits for it to finish, prints its output, deletes it
  az container create -g $RG -n aci-berthbook --image ubuntu:22.04 --os-type Linux --cpu 1 --memory 1 \
    --restart-policy Never --assign-identity $UAMI_ID \
    --environment-variables SQL_FQDN=$SQL_FQDN UAMI_CLIENT_ID=$UAMI_CLIENT_ID SCRIPT_B64=$SCRIPT_B64 \
    --command-line "/bin/bash -c 'echo \$SCRIPT_B64 | base64 -d | bash'" -o none
  while [ "$(az container show -g $RG -n aci-berthbook --query instanceView.state -o tsv)" = "Running" ]; do sleep 15; done
  az container logs -g $RG -n aci-berthbook
  az container delete -g $RG -n aci-berthbook -y -o none
}
```

### The runs

```text
R1  run_worker                                   -- BEFORE any database principal exists for the identity
R2  (as the Entra admin, in BerthBook)
      CREATE USER [id-berthbook-worker] FROM EXTERNAL PROVIDER;
      ALTER ROLE db_datareader ADD MEMBER [id-berthbook-worker];
      SELECT name, type, type_desc, CAST(sid AS UNIQUEIDENTIFIER) AS oid FROM sys.database_principals WHERE type IN ('E','X');
R3  run_worker
R4  (as the Entra admin) in master:   CREATE LOGIN berth_sql WITH PASSWORD = 'Str0ng!Passw0rd#2026';
                         in BerthBook: CREATE USER berth_sql FOR LOGIN berth_sql; ALTER ROLE db_datareader ADD MEMBER berth_sql;
R5  sqlcmd -S $SQL_FQDN -d $DB -U berth_sql -P 'Str0ng!Passw0rd#2026' -Q "SELECT COUNT(*) FROM Marina.Berths;"
R6  az sql server ad-only-auth disable -g $RG -n $SQL ; then repeat R5
R7  az sql server ad-only-auth get -g $RG -n $SQL ; and in T-SQL: SELECT SERVERPROPERTY('IsExternalAuthenticationOnly');
```

For each run R1–R7 state whether it **succeeds or fails**, and for the failures give the SQL error number the client receives. For R3 give the three column values the worker prints.

**Cost and cleanup.** Serverless GP 1 vCore bills per vCore-second only while the database is running (cents for this lab) plus a few GB of storage; the container instance costs cents per run. Delete everything at the end:

```bash
az group delete -n $RG --yes --no-wait
```

## Correct Answer

| Run | Outcome | Detail |
|---|---|---|
| R1 | **Fails** | sqlcmd reports **error 18456**, `Login failed for user '<token-identified principal>'.` — the token is valid, but no database principal maps to the identity's object id |
| R2 | **Succeeds** | one row: `id-berthbook-worker`, type `E`, `EXTERNAL_USER`, `oid` = the identity's principal (object) id |
| R3 | **Succeeds** | `db_user = id-berthbook-worker`, `principal_type = EXTERNAL_USER`, `berths = 3` |
| R4 | **Succeeds** | both statements run: Entra-only authentication does not prevent *creating* SQL principals |
| R5 | **Fails** | **error 18456**, `Login failed for user 'berth_sql'.` — SQL authentication is disabled server-wide |
| R6 | **Succeeds** | after `ad-only-auth disable`, the same R5 command returns `3` |
| R7 | **Succeeds** | `azureAdOnlyAuthentication: false` and `IsExternalAuthenticationOnly = 0` (it was `1` right after provisioning) |

## Explanation

### R1 — a valid token is not a database user

`--authentication-method ActiveDirectoryManagedIdentity -U <client id>` makes sqlcmd (Go) ask the container's identity endpoint for a token for `https://database.windows.net/` on behalf of the **user-assigned** identity (for a system-assigned identity you omit `-U`). Microsoft Entra issues the token, so the gateway accepts the connection — but inside `BerthBook` there is no principal whose `sid` equals the identity's object id, and the engine rejects the login with error **18456**. The user name in the message is the literal placeholder `<token-identified principal>`, because the login was identified by a token, not by a name. As with every 18456, the client only sees the generic text; the detailed state stays server-side (state 1, "Error information isn't available", is what clients get).

Azure RBAC roles on the server (SQL DB Contributor, Owner, ...) would not have helped: "Microsoft Entra authentication for Azure SQL doesn't integrate with Azure RBAC". Database access always needs a database principal.

### R2 — `CREATE USER ... FROM EXTERNAL PROVIDER`

Run by the Entra admin (any Entra principal with `ALTER ANY USER` works; a SQL login could not, because the engine must resolve the name through Microsoft Graph), the statement creates a **contained** user named after the identity's display name. The catalog shows `type = 'E'`, `type_desc = EXTERNAL_USER`, and `CAST(sid AS UNIQUEIDENTIFIER)` equals the identity's `principalId` from `az identity show`. Membership in `db_datareader` is the least privilege the worker needs. Note what was *not* needed: no login in `master`, no password, no `CREATE LOGIN`.

### R3 — the same container now works

Nothing changed on the container side; only the database principal appeared. `USER_NAME()` returns `id-berthbook-worker` and the principal type is `EXTERNAL_USER`. This is the whole passwordless pattern: the identity's credentials are managed by the platform, the connection string (`Authentication=Active Directory Managed Identity;User Id=<client id>` in `Microsoft.Data.SqlClient` terms) contains no secret, and access is granted by a catalog entry that maps an object id to a database user.

### R4 and R5 — Entra-only authentication disables *connecting*, not *creating*

The documentation is explicit: "Although SQL authentication is disabled, Microsoft Entra accounts with proper permissions can still create new SQL authentication logins and users. Newly created SQL authentication accounts can't connect to the server." So R4 succeeds and R5 fails with error 18456. The server also has a system-generated `CloudSA...` server admin (created because provisioning still requires SQL-admin values); it cannot connect either — it is "not a shared account, a fallback, or a break-glass path".

### R6 and R7 — the server-level switch

`az sql server ad-only-auth disable` (PowerShell `Disable-AzSqlServerActiveDirectoryOnlyAuthentication`, REST `.../azureADOnlyAuthentications/default` with `azureADOnlyAuthentication = 0`) re-enables SQL authentication for the whole logical server, and R5 immediately succeeds because `berth_sql` already existed. `ad-only-auth get` reports `azureAdOnlyAuthentication: false`; `SERVERPROPERTY('IsExternalAuthenticationOnly')` goes from `1` to `0`. Two rules attached to the switch: the Entra admin must exist before Entra-only can be enabled (`--enable-ad-only-auth` is why the server was created with an external admin and no `-u/-p`), and while the switch is on the Entra admin cannot be removed. Enabling or disabling requires Owner/Contributor or **SQL Security Manager**; SQL Server Contributor can set the Entra admin but cannot flip the switch (separation of duties).

### Verification notes

On the local SQL Server 2025 the same statements produce different, instructive messages: `CREATE USER [x] FROM EXTERNAL PROVIDER` fails with `Msg 37525 Command 'CREATE USER FROM EXTERNAL PROVIDER' is not supported as Azure Active Directory is not configured for this instance.` (Entra authentication on SQL Server needs Azure Arc or an Azure VM), and `SERVERPROPERTY('IsExternalAuthenticationOnly')` returns `0`.

Hands-on question (Azure subscription required); the T-SQL fragments that do not depend on Azure were checked on SQL Server 2025 RTM 17.0.1000.7; Azure-side behaviour is taken from the official documentation.

## DP-800 Exam Rule to Remember

```text
Managed identity -> Azure SQL, three independent gates:
  1. token:      Entra issues it (UAMI: client id in "User Id=" / sqlcmd -U; SMI: nothing)
  2. principal:  CREATE USER [<identity display name>] FROM EXTERNAL PROVIDER  (type E, sid = object id)
                 missing -> 18456 "Login failed for user '<token-identified principal>'."
  3. permission: ALTER ROLE db_datareader/db_datawriter ADD MEMBER ...   (Azure RBAC never grants DB access)

Entra-only authentication (server-level switch):
  az sql server create --enable-ad-only-auth --external-admin-*   |  az sql server ad-only-auth enable|disable|get
  SERVERPROPERTY('IsExternalAuthenticationOnly') = 1
  SQL logins/users can still be CREATED, they just cannot CONNECT (18456); CloudSA* admin is not a back door
  needs an Entra admin first; flipped by Owner/Contributor/SQL Security Manager, not SQL Server Contributor
sqlcmd (Go): --authentication-method ActiveDirectoryDefault (az login) | ActiveDirectoryManagedIdentity [-U client-id]
```
