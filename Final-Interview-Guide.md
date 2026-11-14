## Important Interview query: Get top 3 highest paid employees per department using LINQ

   var result = await context.Employees
    .GroupBy(e => new { e.DepartmentId, e.Department.Name })
    .Select(g => new
    {
        Department = g.Key.Name,
        TopEmployees = g.OrderByDescending(e => e.Salary)
                        .Take(3)
                        .Select(e => new { e.Name, e.Salary })
                        .ToList()
    })
    .ToListAsync();

    
Here's the order to write it in a 30–45 minute live session. Each step says what to write and what to say while you write it.

**Step 0 – Clarify first (2 min, don't skip)**

Ask before you type:
- "Which entities? Employee and Department?"
- "CRUD only, or reports too?"
- "Should I use EF Core with SQL Server, or in-memory is fine?"
- "Do you need auth, or should I focus on the API?"

Say: *"I'll build it in layers: entities, DbContext, DTOs, service, controller, then cross-cutting concerns."*

**Step 1 – Entities (3 min)**

```csharp
public class Department {
    public int Id { get; set; }
    public string Name { get; set; } = "";
    public List<Employee> Employees { get; set; } = new();
}

public class Employee {
    public int Id { get; set; }
    public string Name { get; set; } = "";
    public string Email { get; set; } = "";
    public decimal Salary { get; set; }
    public int DepartmentId { get; set; }
    public Department Department { get; set; } = null!;
}
```

Say: *"In production I'd add a BaseEntity with audit fields, IsDeleted and RowVersion."*

**Step 2 – DbContext (3 min)**

```csharp
public class AppDbContext(DbContextOptions<AppDbContext> o) : DbContext(o)
{
    public DbSet<Employee> Employees => Set<Employee>();
    public DbSet<Department> Departments => Set<Department>();

    protected override void OnModelCreating(ModelBuilder b)
    {
        b.Entity<Employee>().HasIndex(e => e.Email).IsUnique();
        b.Entity<Employee>().Property(e => e.Salary).HasPrecision(18, 2);
    }
}
```

Say: *"The unique index protects data integrity at the database level, not just in code."*

**Step 3 – DTOs (3 min)**

```csharp
public record EmployeeDto(int Id, string Name, string Email, decimal Salary, string Department);

public record CreateEmployeeRequest(
    [Required, StringLength(100)] string Name,
    [Required, EmailAddress] string Email,
    [Range(1, 10_000_000)] decimal Salary,
    int DepartmentId);
```

Say: *"I never expose entities. DTOs keep the contract stable and avoid over-posting and circular JSON."*

**Step 4 – Custom exceptions (1 min)**

```csharp
public class NotFoundException(string msg) : Exception(msg);
public class ConflictException(string msg) : Exception(msg);
```

**Step 5 – Service: interface, then implementation (10 min, the core)**

```csharp
public interface IEmployeeService {
    Task<List<EmployeeDto>> GetAllAsync(int page, int size, CancellationToken ct);
    Task<EmployeeDto> GetByIdAsync(int id, CancellationToken ct);
    Task<EmployeeDto> CreateAsync(CreateEmployeeRequest r, CancellationToken ct);
    Task DeleteAsync(int id, CancellationToken ct);
}

public class EmployeeService(AppDbContext db) : IEmployeeService
{
    public Task<List<EmployeeDto>> GetAllAsync(int page, int size, CancellationToken ct) =>
        db.Employees.AsNoTracking()
          .OrderBy(e => e.Id)
          .Skip((page - 1) * size).Take(size)
          .Select(e => new EmployeeDto(e.Id, e.Name, e.Email, e.Salary, e.Department.Name))
          .ToListAsync(ct);

    public async Task<EmployeeDto> GetByIdAsync(int id, CancellationToken ct) =>
        await db.Employees.AsNoTracking()
          .Where(e => e.Id == id)
          .Select(e => new EmployeeDto(e.Id, e.Name, e.Email, e.Salary, e.Department.Name))
          .FirstOrDefaultAsync(ct)
        ?? throw new NotFoundException($"Employee {id} not found");

    public async Task<EmployeeDto> CreateAsync(CreateEmployeeRequest r, CancellationToken ct)
    {
        if (!await db.Departments.AnyAsync(d => d.Id == r.DepartmentId, ct))
            throw new NotFoundException($"Department {r.DepartmentId} not found");
        if (await db.Employees.AnyAsync(e => e.Email == r.Email, ct))
            throw new ConflictException($"Email {r.Email} already exists");

        var emp = new Employee { Name = r.Name, Email = r.Email,
                                 Salary = r.Salary, DepartmentId = r.DepartmentId };
        db.Employees.Add(emp);
        await db.SaveChangesAsync(ct);
        return await GetByIdAsync(emp.Id, ct);
    }

    public async Task DeleteAsync(int id, CancellationToken ct)
    {
        var emp = await db.Employees.FindAsync([id], ct)
                  ?? throw new NotFoundException($"Employee {id} not found");
        db.Employees.Remove(emp);
        await db.SaveChangesAsync(ct);
    }
}
```

Say while writing:
- *"AsNoTracking for reads, because it's faster with no change tracking."*
- *"Projection with Select fetches only the columns I need."*
- *"Pagination, since you never return an unbounded list."*
- *"CancellationToken, so if the client disconnects the query stops."*
- *"I'm not adding a generic repository, because DbContext already is a unit of work."*

**Step 6 – Controller (5 min)**

```csharp
[ApiController]
[Route("api/v1/employees")]
public class EmployeesController(IEmployeeService svc) : ControllerBase
{
    [HttpGet]
    public async Task<IActionResult> Get(int page = 1, int size = 10, CancellationToken ct = default)
        => Ok(await svc.GetAllAsync(page, Math.Min(size, 100), ct));

    [HttpGet("{id:int}")]
    public async Task<IActionResult> GetById(int id, CancellationToken ct)
        => Ok(await svc.GetByIdAsync(id, ct));

    [HttpPost]
    public async Task<IActionResult> Create(CreateEmployeeRequest r, CancellationToken ct)
    {
        var dto = await svc.CreateAsync(r, ct);
        return CreatedAtAction(nameof(GetById), new { id = dto.Id }, dto);
    }

    [HttpDelete("{id:int}")]
    public async Task<IActionResult> Delete(int id, CancellationToken ct)
    {
        await svc.DeleteAsync(id, ct);
        return NoContent();
    }
}
```

Say: *"The controller is thin and only handles HTTP. [ApiController] gives automatic 400s for invalid models. POST returns 201 with a Location header."*

**Step 7 – Global exception handler (4 min)**

```csharp
public class GlobalExceptionHandler(ILogger<GlobalExceptionHandler> log) : IExceptionHandler
{
    public async ValueTask<bool> TryHandleAsync(HttpContext ctx, Exception ex, CancellationToken ct)
    {
        var status = ex switch {
            NotFoundException => 404,
            ConflictException => 409,
            _ => 500
        };
        if (status == 500) log.LogError(ex, "Unhandled error");

        ctx.Response.StatusCode = status;
        await ctx.Response.WriteAsJsonAsync(new ProblemDetails {
            Status = status,
            Title = status == 500 ? "Server error" : ex.Message,
            Instance = ctx.Request.Path
        }, ct);
        return true;
    }
}
```

Say: *"One place for errors, a consistent ProblemDetails format, and no try/catch in every controller. I hide internal details on 500s."*

**Step 8 – Program.cs (3 min)**

```csharp
var builder = WebApplication.CreateBuilder(args);

builder.Services.AddDbContext<AppDbContext>(o =>
    o.UseSqlServer(builder.Configuration.GetConnectionString("Default")));
builder.Services.AddScoped<IEmployeeService, EmployeeService>();
builder.Services.AddExceptionHandler<GlobalExceptionHandler>();
builder.Services.AddProblemDetails();
builder.Services.AddControllers();
builder.Services.AddSwaggerGen();

var app = builder.Build();
app.UseExceptionHandler();
app.UseSwagger(); app.UseSwaggerUI();
app.MapControllers();
app.Run();
```

Say: *"The service is Scoped because DbContext is scoped, one per request. A Singleton holding DbContext would be a captive dependency bug."*

**Step 9 – Bonus if time allows (5 min): reports endpoint**

```csharp
[HttpGet("reports/top/{n:int}")]
public async Task<IActionResult> TopN(int n, CancellationToken ct) =>
    Ok(await db.Departments.Select(d => new {
        d.Name,
        Top = d.Employees.OrderByDescending(e => e.Salary).Take(n)
                         .Select(e => new { e.Name, e.Salary })
    }).ToListAsync(ct));
```

**Step 10 – Wrap up verbally (2 min)**

*"With more time I'd add":*
1. JWT auth with role-based `[Authorize(Roles="Admin")]`
2. RowVersion plus ETag/If-Match for concurrency
3. Soft delete with a global query filter, and audit fields in SaveChanges
4. FluentValidation for complex rules
5. Unit tests on the service (xUnit plus InMemory/SQLite)
6. Health checks, rate limiting, Serilog plus App Insights
7. Redis caching for department lookups
8. At scale: split into microservices, use Service Bus events with the outbox pattern, put APIM in front, and use Entra ID

**Tips:**
- **Talk while you type.** Interviewers grade your reasoning more than your syntax.
- **Write the interface before the implementation.** It shows you design first.
- **Don't get stuck on syntax.** Say *"I'd check the exact overload"* and move on.
- **If time is short,** skip Step 9 but always do Step 7. The exception handler is what separates Lead answers from mid-level ones.
