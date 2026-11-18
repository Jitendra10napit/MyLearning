In .NET, Clean Architecture is usually set up as **four projects** with one rule: **dependencies point inward only**. Domain is at the centre and references nothing. Each outer layer may depend on the layers inside it, never the other way around.

```
        API (Presentation)  ──►  Application  ──►  Domain
        Infrastructure      ──►  Application  ──►  Domain
        API also references Infrastructure, but only to register DI
```

## Solution structure

```
src/
├── MyApp.Domain/                 ← no project references, no NuGet packages (ideally)
├── MyApp.Application/            ← references Domain
├── MyApp.Infrastructure/         ← references Application (and Domain through it)
└── MyApp.Api/                    ← references Application + Infrastructure
tests/
├── MyApp.Domain.UnitTests/
├── MyApp.Application.UnitTests/
├── MyApp.Infrastructure.IntegrationTests/
└── MyApp.Api.FunctionalTests/
```

---

## 1. Domain layer: business rules only

```
MyApp.Domain/
├── Common/
│   ├── BaseEntity.cs            (Id, domain events list)
│   ├── BaseAuditableEntity.cs   (CreatedAt, CreatedBy…)
│   ├── ValueObject.cs
│   └── IDomainEvent.cs
├── Entities/
│   ├── Order.cs
│   └── OrderItem.cs
├── ValueObjects/
│   ├── Money.cs
│   └── Address.cs
├── Enums/
│   └── OrderStatus.cs
├── Events/
│   └── OrderPlacedEvent.cs
├── Exceptions/
│   └── OrderAlreadyShippedException.cs
└── Interfaces/                  (optional: domain service contracts, e.g. IPricingPolicy)
```

**Goes here:** entities with behaviour (`order.AddItem()`, `order.Cancel()`), value objects, enums, domain events, domain exceptions, and business invariants.
**Never here:** EF Core attributes, DbContext, DTOs, HTTP, logging, MediatR, or anything with I/O.

---

## 2. Application layer: use cases and orchestration

Feature folders are the standard layout here, with one folder per use case (CQRS style):

```
MyApp.Application/
├── Common/
│   ├── Interfaces/
│   │   ├── IApplicationDbContext.cs   (or IOrderRepository.cs)
│   │   ├── IEmailService.cs
│   │   ├── IDateTimeProvider.cs
│   │   └── ICurrentUserService.cs
│   ├── Behaviours/                    (MediatR pipeline)
│   │   ├── ValidationBehaviour.cs
│   │   ├── LoggingBehaviour.cs
│   │   └── PerformanceBehaviour.cs
│   ├── Exceptions/
│   │   ├── ValidationException.cs
│   │   └── NotFoundException.cs
│   ├── Mappings/                      (AutoMapper/Mapster profiles)
│   └── Models/
│       ├── Result.cs
│       └── PaginatedList.cs
├── Orders/
│   ├── Commands/
│   │   └── CreateOrder/
│   │       ├── CreateOrderCommand.cs
│   │       ├── CreateOrderCommandHandler.cs
│   │       └── CreateOrderCommandValidator.cs
│   ├── Queries/
│   │   └── GetOrderById/
│   │       ├── GetOrderByIdQuery.cs
│   │       ├── GetOrderByIdQueryHandler.cs
│   │       └── OrderDto.cs
│   └── EventHandlers/
│       └── OrderPlacedEventHandler.cs
└── DependencyInjection.cs             (AddApplication(): MediatR, validators, mappers)
```

**Goes here:** commands, queries and handlers; DTOs; FluentValidation validators; **interfaces** for everything external (database, email, blob storage, queues, clock); and pipeline behaviours.
**Never here:** the implementations of those interfaces, EF Core-specific code, or `HttpContext`.

The key idea is that Application **defines** `IEmailService` and Infrastructure **implements** it. That's dependency inversion, and it's what keeps the core testable.

---

## 3. Infrastructure layer: everything that talks to the outside world

```
MyApp.Infrastructure/
├── Persistence/
│   ├── ApplicationDbContext.cs
│   ├── Configurations/                (IEntityTypeConfiguration<T>, Fluent API)
│   │   └── OrderConfiguration.cs
│   ├── Migrations/
│   ├── Interceptors/
│   │   └── AuditableEntityInterceptor.cs
│   ├── Repositories/                  (only if you use the repository pattern)
│   │   └── OrderRepository.cs
│   └── Seed/
│       └── ApplicationDbContextSeed.cs
├── Identity/
│   └── IdentityService.cs
├── Services/
│   ├── EmailService.cs                (SendGrid/SMTP)
│   ├── BlobStorageService.cs          (Azure Blob)
│   └── DateTimeProvider.cs
├── Messaging/
│   └── ServiceBusPublisher.cs
├── Caching/
│   └── RedisCacheService.cs
└── DependencyInjection.cs             (AddInfrastructure(config): DbContext, services, options)
```

**Goes here:** EF Core, Dapper, external API clients, Azure SDKs, file system access, caching, messaging, and Identity.
Use the Fluent API in `Configurations/` rather than data annotations on entities, so Domain stays free of persistence code.

---

## 4. Presentation layer (API): thin entry point

```
MyApp.Api/
├── Controllers/                       (or Endpoints/ for Minimal APIs)
│   └── OrdersController.cs
├── Middleware/
│   └── ExceptionHandlingMiddleware.cs (maps exceptions → ProblemDetails)
├── Filters/
├── Services/
│   └── CurrentUserService.cs          (implements ICurrentUserService using HttpContext)
├── Extensions/
│   └── ServiceCollectionExtensions.cs (Swagger, auth, CORS, versioning)
├── appsettings.json
└── Program.cs
```

Controllers should only send requests to MediatR and return the result:

```csharp
[HttpPost]
public async Task<IActionResult> Create(CreateOrderCommand cmd)
    => Ok(await _mediator.Send(cmd));
```

`Program.cs` is the composition root, where all the layers get wired together:

```csharp
builder.Services
    .AddApplication()
    .AddInfrastructure(builder.Configuration)
    .AddPresentation();
```

---

## Quick "where does this file go?" reference

| File | Layer |
|---|---|
| Entity with business methods | Domain |
| Value object, enum, domain event | Domain |
| DTO / request / response model | Application |
| Command/query + handler + validator | Application |
| `IRepository`, `IEmailService` interface | Application |
| Repository / DbContext implementation | Infrastructure |
| EF entity configuration, migrations | Infrastructure |
| Azure Blob / Service Bus / Redis clients | Infrastructure |
| Controller / Minimal API endpoint | API |
| Exception middleware, Swagger, auth setup | API |
| `CurrentUserService` (needs HttpContext) | API |
| Options classes bound from config | The layer that consumes them |

## Common mistakes interviewers look for

1. **DTOs leaking into Domain**, or **entities returned straight from controllers**. Always map to DTOs.
2. **Domain referencing EF Core** because of `[Key]` or `[Required]` attributes. Use Fluent API configuration instead.
3. **Anaemic entities**, where all the logic sits in handlers. Business rules belong inside the entity.
4. **A generic repository on top of EF Core** with no clear reason. Many teams expose `IApplicationDbContext` instead, since DbContext is already a unit of work. Either approach is acceptable, but be ready to defend your choice.
5. **Business logic in controllers.**
6. **Not enforcing the boundaries.** Add architecture tests with `NetArchTest.Rules` so a wrong reference fails the build:

```csharp
Types.InAssembly(typeof(Order).Assembly)
     .ShouldNot().HaveDependencyOn("MyApp.Infrastructure")
     .GetResult().IsSuccessful.Should().BeTrue();
```

## Reference templates to study

- **Jason Taylor's Clean Architecture template**: `dotnet new install Clean.Architecture.Solution.Template`, then `dotnet new ca-sln`
- **Ardalis (Steve Smith) Clean Architecture template**: `dotnet new install Ardalis.CleanArchitecture.Template`

Generate one of these and read through the folders; it's the quickest way to see the layout working end to end.

If you're asked about variations in an interview, the main ones are: **Vertical Slice** (feature folders across all layers, often in a single project), and splitting **Persistence** out of Infrastructure into its own project in larger systems. Both keep the same rule that dependencies point inward.
