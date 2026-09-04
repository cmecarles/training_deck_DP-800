# SQL Server question — Secure Model Endpoints 1

## Statement

A patent-search startup stores patent abstracts in an Azure SQL Database named `PatentScout` on the logical server `patentscout-sql.database.windows.net`. The database must call an Azure OpenAI embedding deployment named `embed-3-small` on the resource `patentscout-aoai` (endpoint `https://patentscout-aoai.openai.azure.com`) in two ways:

- from stored procedures through `sp_invoke_external_rest_endpoint`, and
- through an `EXTERNAL MODEL` used by `AI_GENERATE_EMBEDDINGS`.

The security review imposes these requirements:

1. **No API key** for Azure OpenAI may be stored anywhere in the database, in app settings, or in Key Vault; the database must authenticate to Azure OpenAI with its own Microsoft Entra identity.
2. The identity must hold the **minimum** Azure RBAC role that allows inference (embedding) calls on `patentscout-aoai`.
3. Outbound traffic from the logical server must be **restricted** so that `sp_invoke_external_rest_endpoint` can reach `patentscout-aoai.openai.azure.com` and nothing else.
4. Both the procedure call and the external model must reuse **one** database scoped credential.

Which configuration meets all four requirements?

### a.

Enable the system-assigned managed identity of the logical server `patentscout-sql`. On the `patentscout-aoai` resource assign that identity the **Cognitive Services OpenAI User** role. Enable *Restrict outbound networking* on the server and add the allowed FQDN `patentscout-aoai.openai.azure.com`. In `PatentScout`:

```sql
CREATE DATABASE SCOPED CREDENTIAL [https://patentscout-aoai.openai.azure.com]
    WITH IDENTITY = 'Managed Identity',
         SECRET = '{"resourceid":"https://cognitiveservices.azure.com"}';
GO
CREATE EXTERNAL MODEL PatentEmbedder
WITH (LOCATION = 'https://patentscout-aoai.openai.azure.com/openai/deployments/embed-3-small/embeddings?api-version=2024-02-01',
      API_FORMAT = 'Azure OpenAI', MODEL_TYPE = EMBEDDINGS, MODEL = 'text-embedding-3-small',
      CREDENTIAL = [https://patentscout-aoai.openai.azure.com]);
GO
EXEC sp_invoke_external_rest_endpoint
     @url = 'https://patentscout-aoai.openai.azure.com/openai/deployments/embed-3-small/embeddings?api-version=2024-02-01',
     @credential = [https://patentscout-aoai.openai.azure.com],
     @payload = N'{"input":"laser diode cooling"}', @response = @r OUTPUT;
```

### b.

Store the Azure OpenAI key in the database:

```sql
CREATE MASTER KEY ENCRYPTION BY PASSWORD = '...';
CREATE DATABASE SCOPED CREDENTIAL [AzureOpenAI]
    WITH IDENTITY = 'HTTPEndpointHeaders', SECRET = '{"api-key":"<key1 of patentscout-aoai>"}';
```

and pass `@credential = [AzureOpenAI]` / `CREDENTIAL = [AzureOpenAI]`. Because the key is encrypted by the database master key, it is not "stored in plaintext", and no RBAC role or outbound rule is needed.

### c.

Same as option a, but assign the server's managed identity the **Cognitive Services Contributor** role on `patentscout-aoai`, since a contributor can do everything a user can do, including creating deployments the team may need later.

### d.

Same as option a, but name the credential after the full deployment URL so that it matches the call exactly:

```sql
CREATE DATABASE SCOPED CREDENTIAL
  [https://patentscout-aoai.openai.azure.com/openai/deployments/embed-3-small/embeddings?api-version=2024-02-01]
    WITH IDENTITY = 'Managed Identity',
         SECRET = '{"resourceid":"https://cognitiveservices.azure.com"}';
```

## Correct Answer

**a**

## Explanation

### Why option a is correct

- **Managed-identity credential.** `CREATE DATABASE SCOPED CREDENTIAL ... WITH IDENTITY = 'Managed Identity'` tells the engine to obtain a Microsoft Entra token with the logical server's managed identity and send it as a bearer token in the request headers. The `SECRET` is a JSON document whose `resourceid` is the **audience** of the token — `https://cognitiveservices.azure.com` for Azure OpenAI / Azure AI services (`https://eventhubs.azure.net` for Event Hubs, an app's `APP_ID` for an App Service protected by Entra sign-in). No key is stored anywhere — requirement 1. The system-assigned identity of the server is used when no user-assigned identity is attached; if user-assigned identities exist, the server's *primary* identity is used.
- **Minimum role.** The Azure OpenAI RBAC documentation lists what each role can do; **Cognitive Services OpenAI User** is the least-privileged role that can "Make inference API calls with Microsoft Entra ID". It cannot see or regenerate keys, create deployments, or fine-tune — requirement 2.
- **Outbound firewall rules.** "Outbound firewall rules limit network traffic from the Azure SQL Database logical server to a customer defined list" and the feature explicitly governs `sp_invoke_external_rest_endpoint` (besides auditing, vulnerability assessment, import/export, `OPENROWSET` and `BULK INSERT`). Enabling *Restrict outbound networking* (`az sql server update ... --set restrictOutboundNetworkAccess="Enabled"`) and adding `patentscout-aoai.openai.azure.com` (`az sql server outbound-firewall-rule create --outbound-rule-fqdn ...`) satisfies requirement 3. Independently of that, Azure SQL Database only allows calls to an allow-list of Azure domains, `*.openai.azure.com` among them.
- **One credential, name matching.** The credential name must be a valid URL whose *scheme + FQDN* match the called URL, whose path is a **prefix** of the called path, and which contains **no query string**. `https://patentscout-aoai.openai.azure.com` is the most generic valid name, so it matches the embeddings URL used both by `@credential` and by `CREATE EXTERNAL MODEL ... CREDENTIAL` — requirement 4. Callers additionally need `EXECUTE ANY EXTERNAL ENDPOINT` (procedure) and `EXECUTE ON EXTERNAL MODEL::PatentEmbedder` (model), plus `REFERENCES` on the credential.

### Why option b is wrong

`IDENTITY = 'HTTPEndpointHeaders'` with `{"api-key": ...}` is the documented **key-based** pattern — valid, but it violates requirement 1: the key is a long-lived secret stored in the database (encrypted by the master key, yes, but retrievable by anyone who can create a credential-using call, and rotated only by hand). It also ignores the credential-name rules that both `sp_invoke_external_rest_endpoint` and `CREATE EXTERNAL MODEL` document for endpoint credentials ("Must be a valid URL" whose scheme + FQDN match the called URL): `[AzureOpenAI]` is not a URL, so it is not the documented way to bind a credential to `https://patentscout-aoai.openai.azure.com` — every Azure OpenAI example in the docs names the credential after the endpoint. Finally, "no outbound rule is needed" contradicts requirement 3.

### Why option c is wrong

This is the subtle distractor. **Cognitive Services Contributor** is a *management-plane* role: it can create resources, view and regenerate keys, create deployments and guardrails — but the same role table marks "Make inference API calls with Microsoft Entra ID" as **not allowed** for it. A managed identity holding only that role would get `401`/`403` on the embeddings call, so the solution does not even work, and if it did it would violate least privilege (requirement 2) by granting key-management rights to a database identity. (Note the special case in the `CREATE EXTERNAL MODEL` docs: on **SQL Server 2025** with an Azure Arc managed identity, the documented requirement is **Cognitive Services OpenAI Contributor**; for Azure SQL Database, `Cognitive Services OpenAI User` is the one the `sp_invoke_external_rest_endpoint` example prescribes.)

### Why option d is wrong

The credential-name rules forbid it twice: "The URL must not contain a query string" (`?api-version=...`), and the credential must point to a path that is **more generic** than the request URL, never more specific. Even if the query string were dropped, a credential scoped to `/openai/deployments/embed-3-small/embeddings` would match only that deployment; the moment the team calls a second deployment they need a second credential, defeating requirement 4. The generic `https://<host>` name is the intended pattern.

### SQL Server 2025 differences

On SQL Server 2025 (17.x) the same objects exist but two server options gate them: `sp_configure 'external rest endpoint enabled', 1` (the procedure is disabled by default; it is enabled by default on Azure SQL Database and Fabric SQL database) and `sp_configure 'allow server scoped db credentials', 1` to let a database scoped credential use the **Azure Arc** managed identity of the host. Both require `ALTER SETTINGS` (`sysadmin`/`serveradmin`) followed by `RECONFIGURE WITH OVERRIDE`. There is no outbound-firewall-rule feature on SQL Server; network egress is controlled at the OS/network level. `sys.database_scoped_credentials` shows the credential's `name`, `credential_identity` and `create_date` but never the secret.

### Least privilege inside the database

The credential and the model are securables of their own, so the application principal needs three grants and nothing more:

```sql
GRANT EXECUTE ANY EXTERNAL ENDPOINT TO [app-patentscout];                         -- sp_invoke_external_rest_endpoint
GRANT REFERENCES ON DATABASE SCOPED CREDENTIAL::[https://patentscout-aoai.openai.azure.com]
    TO [app-patentscout];                                                          -- use the credential
GRANT EXECUTE ON EXTERNAL MODEL::PatentEmbedder TO [app-patentscout];              -- AI_GENERATE_EMBEDDINGS

SELECT name, credential_identity, create_date FROM sys.database_scoped_credentials;  -- no secret column
SELECT name, location, api_format, model_type FROM sys.external_models;
```

Creating the objects requires `CONTROL` on the database (credential) and `CREATE EXTERNAL MODEL` / `ALTER ANY EXTERNAL MODEL` (model). `sp_invoke_external_rest_endpoint` reports the wait type `HTTP_EXTERNAL_CONNECTION` while it waits, and returns the HTTP status code as its return value (0 for any 2xx), so a `401`/`403` from a wrong RBAC role is visible as the procedure's return value, not as a T-SQL error.

Conceptual question (Azure / tooling); not executed against an engine.

## DP-800 Exam Rule to Remember

```text
DATABASE SCOPED CREDENTIAL for model endpoints
  name  = URL: scheme+FQDN must match the call, path may only be a PREFIX, NO query string
  IDENTITY = 'Managed Identity', SECRET = '{"resourceid":"https://cognitiveservices.azure.com"}'
              -> passwordless; server SMI (or primary UAMI); Azure OpenAI role:
                 Cognitive Services OpenAI User  (inference only)   <- minimum
                 Cognitive Services OpenAI Contributor (+ deployments, fine-tuning; SQL 2025 Arc docs)
                 Cognitive Services Contributor  (management, keys; CANNOT call inference)
  IDENTITY = 'HTTPEndpointHeaders', SECRET = '{"api-key":"..."}'  -> key-based (secret stored)
  IDENTITY = 'HTTPEndpointQueryString' / 'Shared Access Signature' -> other key-based forms
Same credential serves sp_invoke_external_rest_endpoint (@credential) and CREATE EXTERNAL MODEL (CREDENTIAL).
Permissions: EXECUTE ANY EXTERNAL ENDPOINT, EXECUTE ON EXTERNAL MODEL::x, REFERENCES on the credential.
Azure SQL DB: allow-listed domains only (*.openai.azure.com, *.cognitiveservices.azure.com ...),
  plus optional outbound firewall rules (Restrict outbound networking + allowed FQDNs).
SQL Server 2025: sp_configure 'external rest endpoint enabled' = 1,
                 'allow server scoped db credentials' = 1 (Arc managed identity), RECONFIGURE WITH OVERRIDE.
```
