For a **Senior .NET / Architect interview**, the easiest way to understand these three is:

> **OAuth 2.0 = Authorization framework**
> **OIDC = Authentication layer built on OAuth 2.0**
> **JWT = Token format commonly used to carry identity/claims**

They are related, but they are **not the same thing**.

---

# 1. First understand the problem

Suppose you have this architecture:

```text
Angular Application
        |
        | Login
        v
Identity Provider
(Entra ID / Auth0 / Keycloak)
        |
        | Access Token
        v
.NET Web API
        |
        v
Order Service
```

You don't want your Angular application to send:

```text
username + password
```

to every API.

Instead:

1. User logs in with the Identity Provider.
2. Identity Provider authenticates the user.
3. It issues tokens.
4. Angular sends an **access token** to your API.
5. API validates the token.
6. API allows/denies access.

---

# 2. What is OAuth 2.0?

**OAuth 2.0 is primarily about authorization.**

It answers:

> **"What is this application allowed to access?"**

For example:

```text
User
  |
  | Give Angular permission
  v
Identity Provider
  |
  | Access Token
  v
Angular
  |
  | Bearer Access Token
  v
Order API
```

The API might receive:

```http
GET /api/orders
Authorization: Bearer eyJhbGciOi...
```

The access token may contain claims such as:

```json
{
  "sub": "12345",
  "scope": "orders.read orders.write",
  "aud": "order-api"
}
```

The API can determine:

```text
Does this token belong to the Order API?
        |
        +-- Yes

Does it have orders.read?
        |
        +-- Yes

Allow request
```

### Important

OAuth 2.0 doesn't require JWT.

An OAuth access token can technically be an **opaque token**:

```text
A8F9C123XYZ...
```

or a JWT:

```text
eyJhbGciOiJIUzI1NiIs...
```

So:

> **OAuth 2.0 is a protocol/framework, while JWT is a token format.**

---

# 3. What is OIDC?

OIDC = **OpenID Connect**

OIDC adds **authentication** on top of OAuth 2.0.

It answers:

> **"Who is the user?"**

OAuth:

```text
What can this application access?
```

OIDC:

```text
Who authenticated?
```

OIDC introduces an important token:

### ID Token

Usually a JWT.

Example:

```json
{
  "iss": "https://login.example.com",
  "sub": "12345",
  "aud": "my-angular-app",
  "name": "Jitendra",
  "email": "user@example.com"
}
```

This tells the client:

```text
The Identity Provider authenticated this user.

User ID = 12345
Name = Jitendra
```

---

# 4. What is JWT?

JWT = **JSON Web Token**

It is a compact token format.

A JWT normally looks like:

```text
xxxxx.yyyyy.zzzzz
```

Three parts:

```text
HEADER.PAYLOAD.SIGNATURE
```

For example:

```text
eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9
.
eyJzdWIiOiIxMjMiLCJyb2xlIjoiQWRtaW4ifQ
.
signature
```

### Header

```json
{
  "alg": "RS256",
  "typ": "JWT"
}
```

### Payload

```json
{
  "sub": "123",
  "name": "Jitendra",
  "role": "Admin",
  "scope": "orders.read",
  "iss": "https://identity-server",
  "aud": "order-api",
  "exp": 1790000000
}
```

### Signature

The Identity Provider signs the token.

Conceptually:

```text
Header + Payload
       |
       v
   Signing Key
       |
       v
   Signature
```

The API can validate:

```text
Was this token really issued by the trusted Identity Provider?
Has it expired?
Was it issued for my API?
Is the signature valid?
```

---

# 5. How all three work together

This is the most important interview concept.

```text
                OAuth 2.0
        Authorization Framework
                  |
                  |
             +----+----+
             |         |
             v         v
        Access Token  Scopes
             |
             |
           JWT
      Token Format
             |
             |
            OIDC
    Authentication Layer
             |
             v
          ID Token
```

A more practical architecture:

```text
                 ┌───────────────────────┐
                 │   Identity Provider   │
                 │                       │
                 │ Entra ID / Auth0 etc. │
                 └───────────┬───────────┘
                             │
                    Authenticate User
                             │
                    ┌────────┴────────┐
                    │                 │
                 ID Token        Access Token
                  (JWT)           (often JWT)
                    │                 │
                    v                 v
                Angular          .NET Web API
```

---

# 6. Real-world example

Let's take:

```text
Angular
    |
    | Login
    v
Microsoft Entra ID
    |
    | Access Token
    v
.NET Order API
```

Suppose Jitendra opens:

```text
https://myshop.com
```

Angular needs the user to authenticate.

It redirects the browser:

```text
Angular
   |
   | Authorization Request
   v
Entra ID
```

Something conceptually like:

```http
GET /authorize?
    client_id=angular-client
    &response_type=code
    &redirect_uri=https://myshop.com/callback
    &scope=openid orders.read
```

Notice:

```text
openid
```

That indicates **OIDC**.

---

# 7. User logs in

The user enters credentials at the Identity Provider.

```text
              Entra ID

          ┌──────────────┐
          │ Login        │
          │              │
          │ User         │
          │ Password     │
          └──────┬───────┘
                 │
              Success
                 │
                 v
```

The application doesn't need to directly handle the user's password.

---

# 8. Authorization Code

For a modern SPA, the recommended flow is generally:

**Authorization Code Flow with PKCE**

The Identity Provider sends a code back:

```text
Entra ID
   |
   | authorization code
   v
Angular
```

Example:

```text
https://myshop.com/callback?code=ABC123
```

The code itself isn't the access token.

---

# 9. Code is exchanged for tokens

The client completes the flow using PKCE.

Conceptually:

```text
Angular
   |
   | authorization code + PKCE verifier
   v
Identity Provider
   |
   +------------------+
   |                  |
   v                  v
ID Token          Access Token
```

For example:

```text
ID Token
   ↓
Who is the user?

Access Token
   ↓
What can the application access?
```

---

# 10. Angular calls .NET API

Angular doesn't normally send the ID token to your API for authorization.

It sends the **access token**.

```http
GET /api/orders

Authorization: Bearer eyJhbGciOiJSUzI1NiIs...
```

Flow:

```text
Angular
   |
   | Bearer Access Token
   v
.NET API
```

---

# 11. What does .NET API do?

ASP.NET Core JWT authentication middleware validates the access token.

Conceptually:

```text
             Access Token
                  |
                  v
        ┌───────────────────┐
        │ JWT Middleware    │
        └─────────┬─────────┘
                  |
        ┌─────────┼─────────┐
        |         |         |
        v         v         v
     Signature  Expiry    Issuer
      valid?     valid?    valid?
        |         |         |
        +---------+---------+
                  |
                Valid
                  |
                  v
             Controller
```

---

# 12. .NET implementation

For a JWT access token:

```csharp
builder.Services
    .AddAuthentication(JwtBearerDefaults.AuthenticationScheme)
    .AddJwtBearer(options =>
    {
        options.Authority = "https://login.microsoftonline.com/{tenant-id}/v2.0";

        options.Audience = "api://order-api";
    });
```

Then:

```csharp
builder.Services.AddAuthorization();
```

And middleware:

```csharp
app.UseAuthentication();
app.UseAuthorization();
```

Controller:

```csharp
[ApiController]
[Route("api/orders")]
public class OrdersController : ControllerBase
{
    [HttpGet]
    [Authorize]
    public IActionResult GetOrders()
    {
        return Ok("Orders");
    }
}
```

Now:

```text
No token
   ↓
401 Unauthorized
```

Valid token:

```text
Valid token
   ↓
Controller
   ↓
200 OK
```

---

# 13. Authentication vs Authorization

This is a **very common interview question**.

### Authentication

> Who are you?

Example:

```text
Jitendra successfully logged in.
```

OIDC handles this.

---

### Authorization

> What are you allowed to do?

Example:

```text
Jitendra can read orders.
Jitendra cannot delete orders.
```

OAuth 2.0 provides the authorization framework.

---

# 14. Scope example

Suppose access token contains:

```json
{
  "sub": "12345",
  "scope": "orders.read orders.write"
}
```

Your API can protect endpoints.

```csharp
[Authorize(Policy = "OrdersRead")]
[HttpGet]
public IActionResult GetOrders()
{
    ...
}
```

Configure:

```csharp
builder.Services.AddAuthorization(options =>
{
    options.AddPolicy("OrdersRead", policy =>
    {
        policy.RequireClaim("scope", "orders.read");
    });
});
```

Now:

```text
orders.read
      |
      v
GET /orders
      |
      v
Allowed
```

But:

```text
orders.delete
      |
      v
DELETE /orders
      |
      v
Forbidden
```

assuming the token lacks the required permission.

---

# 15. Roles vs Scopes

Another important Architect-level distinction.

### Scope

Usually represents what a **client/application is authorized to do** on behalf of a user.

Example:

```text
orders.read
orders.write
```

### Role

Usually represents what the **user is allowed to do**.

Example:

```text
Admin
Manager
Customer
```

JWT could contain:

```json
{
  "sub": "123",
  "roles": [
    "OrderManager"
  ],
  "scope": "orders.read orders.write"
}
```

Then:

```csharp
[Authorize(Roles = "OrderManager")]
```

---

# 16. Why shouldn't API trust any JWT?

Because a JWT being syntactically valid doesn't mean it's trustworthy.

The API must validate things such as:

```text
1. Signature
2. Issuer (iss)
3. Audience (aud)
4. Expiration (exp)
5. Not-before (nbf), when applicable
6. Relevant scopes/roles
```

For example:

```json
{
  "iss": "https://trusted-idp.com",
  "aud": "order-api",
  "exp": 1790000000
}
```

Your API should verify:

```text
iss == trusted identity provider
aud == order-api
exp > current time
signature == valid
```

---

# 17. Access Token vs ID Token

This distinction is **extremely important**.

|             | ID Token           | Access Token             |
| ----------- | ------------------ | ------------------------ |
| Purpose     | Authentication     | Authorization            |
| Answers     | Who is the user?   | What can the app access? |
| Used by     | Client/application | API/resource server      |
| OIDC        | Yes                | No, generally OAuth      |
| JWT         | Usually            | Often, but not required  |
| Sent to API | Normally no        | Yes                      |

Remember:

```text
ID Token
   ↓
Client learns identity

Access Token
   ↓
API grants access
```

---

# 18. Complete enterprise flow

For your .NET/Azure background, imagine:

```text
                ┌─────────────────┐
                │   Angular SPA   │
                └────────┬────────┘
                         │
                         │ 1. Login
                         v
                ┌─────────────────┐
                │   Entra ID      │
                │ Identity        │
                │ Provider        │
                └────────┬────────┘
                         │
                   2. Authenticate
                         │
                         v
                  Authorization
                     Code
                         │
                         v
                ┌─────────────────┐
                │ Angular + PKCE  │
                └────────┬────────┘
                         │
                  3. Get tokens
                         │
              ┌──────────┴─────────┐
              │                    │
           ID Token           Access Token
              │                    │
              │                    v
              │             ┌──────────────┐
              │             │ .NET API     │
              │             └──────┬───────┘
              │                    │
              │             Validate JWT
              │                    │
              │                    v
              │             Authorization
              │                    │
              │                    v
              │             Order Service
              │                    │
              │                    v
              │             Azure Service Bus
              │
              v
         User identity
```

---

# 19. Where JWT actually fits

Don't think:

```text
JWT = Authentication
```

Instead:

```text
JWT
 ↓
Token format
```

It can carry claims:

```json
{
    "sub": "12345",
    "name": "Jitendra",
    "roles": ["Admin"],
    "scope": "orders.read",
    "iss": "https://identity-provider",
    "aud": "order-api",
    "exp": 1790000000
}
```

JWT itself doesn't magically authenticate or authorize the user.

The **issuer, signing mechanism, validation rules, and application policies** make the token meaningful.

---

# 20. Simple interview answer

If an interviewer asks:

> **"Explain JWT, OAuth 2.0 and OIDC."**

You can say:

> **OAuth 2.0 is an authorization framework used to obtain access tokens and control what an application can access. OIDC is an authentication protocol built on top of OAuth 2.0 that adds user identity and the ID token. JWT is a token format commonly used for ID tokens and access tokens.**
>
> **For example, in an Angular + .NET application, the user authenticates with Entra ID using OIDC Authorization Code Flow with PKCE. The client receives an ID token for identity and an access token for calling the .NET API. Angular sends the access token as a Bearer token. ASP.NET Core validates the token's signature, issuer, audience and expiration, and then authorization policies check scopes or roles before allowing access to the API.**

### One-line memory trick

```text
OIDC → WHO are you?
OAuth → WHAT can you access?
JWT → HOW is the token represented?
```

And the complete flow:

```text
User
 ↓
OIDC Authentication
 ↓
Authorization Code + PKCE
 ↓
Tokens
 ├── ID Token (identity)
 └── Access Token (API authorization)
              ↓
          JWT (commonly)
              ↓
          .NET API
              ↓
       Validate + Authorize
              ↓
           Resource
```

This is the level of distinction I would use in a **Senior/Staff/Architect interview**, because it prevents the common mistake of saying **"OAuth is authentication"** or **"JWT is an authentication protocol."**
