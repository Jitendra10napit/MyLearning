# **How to Apply Security to ASP.NET Core Web API – Step by Step**

For a Lead Engineer / Technical Architect interview, you should explain API security in layers. Authentication alone is not enough; we need to secure the entire API lifecycle.

Let's take an example of an **Employee Management API using ASP.NET Core, Angular, SQL Server and Microsoft Entra ID**.

## **1. Understand the API security layers**

## **API Security Architecture**

**Angular / External Client**

HTTPS · Secure token handling

**Azure API Management**

Rate limiting · IP restrictions · API policies

**ASP.NET Core Security Middleware**

HTTPS · CORS · Authentication · Authorization

**Business Layer**

Validation · Business permissions · Tenant isolation

**Database / External Services**

SQL Server · Key Vault · Service Bus

## **2. Step 1 – Enable HTTPS**

Always encrypt communication between client and API.

In `Program.cs`:

```
var app = builder.Build();

app.UseHttpsRedirection();
```

Production should use valid TLS certificates and enforce HTTPS at the ingress or hosting layer too.

Why?

- Prevents credentials from being transmitted in plain text.
- Protects bearer tokens.
- Protects sensitive employee information.

## **3. Step 2 – Implement Authentication using JWT / Entra ID**

Authentication answers: **Who is calling my API?**

Install:

```
dotnet add package Microsoft.Identity.Web
```

Configure `Program.cs`:

```
builder.Services
    .AddAuthentication()
    .AddMicrosoftIdentityWebApi(
        builder.Configuration.GetSection("AzureAd"));
```

Configure `appsettings.json`:

```
{
  "AzureAd": {
    "Instance": "https://login.microsoftonline.com/",
    "TenantId": "YOUR_TENANT_ID",
    "ClientId": "YOUR_API_CLIENT_ID"
  }
}
```

Add middleware:

```
app.UseAuthentication();
app.UseAuthorization();
```

Protect API:

```
[Authorize]
[HttpGet("employees")]
public IActionResult GetEmployees()
{
    return Ok("Employee data");
}
```

Now an unauthenticated request receives HTTP 401.

## **4. Step 3 – Implement Authorization using roles and policies**

Authorization answers: **What is the caller allowed to do?**

Suppose we have three roles:

| Role     | Permission                   |
| -------- | ---------------------------- |
| Admin    | Create, Read, Update, Delete |
| Manager  | Read, Update                 |
| Employee | Read own profile             |

Example:

```
[Authorize(Roles = "Admin")]
[HttpDelete("{id}")]
public IActionResult Delete(int id)
{
    return NoContent();
}
```

Manager or Admin:

```
[Authorize(Roles = "Admin,Manager")]
[HttpPut("{id}")]
public IActionResult Update(int id)
{
    return Ok();
}
```

For more flexible security, use policies.

```
builder.Services.AddAuthorization(options =>
{
    options.AddPolicy("CanReadEmployees",
        policy =>
            policy.RequireClaim(
                "Employee.Read", "true"));
});
```

Apply:

```
[Authorize(Policy = "CanReadEmployees")]
[HttpGet]
public IActionResult GetEmployees()
{
    return Ok();
}
```

Note: The claim name and values must match the actual claims issued by your identity provider. For Entra ID, delegated scopes are commonly represented by `scp`, while application permissions are commonly represented by `roles`.

## **5. Step 4 – Implement API-level data security**

This is very important for multi-tenant applications.

Suppose:

- Tenant A has 100 employees.
- Tenant B has 200 employees.

Tenant A must never access Tenant B's data.

One approach is to extract the tenant identity from a validated token claim and apply it to every database query.

```
public class EmployeeService
{
    private readonly AppDbContext _db;
    private readonly IHttpContextAccessor _http;

    public EmployeeService(
        AppDbContext db,
        IHttpContextAccessor http)
    {
        _db = db;
        _http = http;
    }

    public async Task<List<Employee>> GetEmployees()
    {
        var tenantId = _http.HttpContext!
            .User.FindFirst("tenant_id")?.Value;

        return await _db.Employees
            .Where(x => x.TenantId == tenantId)
            .ToListAsync();
    }
}
```

In Microsoft Entra ID, the tenant identifier is commonly represented by the `tid` claim. Use a claim mapped and validated according to your token configuration, rather than trusting a tenant ID supplied by the client.

Also enforce tenant isolation on updates, deletes and individual employee lookups, not just list queries.

## **6. Step 5 – Input validation and SQL injection prevention**

Never trust client input.

Use DTO validation:

```
public class CreateEmployeeDto
{
    [Required]
    [MaxLength(100)]
    public string Name { get; set; }

    [Required]
    [EmailAddress]
    public string Email { get; set; }
}
```

Use EF Core parameterized queries:

```
var employee = await _db.Employees
    .FirstOrDefaultAsync(
        x => x.Email == email);
```

Avoid building SQL queries using string concatenation.

Also validate:

- Request size.
- Allowed file extensions and file content.
- Numeric ranges.
- Business rules.
- Unexpected properties in sensitive DTOs.

## **7. Step 6 – Implement CORS**

CORS controls which browser origins are allowed to call your API.

Suppose Angular runs on:

`https://myemployeeapp.com`

Configure:

```
builder.Services.AddCors(options =>
{
    options.AddPolicy("AngularPolicy", policy =>
    {
        policy
            .WithOrigins("https://myemployeeapp.com")
            .AllowAnyHeader()
            .AllowAnyMethod();
    });
});
```

Middleware:

```
app.UseCors("AngularPolicy");
```

Do not use `AllowAnyOrigin()` with credentials in production.

Remember: CORS is a browser security mechanism, not authentication. External server-to-server clients are not restricted by browser CORS.

## **8. Step 7 – Implement Rate Limiting**

Suppose someone sends 10,000 requests per minute to your API.

Rate limiting protects your API from excessive traffic and abuse.

ASP.NET Core:

```
builder.Services.AddRateLimiter(options =>
{
    options.AddFixedWindowLimiter(
        "EmployeePolicy", limiter =>
        {
            limiter.PermitLimit = 100;
            limiter.Window = TimeSpan.FromMinutes(1);
            limiter.QueueLimit = 0;
        });
});
```

Apply:

```
app.UseRateLimiter();
```

On controller:

```
[EnableRateLimiting("EmployeePolicy")]
[HttpGet]
public IActionResult GetEmployees()
{
    return Ok();
}
```

A client exceeding the configured limit receives HTTP 429.

In distributed deployments, consider API Management or a distributed rate-limiting strategy rather than relying only on per-instance counters.

## **9. Step 8 – Secure secrets and configuration**

Never hardcode:

- Database passwords.
- JWT signing keys.
- API keys.
- Client secrets.
- Connection strings containing credentials.

Use Azure Key Vault.

Example:

```
builder.Configuration.AddAzureKeyVault(
    new Uri("https://myvault.vault.azure.net/"),
    new DefaultAzureCredential());
```

Prefer Managed Identity for accessing Key Vault from Azure App Service.

## **10. Step 9 – Logging and monitoring**

Use Application Insights and Azure Monitor.

Log:

- User ID or client application ID.
- API endpoint.
- Request correlation ID.
- Response status.
- Execution duration.
- Authorization failures.

Never log passwords, access tokens, refresh tokens or other secrets.

Example:

```
logger.LogInformation(
    "Employee API called by {UserId}",
    userId);
```

## **11. Step 10 – Secure external service communication**

Suppose Payroll Service calls your Employee API.

Use OAuth 2.0 Client Credentials flow.

```
Payroll Service
      |
      | Client ID + Certificate
      v
Microsoft Entra ID
      |
      | Application Access Token
      v
Employee API
      |
      | Validate JWT + App Role
      v
Employee Data
```

Use separate application permissions for each external service.

For example:

| External application | Permission             |
| -------------------- | ---------------------- |
| Payroll Service      | Employee.Read.All      |
| Reporting Service    | Employee.Read.All      |
| HR Service           | Employee.ReadWrite.All |

Use least privilege, and consider network restrictions and private connectivity for sensitive integrations.

# **12. Complete ASP.NET Core security configuration**

Here is a simplified combined example of the middleware setup.

```

using Microsoft.Identity.Web;
using System.Threading.RateLimiting;

var builder = WebApplication.CreateBuilder(args);

builder.Services.AddControllers();

builder.Services
    .AddAuthentication()
    .AddMicrosoftIdentityWebApi(
        builder.Configuration.GetSection("AzureAd"));

builder.Services.AddAuthorization(options =>
{
    options.AddPolicy("EmployeeRead", policy =>
    {
        policy.RequireAuthenticatedUser();
        policy.RequireClaim("Employee.Read", "true");
    });
});

builder.Services.AddCors(options =>
{
    options.AddPolicy("AngularPolicy", policy =>
    {
        policy.WithOrigins("https://myemployeeapp.com")
              .AllowAnyHeader()
              .AllowAnyMethod();
    });
});

builder.Services.AddRateLimiter(options =>
{
    options.AddFixedWindowLimiter("ApiPolicy", limiter =>
    {
        limiter.PermitLimit = 100;
        limiter.Window = TimeSpan.FromMinutes(1);
        limiter.QueueLimit = 0;
    });
});

var app = builder.Build();

app.UseHttpsRedirection();

app.UseRouting();

app.UseCors("AngularPolicy");

app.UseAuthentication();

app.UseRateLimiter();

app.UseAuthorization();

app.MapControllers();

app.Run();

```

This is a starting configuration, not a complete production security implementation. The policy claim must match your issued token; configure forwarded headers, trusted proxy handling, error handling, audit logging, and appropriate per-user or per-client rate limits for your deployment.

# **13. Final checklist – What should a Technical Architect implement?**

**API Security Implementation**

0/13 completed

HTTPS / TLS encryption

JWT authentication using Entra ID

Role-based and policy-based authorization

API scope and application permission validation

Multi-tenant data isolation

Input validation and SQL injection prevention

CORS configuration

Rate limiting and throttling

Secure secrets using Azure Key Vault

External API authentication using OAuth 2.0

Centralized exception handling

Logging and monitoring using Application Insights

Security testing and vulnerability scanning

## **14. Interview-ready answer**

If an interviewer asks, "How do you secure your ASP.NET Core API?", explain:

"I follow a defense-in-depth approach.

First, I enforce HTTPS for secure communication. For authentication, I integrate Microsoft Entra ID using OAuth 2.0 and OIDC, and validate JWT access tokens in ASP.NET Core.

For authorization, I implement role-based and policy-based access control to ensure users can only access permitted resources.

I also implement input validation, SQL injection prevention, CORS restrictions, rate limiting and tenant-level data isolation.

For external integrations, I use OAuth Client Credentials flow with application permissions. If a downstream service needs to act on behalf of a user, I use the OBO flow.

Finally, I store secrets in Azure Key Vault, use Managed Identity, and monitor security events through Application Insights and Azure Monitor."

**One important architect-level point:** Security must be enforced at multiple levels: API gateway, application, business logic, database and infrastructure. Adding `[Authorize]` alone does not make an API secure.
