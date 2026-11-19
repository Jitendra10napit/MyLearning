**The On-Behalf-Of (OBO) flow**

**The problem it solves**

Angular calls **Employee API**. To answer, Employee API must call **Payroll API**, and Payroll must know **which user** is asking, so it shows Amit only *his* salary.

```
Angular ──(token for Employee API)──► Employee API ──(??? token)──► Payroll API
```

What token should Employee API send to Payroll?

| Option | Problem |
|---|---|
| ❌ **Forward the same token** | Its audience is `api://employee-api`, so Payroll must reject it. If Payroll accepts it, any token for any API would work (a security hole) |
| ❌ **Use Employee API's own identity** (client credentials) | Payroll sees only "Employee API" and **loses the user**, so it can't apply per-user permissions or audit |
| ✅ **OBO** | Employee API **exchanges** the user's token for a **new token for Payroll** that **still represents the same user** |

**Analogy: an office reception desk**

You (the user) have a visitor pass for **Building A** (Employee API). Building A's staff need a document from **Building B** (Payroll) for you.

They don't hand over *your* Building A pass (wrong building), and they don't go as themselves (B needs to know it's for *you*). Instead they go to **central security** (Entra ID) and say:

> *"Here's Amit's pass for Building A, and here's our staff ID. Please issue a pass for Building B on Amit's behalf."*

Security checks both and issues a **new pass for Building B, in Amit's name, requested by Building A**.

**Step by step**

```
① Angular ── access token (aud = Employee API, user = Amit) ──► Employee API

② Employee API ──► Entra ID (token endpoint):
      "Here is the user's token (assertion) + my own credentials.
       Give me a token for Payroll API, on behalf of this user."

③ Entra ID checks:
      ✓ the incoming token is valid and meant for Employee API
      ✓ Employee API's credentials (secret / certificate) are valid
      ✓ the user has consented to Payroll permissions
   → issues a NEW access token:
      aud = Payroll API, user = Amit (sub), scopes = Payroll.Read
      azp/appid = Employee API (who asked)

④ Employee API ── new token ──► Payroll API

⑤ Payroll validates it: right audience, real user Amit → returns Amit's salary only
```

**The token exchange request (what happens under the hood)**

```http
POST https://login.microsoftonline.com/{tenant}/oauth2/v2.0/token
Content-Type: application/x-www-form-urlencoded

grant_type=urn:ietf:params:oauth:grant-type:jwt-bearer
&client_id=<employee-api-client-id>
&client_secret=<secret or certificate assertion>
&assertion=<the user's incoming access token>
&scope=api://payroll-api/Payroll.Read
&requested_token_use=on_behalf_of
```

**Code: Microsoft.Identity.Web does it for you**

```
dotnet add package Microsoft.Identity.Web
dotnet add package Microsoft.Identity.Web.DownstreamApi
```

**appsettings.json (Employee API):**

```json
"AzureAd": {
  "Instance": "https://login.microsoftonline.com/",
  "TenantId": "<tenant-id>",
  "ClientId": "<employee-api-client-id>",
  "Audience": "api://employee-api",
  "ClientCredentials": [
    { "SourceType": "KeyVault",
      "KeyVaultUrl": "https://myvault.vault.azure.net",
      "KeyVaultCertificateName": "employee-api-cert" }
  ]
},
"PayrollApi": {
  "BaseUrl": "https://payroll.mycompany.com",
  "Scopes": [ "api://payroll-api/Payroll.Read" ]
}
```

**Program.cs:**

```csharp
builder.Services
    .AddAuthentication(JwtBearerDefaults.AuthenticationScheme)
    .AddMicrosoftIdentityWebApi(builder.Configuration, "AzureAd")
        .EnableTokenAcquisitionToCallDownstreamApi()          // turns on OBO
        .AddDownstreamApi("PayrollApi", builder.Configuration.GetSection("PayrollApi"))
        .AddDistributedTokenCaches();                         // cache OBO tokens (Redis in prod)
```

**Controller:**

```csharp
[Authorize]
[ApiController]
[Route("api/v1/employees")]
public class EmployeesController(IDownstreamApi downstream) : ControllerBase
{
    [HttpGet("me/salary")]
    public async Task<IActionResult> MySalary(CancellationToken ct)
    {
        // Behind the scenes: take the incoming token → OBO exchange → call Payroll
        var salary = await downstream.GetForUserAsync<SalaryDto>(
            "PayrollApi", o => o.RelativePath = "salary/me", cancellationToken: ct);

        return Ok(salary);
    }
}
```

**Payroll API** just validates normally, and **knows the user**:

```csharp
[Authorize]
[RequiredScope("Payroll.Read")]
[HttpGet("salary/me")]
public IActionResult GetMine()
{
    var userId = User.FindFirst("oid")?.Value;   // Amit's object id, from the OBO token
    return Ok(_payroll.GetSalary(userId));
}
```

**Entra ID setup (one time)**

1. **Payroll API** app registration → **Expose an API** → add the scope `Payroll.Read`.
2. **Employee API** app registration → **API permissions** → add `Payroll.Read` (delegated). **Grant admin consent**, because the user never sees a consent screen for downstream APIs.
3. Employee API needs a **credential** (certificate preferred, or a secret), because OBO requires a **confidential client**.
4. Optional: list the Angular app as a **known client application** on Employee API, so a user consents to everything in one screen at login.

**OBO vs other flows**

| Flow | Who's the caller | User in the downstream token? | Use when |
|---|---|---|---|
| **Auth code + PKCE** | The user, through the client | ✅ | SPA or web app logs the user in |
| **On-Behalf-Of** | API, **acting for the user** | ✅ | API → API, the user's permissions matter |
| **Client credentials / Managed Identity** | The app itself | ❌ | Background jobs, daemons, service-to-service with no user |

**Things to watch (senior points)**

- **Only for confidential clients:** a backend API with a secret or certificate. An SPA **can't** do OBO, because it can't keep a secret.
- **Cache OBO tokens.** Exchanging on every request is slow and gets throttled. Microsoft.Identity.Web caches per user, and you can use Redis for multiple instances.
- **Least privilege:** request only the scopes you need for each downstream API.
- **Chains:** OBO can repeat (A → B → C), but every hop adds latency and a dependency, so keep chains short.
- **Prefer certificates or a managed identity federated credential** over secrets, stored in Key Vault.
- **Conditional Access / MFA errors** can come back during OBO (an `interaction_required` claims challenge). The API must pass that challenge back to the client so the user can re-authenticate.
- **Audit:** the OBO token carries both the **user** (`oid`/`sub`) and the **calling app** (`azp`/`appid`), so Payroll logs show *"Amit, through Employee API"*.

**How to say it in the interview (D-A-E-T-E)**

> "**Define:** On-Behalf-Of lets an API that received a user's token exchange it with the identity provider for a *new* token to call a downstream API, while still representing the same user.
>
> **Analogy:** Like an office asking central security for a pass to another building *in my name*, instead of handing over my pass or going in as themselves.
>
> **Example:** Angular sends a token for Employee API. Employee API uses Microsoft.Identity.Web with `EnableTokenAcquisitionToCallDownstreamApi` to get a token for Payroll API with the `Payroll.Read` scope, so Payroll returns only that user's salary and logs both the user and the calling app.
>
> **Trade-off:** It needs a confidential client with a certificate in Key Vault, admin consent for the downstream scopes, and token caching, because each exchange is a network call. Long chains add latency, so for calls that don't need the user context I use client credentials or Managed Identity instead.
>
> **Why not forward the token:** its audience is wrong. Accepting it would break audience validation and open a confused-deputy hole."


**Client credentials and Managed Identity: service-to-service calls with no user**

**When to use them**

When **no user is involved**: a background job, a worker, a scheduled task, a service calling another service *as itself*.

```
Nightly Payroll Job ──(token: "I am Payroll Job")──► Employee API
Media Worker        ──(token)──► Blob Storage / Service Bus / SQL
```

Here the caller is **the application**, not Amit.

**The difference in one line**

- **Client credentials:** the app proves who it is with a **secret or certificate** that **you** create, store and rotate.
- **Managed Identity:** the same idea, but **Azure manages the credential** for you. No secret exists at all.

Analogy: client credentials = **an ID card you must carry and renew yourself**. Managed Identity = **the building recognizes your face**: nothing to carry, nothing to lose.

**Part 1: Managed Identity (use this first on Azure)**

**What it is:** Azure gives your App Service, Function, Container App, AKS pod or VM its **own identity in Entra ID**. Your code asks the local Azure endpoint for a token, and Azure handles the credential and its rotation behind the scenes.

**Two types:**

| Type | Lifecycle | Use when |
|---|---|---|
| **System-assigned** | Created with the resource, deleted with it | One app, simple setup |
| **User-assigned** | A standalone resource you attach to one or many apps | Several apps share the same permissions, or the identity must survive redeploys |

**Step 1: turn it on**

```bash
az webapp identity assign -g myRg -n employee-api           # system-assigned
```

**Step 2: give it permissions (RBAC)**

```bash
# Read/write blobs
az role assignment create --assignee <principal-id> \
  --role "Storage Blob Data Contributor" \
  --scope /subscriptions/<sub>/resourceGroups/myRg/providers/Microsoft.Storage/storageAccounts/mystorage

# Send to Service Bus
az role assignment create --assignee <principal-id> \
  --role "Azure Service Bus Data Sender" --scope <servicebus-resource-id>

# Read Key Vault secrets
az role assignment create --assignee <principal-id> \
  --role "Key Vault Secrets User" --scope <keyvault-resource-id>
```

**Step 3: code with `DefaultAzureCredential` (no secrets anywhere)**

```csharp
var credential = new DefaultAzureCredential();

// Blob Storage
var blobs = new BlobServiceClient(new Uri("https://mystorage.blob.core.windows.net"), credential);

// Service Bus
var bus = new ServiceBusClient("mysbns.servicebus.windows.net", credential);

// Key Vault
var secrets = new SecretClient(new Uri("https://myvault.vault.azure.net"), credential);

// Key Vault as a configuration source
builder.Configuration.AddAzureKeyVault(new Uri("https://myvault.vault.azure.net"), credential);
```

**Azure SQL with Managed Identity (no password in the connection string):**

```json
"ConnectionStrings": {
  "Default": "Server=tcp:mysql.database.windows.net;Database=EmployeeDb;Authentication=Active Directory Default;"
}
```

```sql
-- Run once in the database, as the Entra admin
CREATE USER [employee-api] FROM EXTERNAL PROVIDER;
ALTER ROLE db_datareader ADD MEMBER [employee-api];
ALTER ROLE db_datawriter ADD MEMBER [employee-api];
```

**How `DefaultAzureCredential` works everywhere**

It tries several sources in order, and uses the **first one that works**:

```
Environment variables → Workload Identity (AKS) → Managed Identity (in Azure)
→ Visual Studio → Azure CLI (az login) → ...
```

- **In Azure:** it uses the **Managed Identity**.
- **On your laptop:** it uses **your own login** from Visual Studio or `az login`.

The **same code** runs locally and in production. Give yourself the same RBAC roles on dev resources.

In production you can pin it to Managed Identity to avoid surprises and speed up startup:

```csharp
var credential = new ManagedIdentityCredential();                        // system-assigned
var credential = new ManagedIdentityCredential("<user-assigned-client-id>");  // user-assigned
```

**Part 2: calling your own API with an app identity**

The nightly **Payroll Job** must call **Employee API** as itself.

**Step 1: define an app role on Employee API** (Entra ID → Employee API app registration → **App roles**):

```
Display name: Employees.Read.All
Value:        Employees.Read.All
Allowed:      Applications          ← for apps, not users
```

**Step 2: assign that role to the caller**
- With **client credentials**: Payroll Job app registration → API permissions → add `Employees.Read.All` (**Application** type) → **grant admin consent**.
- With **Managed Identity**: assign the app role to the MI's service principal (a one-time PowerShell or Graph call, `New-MgServicePrincipalAppRoleAssignment`).

**Step 3: the caller gets a token**

Scope is always **`<api>/.default`**, meaning "all application permissions granted to me for this API".

**With Managed Identity:**

```csharp
var credential = new DefaultAzureCredential();
var token = await credential.GetTokenAsync(
    new TokenRequestContext(["api://employee-api/.default"]), ct);

http.DefaultRequestHeaders.Authorization =
    new AuthenticationHeaderValue("Bearer", token.Token);

var employees = await http.GetFromJsonAsync<List<EmployeeDto>>("https://employee-api/api/v1/employees", ct);
```

**With client credentials (outside Azure, or no MI available), using MSAL:**

```csharp
var app = ConfidentialClientApplicationBuilder
    .Create("<payroll-job-client-id>")
    .WithCertificate(cert)                       // preferred over WithClientSecret("...")
    .WithAuthority("https://login.microsoftonline.com/<tenant-id>")
    .Build();

var result = await app.AcquireTokenForClient(["api://employee-api/.default"])
                      .ExecuteAsync(ct);         // MSAL caches the token in memory

http.DefaultRequestHeaders.Authorization =
    new AuthenticationHeaderValue("Bearer", result.AccessToken);
```

**Cleaner: let a DelegatingHandler attach the token to every call**

```csharp
public class AppTokenHandler(TokenCredential credential) : DelegatingHandler
{
    private static readonly string[] Scopes = ["api://employee-api/.default"];

    protected override async Task<HttpResponseMessage> SendAsync(HttpRequestMessage req, CancellationToken ct)
    {
        var token = await credential.GetTokenAsync(new TokenRequestContext(Scopes), ct);
        req.Headers.Authorization = new AuthenticationHeaderValue("Bearer", token.Token);
        return await base.SendAsync(req, ct);
    }
}

// Program.cs
builder.Services.AddSingleton<TokenCredential>(new DefaultAzureCredential());
builder.Services.AddTransient<AppTokenHandler>();
builder.Services.AddHttpClient<IEmployeeApiClient, EmployeeApiClient>(c =>
        c.BaseAddress = new Uri("https://employee-api"))
    .AddHttpMessageHandler<AppTokenHandler>();
```

**Step 4: Employee API checks the app role**

The token has no user. It carries **`roles`** (app roles) instead of **`scp`** (user scopes):

```json
{ "aud": "api://employee-api", "roles": ["Employees.Read.All"], "azp": "<payroll-job-id>", "idtyp": "app" }
```

```csharp
[Authorize(Roles = "Employees.Read.All")]      // app permission
[HttpGet]
public Task<IActionResult> GetAll(...) { ... }
```

If one endpoint serves both users and apps, accept either:

```csharp
[Authorize]
[RequiredScopeOrAppPermission(
    RequiredScopesConfigurationKey = "AzureAd:Scopes:Read",       // users: scp
    RequiredAppPermissionsConfigurationKey = "AzureAd:AppPermissions:Read")]   // apps: roles
```

**Part 3: running outside Azure or on Kubernetes**

| Where it runs | Use |
|---|---|
| App Service, Functions, Container Apps, VM | **Managed Identity** |
| **AKS** | **Workload Identity**: the pod's service account is federated to a user-assigned MI, still with no secrets |
| GitHub Actions / Azure DevOps | **Federated credentials (OIDC)**: the pipeline gets tokens without stored secrets |
| On-prem / another cloud | **Client credentials with a certificate**, or **Azure Arc** to get a Managed Identity |

**Comparison**

| | **Client credentials** | **Managed Identity** |
|---|---|---|
| Credential | A secret or certificate **you** manage | **Azure manages it**, there's nothing to store |
| Rotation | Your job (and expiry outages are common) | Automatic |
| Leak risk | Secret in config, code or a pipeline | None, there's no secret |
| Works outside Azure | ✅ Yes | ❌ No (except Arc / federation) |
| Code | MSAL `AcquireTokenForClient` | `DefaultAzureCredential` |
| Permissions | App roles / API permissions + admin consent | RBAC roles for Azure resources, app roles for your APIs |
| Token type | App-only (`roles`, no user) | App-only (`roles`, no user) |

**Pick the right flow (one table to remember)**

| Situation | Flow |
|---|---|
| User logs into Angular | Auth code + PKCE → ID token + access token |
| API calls another API **for that user** | **On-Behalf-Of** |
| Job or service calls an API or Azure resource **as itself**, running in Azure | **Managed Identity** |
| Same, but running outside Azure | **Client credentials (certificate)** |
| CI/CD pipeline deploys to Azure | **Federated credentials (OIDC)** |

**How to say it in the interview (D-A-E-T-E)**

> "**Define:** For service-to-service calls with no user, the app authenticates as itself. With client credentials it presents a secret or certificate. With Managed Identity, Azure issues and rotates the credential for the resource, so there's no secret at all.
>
> **Analogy:** Client credentials is an ID card you have to carry and renew. Managed Identity is the building recognizing your face.
>
> **Example:** Our workers use `DefaultAzureCredential` with Managed Identity for Blob, Service Bus, Key Vault and Azure SQL, granted through RBAC roles like Storage Blob Data Contributor. When a job calls our own API, it requests a token for `api://employee-api/.default`, and the API authorizes it with an app role like `Employees.Read.All`.
>
> **Trade-off:** Managed Identity only works inside Azure. Outside it I use client credentials with a certificate in Key Vault, or federated credentials for pipelines. App tokens have no user context, so for user-specific data I use On-Behalf-Of instead.
>
> **Experience:** Moving from connection strings and secrets to Managed Identity removed all secret-rotation work and the outages that came from expired secrets."
