# **How to Apply Security to ASP.NET Core Web API – Step by Step**

For a Lead Engineer / Technical Architect interview, you should explain API security in layers. Authentication alone is not enough; we need to secure the entire API lifecycle.

Let's take an example of an **Employee Management API using ASP.NET Core, Angular, SQL Server and Microsoft Entra ID**.

## **1. Understand the API security layers**

## **API Security Architecture**

**Angular / External Client**

HTTPS · Secure token handling

**Azure API Management**

Rate limiting · IP restrictions · API policies

**ASP.NET Core Security Middleware**# **Security Architecture: Session vs JWT, Authentication, Authorization, OAuth 2.0, OIDC and Refresh Tokens**

Let's design this from a **Senior .NET Engineer / Technical Architect interview perspective**, using a practical Employee Management API.

We will cover:

1. Authentication vs Authorization.
2. Session-based authentication.
3. JWT-based authentication.
4. OAuth 2.0 vs OpenID Connect (OIDC).
5. Access token and refresh token.
6. External services accessing your APIs.
7. Complete ASP.NET Core implementation flow.
8. Azure security architecture.

# **1. First understand the security concepts**

These concepts are related but solve different problems.

| Concept                | Purpose                            | Example                           |
| ---------------------- | ---------------------------------- | --------------------------------- |
| Authentication (AuthN) | Who are you?                       | Verify username/password          |
| Authorization (AuthZ)  | What can you access?               | Admin can delete employees        |
| Session                | Maintain logged-in state on server | ASP.NET Core Session              |
| JWT                    | Carry signed identity/claims       | Access token                      |
| OAuth 2.0              | Delegate API access                | Application accesses Employee API |
| OIDC                   | Authenticate users                 | Login using Microsoft             |
| Refresh Token          | Obtain a new access token          | Renew expired access              |
| MFA                    | Additional identity verification   | OTP / Authenticator               |

**Remember:**

- Authentication = Identity.
- Authorization = Permission.
- OAuth 2.0 = Access delegation.
- OIDC = User authentication on top of OAuth 2.0.

# **2. Understand the complete architecture**

Suppose we have:

- Angular application.
- ASP.NET Core Employee API.
- SQL Server.
- Microsoft Entra ID (formerly Azure AD).
- External Payroll Service.

## **Enterprise authentication architecture**

**Angular Application**

User login

**Microsoft Entra ID – Identity Provider**

OIDC · OAuth 2.0 · MFA · Token issuance

**ASP.NET Core Employee API – Resource Server**

JWT validation · Authentication · Authorization

**Business Logic / SQL Server**

**External Payroll Service**

Separate application identity · OAuth client credentials

Secure API-to-API communication

**Employee API**

# **3. Session-based authentication**

Let's first understand traditional session authentication.

**Scenario:** User logs in to an Employee Management application.

### **Step-by-step flow**

1. Angular sends username and password to the backend login endpoint.
2. ASP.NET Core validates the credentials.
3. Server creates a session and stores session data in a session store.
4. Server sends a session identifier through an HTTP-only cookie.
5. Browser automatically sends the cookie with subsequent requests.
6. Server reads the session ID and retrieves the user's session.
7. Server checks permissions and processes the request.

[JWT Authentication Tutorial with Node JS - DEV Community](https://images.openai.com/static-rsc-4/KNpoKJIT_tZrabJVPnWS2fddxo_Hc4JY6kA4rm_9vCjUeLoubOFphHa2y4u2cDz4NLtVWMDSatChlnKFWBkLkpwCFKRvZkumqlvoLdQywPiwQxxOq_DfMtQtwjFhm6Dmr9a2wVQNu7n2D0ADKJtRP1vKkcV4jUmZ0Jv0d79oE0I?purpose=inline)

[dev.to](https://dev.to/harsha/jwt-authentication-tutorial-with-node-js-3bgo?utm_source=chatgpt.com)

### **Example**

Login request:

```
POST /api/auth/login
```

```
{
  "email": "user@example.com",
  "password": "Password123"
}
```

Server creates session:

```
HttpContext.Session.SetString(
    "UserId", "101");
```

Server configures a cookie-backed session identifier.

Browser sends:

```
Cookie: .AspNetCore.Session=abc123
```

Server reads:

```
var userId = HttpContext.Session
    .GetString("UserId");
```

### **ASP.NET Core configuration**

```
builder.Services.AddDistributedMemoryCache();

builder.Services.AddSession(options =>
{
    options.IdleTimeout =
        TimeSpan.FromMinutes(20);

    options.Cookie.HttpOnly = true;
    options.Cookie.SecurePolicy =
        CookieSecurePolicy.Always;

    options.Cookie.SameSite =
        SameSiteMode.Lax;
});

var app = builder.Build();

app.UseSession();
```

For production, use a distributed session store such as Redis rather than in-memory storage when running multiple API instances.

**Advantages:**

- Easy to implement.
- Server controls session expiration.
- Session can be invalidated centrally.

**Disadvantages:**

- Server-side session storage required.
- Distributed session management adds complexity.
- Cookie-based authentication requires CSRF protection for state-changing requests.

# **4. JWT-based authentication**

JWT stands for JSON Web Token.

Unlike session authentication, the server does not need to look up a session for every request. It validates the token signature and claims.

### **JWT structure**

A JWT has three parts:

```
Header.Payload.Signature
```

Example decoded payload:

```
{
  "sub": "101",
  "name": "Jitendra",
  "role": "Admin",
  "iss": "https://identity.example.com",
  "aud": "employee-api",
  "exp": 1795000000
}
```

Important fields:

| Claim | Meaning         |
| ----- | --------------- |
| sub   | User identifier |
| name  | User name       |
| role  | User role       |
| iss   | Token issuer    |
| aud   | Token audience  |
| exp   | Expiration time |

Remember: JWT payload is encoded, not encrypted. Never put passwords or sensitive secrets in it.

### **JWT authentication flow**

[Building a Production-Grade JWT, JWKS and OpenID Connect Provider in Go | by Md. Mehedi Hasan | Feb, 2026 | Medium](https://images.openai.com/static-rsc-4/4nVuxPsT7WFFGwbl9LdWPrREw-KAmtxkGPLMSv_R9nc8LF00nqN5HNp3PH4HMi8bPUcm63d2AMWHQQe6EduqDDXrIPzsIDKuvT7fz7bfjjvIFZThSTs1ea1_dTrTtplJ-kFEuPJyHgSycA3gM5qkX4QG4smhJ-eQd3T4vzxaBxM?purpose=inline)

[medium.com](https://medium.com/%40rabbi.cse.sust.bd/building-a-production-grade-jwt-jwks-and-openid-connect-provider-in-go-b8c964cc7cdd?utm_source=chatgpt.com)

1. User submits credentials.
2. Identity provider validates credentials.
3. Identity provider generates a signed access token.
4. Client stores the token securely.
5. Client sends the token to the API.
6. API validates signature, issuer, audience and expiry.
7. API checks authorization policies.
8. API returns the response.

Example API request:

```
GET /api/v1/employees
Authorization: Bearer eyJhbGciOi...
```

### **ASP.NET Core JWT validation**

For a real production system, prefer tokens issued by Microsoft Entra ID or another trusted identity provider instead of implementing your own token issuer.

Install:

```
dotnet add package Microsoft.AspNetCore.Authentication.JwtBearer
```

Configuration:

```
builder.Services
    .AddAuthentication("Bearer")
    .AddJwtBearer("Bearer", options =>
    {
        options.Authority =
            "https://login.microsoftonline.com/{tenant-id}/v2.0";

        options.Audience = "{employee-api-client-id}";
    });

builder.Services.AddAuthorization();

app.UseAuthentication();
app.UseAuthorization();
```

Protect your endpoint:

```
[Authorize]
[HttpGet]
public IActionResult GetEmployees()
{
    return Ok("Employees data");
}
```

Only administrators:

```
[Authorize(Roles = "Admin")]
[HttpDelete("{id}")]
public IActionResult DeleteEmployee(int id)
{
    return NoContent();
}
```

For production, also validate the required scopes or app roles, depending on your authorization model.

# **5. OAuth 2.0 and OpenID Connect (OIDC)**

This is the most important part of modern enterprise security.

Let's understand the difference with an example.

Suppose an employee logs in using Microsoft.

- OIDC verifies the employee's identity.
- OAuth 2.0 grants permission to access the Employee API.
- JWT access token carries the permissions.
- Refresh token helps obtain new tokens without asking the employee to log in again.

## **5.1 OAuth 2.0 vs OIDC**

| OAuth 2.0                   | OIDC                           |
| --------------------------- | ------------------------------ |
| Authorization framework     | Authentication protocol        |
| Provides access tokens      | Provides ID tokens             |
| Grants access to APIs       | Establishes user identity      |
| Uses scopes and permissions | Uses identity claims           |
| Example: Read employee data | Example: Login using Microsoft |

### **Three important tokens**

**ID Token**

Identity

Tells the client who logged in.

```
{
  "sub": "101",
  "name": "Jitendra",
  "email": "user@example.com"
}
```

**Access Token**

API access

Allows the client to access a protected API.

```
{
  "aud": "employee-api",
  "scp": "Employee.Read"
}
```

**Refresh Token**

Renewal

Allows the client to obtain new tokens through the identity provider.

An opaque, sensitive credential. It is not normally a JWT and is not sent to your Employee API.

Microsoft's identity platform distinguishes these tokens by their intended purpose.&#x20;

[image](https://www.google.com/s2/favicons?domain=https://learn.microsoft.com\&sz=32)

Microsoft Learn+1



## **5.2 OIDC authentication flow – step by step**

We will use:

- Angular frontend.
- Microsoft Entra ID.
- ASP.NET Core Web API.

[Securing ASP.NET Core Apps With OIDC and Microsoft Entra ID](https://images.openai.com/static-rsc-4/p7RtBDB27rxYVuLeuCrvnBih_ij83paBaQ516ppZ956hiDwGmjoOC6j6L2U6BwOv_A1wFT5dx1w2vSjlLb5Tlw47-_QlAFlB8acJcIcEDedDXjJr9HoYc4U5hWI4xC1FEVyDRhY_4zMITlDFSP5_ZE1us1DolePG7GX35OySWPs?purpose=inline)

[juliocasal.com](https://juliocasal.com/blog/Securing-Aspnet-Core-Applications-With-OIDC-And-Microsoft-Entra-ID?utm_source=chatgpt.com)

**Step 1: User clicks Login**

Angular redirects the user to Microsoft login.

**Step 2: Microsoft authenticates the user**

Microsoft checks:

- Username and password.
- MFA, if configured.
- User account status.
- Tenant access policies.

**Step 3: Authorization code is returned**

Microsoft redirects to the registered redirect URI with an authorization code.

**Step 4: Angular exchanges the code**

Using MSAL and PKCE, Angular obtains the tokens from Microsoft.

**Step 5: Microsoft issues tokens**

- ID token for user identity.
- Access token for Employee API.
- Refresh capability according to client and identity-provider policy.

**Step 6: Angular calls Employee API**

```
GET /api/v1/employees
Authorization: Bearer <access_token>
```

**Step 7: API validates the access token**

ASP.NET Core validates:

- Signature.
- Issuer.
- Audience.
- Expiration.
- Required scopes or roles.

**Step 8: API returns employee information.**

For Angular SPAs, use Authorization Code Flow with PKCE rather than the older implicit flow.&#x20;

[image](https://www.google.com/s2/favicons?domain=https://learn.microsoft.com\&sz=32)

Microsoft Learn+1



# **6. Refresh token – how does it work?**

This is a very common interview question.

Suppose:

- Access token lifetime: 60 minutes (illustrative).
- Refresh token lifetime: governed by identity-provider policy.
- User continues working after one hour.

Without refresh token, the user would need to log in again.

### **Refresh token flow**

## **Access token renewal**

**Angular Application**

Access token expires

**Microsoft Entra ID**

Validate refresh grant and client

**New Access Token**

And potentially a replacement refresh token

**Angular calls Employee API again**

### **What happens internally?**

1. Angular detects that the access token is expired or needs renewal.
2. MSAL attempts silent token acquisition.
3. The identity library uses its cached credentials and refresh mechanism.
4. Microsoft validates the request.
5. Microsoft issues a new access token.
6. Angular calls the API using the new token.
7. If silent renewal fails due to revoked credentials, expired sessions or interaction requirements, the user is asked to sign in again.

**Important security rule:**

Do not manually store refresh tokens in browser local storage or write your own token renewal mechanism when using Entra ID and MSAL.

For public clients, refresh token rotation or sender-constrained tokens are important protections against replay attacks.&#x20;

[image](https://www.google.com/s2/favicons?domain=https://www.rfc-editor.org\&sz=32)

rfc-editor.org+1



# **7. How to implement this using Angular and .NET**

Let's translate the architecture into implementation steps.

## **Step 1: Register applications in Microsoft Entra ID**

You typically register two applications.

**Application 1: Angular Client**

Public client

- Platform: Single-page application.
- Redirect URI: `http://localhost:4200`
- Configure API permissions.
- Use MSAL Angular.

**Application 2: Employee API**

Resource server

- Expose an API.
- Application ID URI: `api://employee-api`
- Define delegated scope: `Employee.Read`
- Define delegated scope: `Employee.Write`
- Define application role: `Employee.Read.All`

The IDs above are illustrative. Use the actual application IDs and scopes generated in your tenant.

## **Step 2: Configure Angular MSAL**

Install packages:

```
npm install @azure/msal-browser @azure/msal-angular
```

Configure the MSAL instance:

```
export const msalConfig = {
  auth: {
    clientId: 'ANGULAR_CLIENT_ID',
    authority:
      'https://login.microsoftonline.com/TENANT_ID',
    redirectUri: 'http://localhost:4200'
  }
};
```

Define API scope:

```
export const apiRequest = {
  scopes: [
    'api://EMPLOYEE_API_CLIENT_ID/Employee.Read'
  ]
};
```

Your Angular service can obtain the access token using MSAL's `acquireTokenSilent()` and attach it to API requests using an HTTP interceptor.

Conceptually:

```
this.msalService.acquireTokenSilent(apiRequest)
  .subscribe(result => {
    const accessToken = result.accessToken;

    // Send token in Authorization header
  });
```

In production, configure an MSAL interceptor or equivalent centralized token handling instead of duplicating this code in each service.

## **Step 3: Configure .NET API**

Install:

```
dotnet add package Microsoft.Identity.Web
```

In `Program.cs`:

```
using Microsoft.Identity.Web;

builder.Services
    .AddAuthentication()
    .AddMicrosoftIdentityWebApi(
        builder.Configuration.GetSection("AzureAd"));

builder.Services.AddAuthorization();
```

In `appsettings.json`:

```
{
  "AzureAd": {
    "Instance": "https://login.microsoftonline.com/",
    "TenantId": "YOUR_TENANT_ID",
    "ClientId": "EMPLOYEE_API_CLIENT_ID"
  }
}
```

Middleware:

```
app.UseAuthentication();
app.UseAuthorization();
```

Protect your controller:

```
[ApiController]
[Route("api/v1/employees")]
[Authorize]
public class EmployeesController : ControllerBase
{
    [HttpGet]
    [RequiredScope("Employee.Read")]
    public IActionResult GetEmployees()
    {
        return Ok("Employee data");
    }
}
```

The API validates tokens issued for it. It does not need to know whether Angular obtained the token using an authorization code or another supported OAuth flow.

See the official implementation guidance: [ASP.NET Core JWT Bearer authentication](https://learn.microsoft.com/aspnet/core/security/authentication/configure-jwt-bearer-authentication?view=aspnetcore-10.0\&utm_source=chatgpt.com).

# **8. How can an external service access your API?**

This is particularly important for microservices and enterprise architecture interviews.

Suppose you have:

- Employee API – owned by your application.
- Payroll API – owned by an external company.
- Both need to communicate securely.

The question is: **How can Payroll API access Employee API without using an employee's username and password?**

There are three main approaches.

## **Approach 1: OAuth 2.0 Client Credentials Flow**

Recommended for service-to-service communication

Use this when an external service accesses your API without a logged-in user.

Example:

Payroll Service needs to retrieve employee information.

### **Step-by-step flow**

**External Payroll Service**

Client ID + Client Credential

**Microsoft Entra ID – Token Endpoint**

Validates application identity

**Access Token**

Application permissions / roles

**Employee API**

Validates token and application permissions

**Employee Data**

### **Step 1: Register external application**

Create an application registration for Payroll Service.

Provide:

- Client ID.
- Tenant ID.
- Certificate or client secret.
- Application permission such as `Employee.Read.All`.

Grant admin consent for the required application permission.

### **Step 2: Payroll requests an access token**

Token endpoint:

```
POST https://login.microsoftonline.com/{tenant-id}/oauth2/v2.0/token
Content-Type: application/x-www-form-urlencoded
```

Request:

```
client_id=PAYROLL_CLIENT_ID
client_secret=PAYROLL_SECRET
grant_type=client_credentials
scope=api://EMPLOYEE_API_ID/.default
```

In production, prefer certificate credentials or federated credentials over long-lived client secrets where supported.

### **Step 3: Microsoft returns an access token**

Example:

```
{
  "token_type": "Bearer",
  "expires_in": 3600,
  "access_token": "eyJhbGciOi..."
}
```

### **Step 4: Payroll calls your Employee API**

```
GET /api/v1/employees
Authorization: Bearer eyJhbGciOi...
```

### **Step 5: Employee API validates token**

It verifies:

- Token signature.
- Issuer.
- Audience.
- Expiration.
- Application role / permission.

For example:

```
[Authorize(
    AuthenticationSchemes = "Bearer",
    Policy = "PayrollAccess")]
[HttpGet("payroll-data")]
public IActionResult GetPayrollData()
{
    return Ok("Employee data for payroll");
}
```

Configure the policy:

```
builder.Services.AddAuthorization(options =>
{
    options.AddPolicy("PayrollAccess", policy =>
    {
        policy.RequireRole("Employee.Read.All");
    });
});
```

The API must also be configured to map and validate the appropriate application-role claims from its issued access tokens.

**Important:** Client credentials flow does not use a user refresh token. When its access token expires, the application obtains another access token using its client credentials.

Official reference: [Microsoft identity platform – application types and flows](https://learn.microsoft.com/en-us/azure/active-directory/develop/web-api?utm_source=chatgpt.com).

## **Approach 2: On-Behalf-Of (OBO) Flow**

When an external/downstream API needs user context

Suppose:

1. Jitendra logs into Angular.
2. Angular calls Employee API.
3. Employee API needs to call Payroll API on behalf of Jitendra.
4. Payroll API should know which user initiated the request.

In this scenario, use OAuth On-Behalf-Of flow.

## **On-Behalf-Of flow**

Angular – User login

Employee API – Receives user token

Microsoft Entra ID – Token exchange

Payroll API – Receives delegated token

Why use OBO?

- Preserves user identity.
- Downstream API receives delegated permissions.
- Access can be limited by user permissions.
- Provides better user-level auditing.

For .NET, Microsoft Identity Web supports OBO token acquisition.

Reference: [Microsoft Identity Web – On-Behalf-Of flow](https://learn.microsoft.com/en-us/aspnet/core/security/authentication/configure-jwt-bearer-authentication?view=aspnetcore-10.0\&utm_source=chatgpt.com).

## **Approach 3: API Key**

Suppose an external vendor has a simple integration with your application.

You can issue an API key.

```
GET /api/v1/employees
X-API-Key: YOUR_API_KEY
```

Your API validates the key against a secure credential store.

However:

- API keys identify the calling application, not an individual user.
- They don't inherently provide fine-grained authorization.
- They should be rotated and revoked.
- They must only be transmitted over HTTPS.

For enterprise integrations, prefer OAuth 2.0 client credentials over relying solely on API keys.

# **9. Which authentication method should you choose?**

This is an important architecture decision.

| Scenario                                    | Recommended approach                   |
| ------------------------------------------- | -------------------------------------- |
| Traditional server-rendered web application | Cookie authentication + OIDC           |
| Angular SPA                                 | OIDC + OAuth Authorization Code + PKCE |
| Mobile application                          | OAuth Authorization Code + PKCE        |
| Angular calling .NET API                    | JWT access token                       |
| Microservice calling microservice           | Client Credentials                     |
| API calling another API for a user          | OBO                                    |
| Simple legacy vendor integration            | API key with additional controls       |
| Enterprise SSO                              | Microsoft Entra ID + OIDC              |

## **10. Session vs JWT vs OAuth/OIDC**

| Feature          | Session                                | JWT                            | OAuth/OIDC                                 |
| ---------------- | -------------------------------------- | ------------------------------ | ------------------------------------------ |
| Main purpose     | Maintain server-side login state       | Token-based identity/claims    | Authentication and delegated authorization |
| Server storage   | Session state                          | Not necessarily                | Identity provider manages grants           |
| Stateless API    | No, typically                          | Yes, during validation         | Depends on implementation                  |
| Revocation       | Destroy session                        | Requires additional strategy   | Revoke grants/tokens according to provider |
| External service | Not ideal                              | Possible with a trusted issuer | Client credentials                         |
| Scaling          | Distributed session store often needed | Easy to scale validation       | Identity provider handles token issuance   |

One important distinction: JWT is a token format, not an authentication protocol. OAuth and OIDC define the protocols and flows that can issue and use tokens.

# **11. Production security checklist for a Technical Architect**

These are the security controls I would include in an enterprise API.

| Security area      | Recommended implementation                                |
| ------------------ | --------------------------------------------------------- |
| Transport security | HTTPS / TLS                                               |
| Authentication     | Entra ID / OIDC                                           |
| Authorization      | Role-based + policy-based                                 |
| Token validation   | Issuer + audience + signature + expiry                    |
| Access token       | Short-lived                                               |
| Refresh token      | Secure storage + rotation/replay protection               |
| Browser security   | HTTP-only, Secure cookies where applicable                |
| CSRF               | Anti-forgery protection for cookie-authenticated requests |
| CORS               | Allow only trusted frontend origins                       |
| API protection     | Rate limiting                                             |
| Secrets            | Azure Key Vault                                           |
| Service identity   | Managed Identity / certificates                           |
| Auditing           | Log user ID, client ID and correlation ID                 |
| Monitoring         | Application Insights                                      |
| Network            | Private endpoints where appropriate                       |

**One additional architectural recommendation:** For browser applications, consider the Backend-for-Frontend (BFF) pattern. Angular communicates with a trusted ASP.NET Core backend using a secure HTTP-only cookie, while the backend manages OAuth tokens server-side. This reduces exposure of access and refresh tokens to browser JavaScript.

# **12. Interview questions to test your understanding**

**Security mock interview**

1 of 6

## **What happens when a JWT access token expires?**

Reveal answer

PreviousNext

## **13. Official documentation for further learning**

1. [Microsoft – Authorization Code Flow and PKCE](https://learn.microsoft.com/en-us/entra/identity-platform/v2-oauth2-auth-code-flow?utm_source=chatgpt.com)
2. [ASP.NET Core JWT Bearer Authentication](https://learn.microsoft.com/aspnet/core/security/authentication/configure-jwt-bearer-authentication?view=aspnetcore-10.0\&utm_source=chatgpt.com)
3. [ASP.NET Core OpenID Connect](https://learn.microsoft.com/en-us/aspnet/core/security/authentication/configure-oidc-web-authentication?utm_source=chatgpt.com)
4. [OAuth 2.0 Security Best Current Practice (RFC 9700)](https://www.rfc-editor.org/rfc/rfc9700.html?utm_source=chatgpt.com)

**My recommendation for your Lead / Technical Architect interviews:** Be prepared to draw the Angular → Entra ID → .NET API flow, explain refresh token renewal, and then draw a separate Payroll Service → Entra ID → Employee API client credentials flow. These two diagrams demonstrate that you understand both user authentication and machine-to-machine security.

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
