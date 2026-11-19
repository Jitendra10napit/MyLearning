# **LLD: API Design – Step-by-Step Approach**

For a Lead Software Engineer / Technical Architect interview, you should explain API design in a structured way, starting from requirements and ending with implementation details.

Let's take a practical example:

**Design an Employee Management API using ASP.NET Core Web API, SQL Server and Azure.**

The goal is to demonstrate how you convert business requirements into an API architecture and Low-Level Design (LLD).

## **Step 1: Understand the requirements**

Before writing code, understand what the business wants.

### **1.1 Functional requirements (What the system should do)**

Functional requirements describe the features and business operations.

| ID  | Requirement                            |
| --- | -------------------------------------- |
| FR1 | Create an employee                     |
| FR2 | Get employee by ID                     |
| FR3 | Get all employees                      |
| FR4 | Update employee details                |
| FR5 | Delete employee                        |
| FR6 | Search employees by name or department |
| FR7 | Support pagination and sorting         |
| FR8 | Assign employees to departments        |

### **1.2 Non-functional requirements (How the system should perform)**

Non-functional requirements describe quality attributes and constraints.

| Requirement      | Design approach                          |
| ---------------- | ---------------------------------------- |
| Performance      | Redis caching, optimized SQL queries     |
| Scalability      | Azure App Service scale-out              |
| Security         | OAuth 2.0 / JWT authentication           |
| Availability     | Multiple instances, health checks        |
| Reliability      | Retry, timeout, circuit breaker          |
| Maintainability  | SOLID, Clean Architecture                |
| Observability    | Application Insights, structured logging |
| Data consistency | SQL transactions                         |
| API versioning   | `/api/v1/employees`                      |
| Rate limiting    | ASP.NET Core rate limiter                |

**Interview tip:** Functional requirements tell us what to build. Non-functional requirements tell us how well the system must work.

## **Step 2: Identify API endpoints**

Convert functional requirements into REST API endpoints.

| Method | Endpoint                   | Purpose          |
| ------ | -------------------------- | ---------------- |
| POST   | `/api/v1/employees`        | Create employee  |
| GET    | `/api/v1/employees/{id}`   | Get employee     |
| GET    | `/api/v1/employees`        | Get employees    |
| PUT    | `/api/v1/employees/{id}`   | Update employee  |
| DELETE | `/api/v1/employees/{id}`   | Delete employee  |
| GET    | `/api/v1/employees/search` | Search employees |

Example request:

```
POST /api/v1/employees
Content-Type: application/json
Authorization: Bearer <token>
```

```
{
  "firstName": "Jitendra",
  "lastName": "Napit",
  "email": "jitendra@example.com",
  "departmentId": 1
}
```

Example response:

```
{
  "id": 101,
  "firstName": "Jitendra",
  "lastName": "Napit",
  "email": "jitendra@example.com",
  "departmentId": 1
}
```

## **Step 3: Define the architecture**

Choose an architecture based on complexity, scalability and maintainability.

For this example, use Clean Architecture with layered separation.

## **Employee API – Architecture**

**Client / Consumer**

Angular / Mobile / External Applications

**API Layer – ASP.NET Core**

Controller · Authentication · Middleware · Validation

**Application Layer**

Services · DTOs · Interfaces · Business Logic · CQRS

**Domain Layer**

Entities · Value Objects · Business Rules

**Infrastructure Layer**

EF Core · Repository · SQL Server · Redis · Azure Service Bus

**Why this architecture?**

- Separation of concerns.
- Business logic is independent of database implementation.
- Easy unit testing.
- Easier to maintain and extend.
- Supports SOLID principles.

## **Step 4: Design the database**

Identify entities and relationships from the requirements.

**Employee table**

| Column       | Data type     | Constraint  |
| ------------ | ------------- | ----------- |
| Id           | INT           | Primary Key |
| FirstName    | NVARCHAR(100) | NOT NULL    |
| LastName     | NVARCHAR(100) | NOT NULL    |
| Email        | NVARCHAR(150) | UNIQUE      |
| DepartmentId | INT           | Foreign Key |
| CreatedAt    | DATETIME2     | NOT NULL    |
| UpdatedAt    | DATETIME2     | NULL        |

**Department table**

| Column | Data type     | Constraint  |
| ------ | ------------- | ----------- |
| Id     | INT           | Primary Key |
| Name   | NVARCHAR(100) | NOT NULL    |

Relationship:

- One department can have multiple employees.
- One employee belongs to one department.

```
Department (1) -------- (*) Employee
```

Add indexes based on query patterns:

```
CREATE UNIQUE INDEX IX_Employee_Email
ON Employees(Email);

CREATE INDEX IX_Employee_DepartmentId
ON Employees(DepartmentId);
```

## **Step 5: Design classes and interfaces (LLD)**

Now decide which classes are required and what responsibility each class has.

**Suggested project structure**

```
EmployeeManagement
│
├── Employee.API
│   ├── Controllers
│   │   └── EmployeesController.cs
│   ├── Middleware
│   │   └── ExceptionMiddleware.cs
│   └── Program.cs
│
├── Employee.Application
│   ├── Interfaces
│   │   └── IEmployeeService.cs
│   ├── Services
│   │   └── EmployeeService.cs
│   └── DTOs
│       ├── CreateEmployeeDto.cs
│       └── EmployeeResponseDto.cs
│
├── Employee.Domain
│   └── Entities
│       └── Employee.cs
│
└── Employee.Infrastructure
    ├── Data
    │   └── AppDbContext.cs
    └── Repositories
        ├── IEmployeeRepository.cs
        └── EmployeeRepository.cs
```

### **5.1 Domain entity**

```
public class Employee
{
    public int Id { get; set; }
    public string FirstName { get; set; }
    public string LastName { get; set; }
    public string Email { get; set; }
    public int DepartmentId { get; set; }
}
```

### **5.2 DTO**

Never expose database entities directly through your API.

```
public class CreateEmployeeDto
{
    [Required]
    public string FirstName { get; set; }

    [Required]
    public string LastName { get; set; }

    [Required]
    [EmailAddress]
    public string Email { get; set; }

    public int DepartmentId { get; set; }
}
```

### **5.3 Repository interface**

```
public interface IEmployeeRepository
{
    Task<Employee> GetByIdAsync(int id);

    Task<List<Employee>> GetAllAsync();

    Task AddAsync(Employee employee);

    Task UpdateAsync(Employee employee);

    Task DeleteAsync(Employee employee);
}
```

### **5.4 Service interface**

```
public interface IEmployeeService
{
    Task<EmployeeResponseDto> GetByIdAsync(int id);

    Task<int> CreateAsync(CreateEmployeeDto dto);
}
```

### **5.5 Service implementation**

```
public class EmployeeService : IEmployeeService
{
    private readonly IEmployeeRepository _repository;

    public EmployeeService(IEmployeeRepository repository)
    {
        _repository = repository;
    }

    public async Task<int> CreateAsync(CreateEmployeeDto dto)
    {
        var employee = new Employee
        {
            FirstName = dto.FirstName,
            LastName = dto.LastName,
            Email = dto.Email,
            DepartmentId = dto.DepartmentId
        };

        await _repository.AddAsync(employee);

        return employee.Id;
    }

    public async Task<EmployeeResponseDto> GetByIdAsync(int id)
    {
        var employee = await _repository.GetByIdAsync(id);

        if (employee == null)
            throw new KeyNotFoundException("Employee not found");

        return new EmployeeResponseDto
        {
            Id = employee.Id,
            FirstName = employee.FirstName,
            LastName = employee.LastName,
            Email = employee.Email
        };
    }
}
```

### **5.6 Controller**

```
[ApiController]
[Route("api/v1/employees")]
[Authorize]
public class EmployeesController : ControllerBase
{
    private readonly IEmployeeService _service;

    public EmployeesController(IEmployeeService service)
    {
        _service = service;
    }

    [HttpPost]
    public async Task<IActionResult> Create(
        CreateEmployeeDto dto)
    {
        var id = await _service.CreateAsync(dto);

        return CreatedAtAction(
            nameof(GetById),
            new { id },
            new { id });
    }

    [HttpGet("{id}")]
    public async Task<IActionResult> GetById(int id)
    {
        var employee = await _service.GetByIdAsync(id);

        return Ok(employee);
    }
}
```

## **Step 6: Decide which design patterns to use**

Don't use every design pattern in every project. Select patterns according to actual requirements.

| Pattern              | Where to use             | Why                                    |
| -------------------- | ------------------------ | -------------------------------------- |
| Repository           | Data access              | Abstract database operations           |
| Dependency Injection | Services                 | Loose coupling                         |
| Factory              | Object creation          | Centralize complex creation            |
| Strategy             | Different business rules | Switch algorithms dynamically          |
| Decorator            | Caching/logging          | Add behavior without modifying service |
| Mediator             | CQRS                     | Separate commands and queries          |
| Specification        | Complex filtering        | Reusable query conditions              |
| Unit of Work         | Multiple DB operations   | Coordinate transactions                |

**Example: When should we use Strategy Pattern?**

Suppose employee salary calculation differs by employee type.

```
public interface ISalaryStrategy
{
    decimal CalculateSalary(Employee employee);
}

public class PermanentSalaryStrategy : ISalaryStrategy
{
    public decimal CalculateSalary(Employee employee)
    {
        return 50000;
    }
}

public class ContractSalaryStrategy : ISalaryStrategy
{
    public decimal CalculateSalary(Employee employee)
    {
        return 30000;
    }
}
```

We can inject the appropriate strategy based on employee type.

## **Step 7: Define the API request flow**

This is one of the most important parts of an LLD interview.

Example: User creates an employee.

## **POST Employee – Request Flow**

1. **Client sends HTTP request**

   Angular sends employee details with JWT token.
2. **Authentication middleware**

   Validates JWT token and identifies the user.
3. **Authorization**

   Checks whether the user has permission to create employees.
4. **Controller**

   Receives request and validates DTO.
5. **Application service**

   Executes business validations and rules.
6. **Repository**

   Checks database constraints and persists employee through EF Core.
7. **SQL Server**

   Inserts employee and returns generated ID.
8. **API response**

   Returns HTTP 201 Created.

## **Step 8: Handle errors and response codes**

Design a consistent API response strategy.

| HTTP status | Scenario                   |
| ----------- | -------------------------- |
| 200         | Successful GET / UPDATE    |
| 201         | Employee created           |
| 204         | Successful DELETE          |
| 400         | Invalid request            |
| 401         | Unauthorized               |
| 403         | Forbidden                  |
| 404         | Employee not found         |
| 409         | Duplicate email / conflict |
| 429         | Too many requests          |
| 500         | Internal server error      |

Use centralized exception handling instead of writing try-catch in every controller.

ASP.NET Core provides `IExceptionHandler` and `ProblemDetails` for this purpose.

Example response:

```
{
  "type": "about:blank",
  "title": "Employee not found",
  "status": 404,
  "detail": "Employee with ID 101 does not exist."
}
```

## **Step 9: Implement non-functional requirements**

Now explain how you would satisfy production-level requirements.

| Requirement       | Implementation                           |
| ----------------- | ---------------------------------------- |
| Authentication    | Azure AD / Microsoft Entra ID, JWT       |
| Authorization     | Role-based or policy-based authorization |
| Performance       | Redis cache, SQL indexing                |
| Scalability       | Azure App Service autoscaling            |
| Reliability       | Retry, timeout, circuit breaker          |
| Monitoring        | Azure Monitor, Application Insights      |
| Logging           | Serilog, structured logs                 |
| Rate limiting     | ASP.NET Core rate limiting               |
| Availability      | Multiple App Service instances           |
| Deployment        | Azure DevOps CI/CD                       |
| Configuration     | Azure Key Vault                          |
| API documentation | Swagger / OpenAPI                        |

### **Example: Caching**

For frequently requested employee details:

```
GET Employee
     |
     v
Check Redis Cache
     |
     |---- Cache Hit ----> Return Data
     |
     |---- Cache Miss
                |
                v
          SQL Server
                |
                v
          Store in Redis
                |
                v
          Return Data
```

### **Example: Reliability**

Suppose SQL Server or another downstream service is temporarily unavailable.

- Retry: Retry transient failures with exponential backoff.
- Circuit Breaker: Temporarily stop calling an unhealthy dependency.
- Timeout: Prevent requests from waiting indefinitely.
- Idempotency: Prevent duplicate creation when clients retry POST requests.

These patterns improve reliability but should be configured according to the downstream service and operation.

## **Step 10: Define testing strategy**

As a Lead Engineer, explain how you ensure the API is reliable.

| Testing type        | What to test                     | Tool                  |
| ------------------- | -------------------------------- | --------------------- |
| Unit testing        | Service business logic           | xUnit, Moq            |
| Integration testing | API + database                   | WebApplicationFactory |
| API testing         | Endpoints and responses          | Postman               |
| Performance testing | Concurrent requests              | JMeter                |
| Security testing    | Authentication and authorization | OWASP tools           |

Example test cases:

1. Create employee with valid details → 201.
2. Create employee with invalid email → 400.
3. Create employee with duplicate email → 409.
4. Get employee with valid ID → 200.
5. Get employee with invalid ID → 404.
6. Call API without token → 401.
7. Call API without required permission → 403.

## **Step 11: Deployment architecture on Azure**

## **Azure deployment architecture**

**Angular Application**

**Azure Front Door / API Management**

Routing · API policies · Rate limiting

**Azure App Service**

ASP.NET Core Web API · Multiple instances

**Azure SQL**

Persistent data

**Azure Redis**

Distributed cache

**Cross-cutting Azure Services**

Key Vault · Application Insights · Azure Monitor · Service Bus

For CI/CD:

1. Developer pushes code to Git.
2. Azure DevOps pipeline builds the application.
3. Run unit and integration tests.
4. Perform code quality and security checks.
5. Build Docker image if using containers.
6. Deploy to Azure App Service.
7. Run smoke tests.
8. Monitor logs and application health.

## **Step 12: Explain your design decisions in the interview**

Interviewers often ask why you selected a particular technology or pattern.

| Interview question        | Suggested answer                                                 |
| ------------------------- | ---------------------------------------------------------------- |
| Why Clean Architecture?   | To separate business logic from infrastructure.                  |
| Why Repository Pattern?   | To abstract database operations.                                 |
| Why DTO?                  | To avoid exposing internal entities and control API contracts.   |
| Why Dependency Injection? | To achieve loose coupling and testability.                       |
| Why Redis?                | To reduce repeated database calls.                               |
| Why CQRS?                 | When read and write operations need different models or scaling. |
| Why API versioning?       | To support existing clients while introducing changes.           |
| Why Azure Service Bus?    | To process asynchronous operations and decouple services.        |
| Why middleware?           | To handle cross-cutting concerns centrally.                      |
| Why pagination?           | To avoid loading large datasets into memory.                     |

## **Step 13: Your final interview explanation (2-minute answer)**

You can use this answer when the interviewer asks:

**"How do you approach API design from requirements to implementation?"**

I follow a step-by-step approach when designing APIs.

**1. Requirement Analysis:** First, I gather functional and non-functional requirements. Functional requirements define business operations, while non-functional requirements cover performance, scalability, security, availability and maintainability.

**2. API Design:** I identify REST endpoints, HTTP methods, request and response DTOs, status codes, pagination, filtering and API versioning.

**3. Architecture Design:** Based on the complexity, I select an appropriate architecture such as Clean Architecture with separate API, Application, Domain and Infrastructure layers.

**4. Database Design:** I identify entities, relationships, primary keys, foreign keys, indexes and constraints.

**5. LLD:** I define classes, interfaces, services, repositories and their responsibilities. I apply SOLID principles and appropriate design patterns such as Repository, Strategy, Factory and Dependency Injection.

**6. Security:** I implement OAuth 2.0 / JWT authentication, role-based authorization, input validation and secure configuration using Azure Key Vault.

**7. Non-functional Requirements:** I implement caching using Redis, logging and monitoring using Application Insights, and resilience using retry, timeout and circuit breaker patterns.

**8. Testing and Deployment:** I write unit and integration tests, validate APIs using Postman and deploy using Azure DevOps CI/CD pipelines.

Finally, I monitor the application in production and continuously optimize performance and reliability.

## **Remember this framework for any LLD/API interview**

# **R → A → D → C → P → S → T → D**

Your API design interview checklist

**R**

**Requirements**

Functional + Non-functional

**A**

**API Design**

Endpoints + HTTP methods + DTOs

**D**

**Database Design**

Entities + Relationships + Indexes

**C**

**Class Design**

Interfaces + Services + Repository

**P**

**Patterns**

SOLID + Design Patterns

**S**

**Security & Scalability**

JWT + Cache + Resilience

**T**

**Testing**

Unit + Integration + Performance

**D**

**Deployment**

Azure + CI/CD + Monitoring

## **Practice question**

**LLD interview practice**

1 / 4

**You have 10 million employees. How would you design the GET Employees API?**

Show answer

PreviousNext question

**My recommendation:** For a Technical Architect / Lead Engineer interview, don't just explain which patterns you use. Explain the requirement behind the decision, the alternatives you considered, and why your selected design is appropriate. That demonstrates architectural thinking rather than just coding knowledge.
