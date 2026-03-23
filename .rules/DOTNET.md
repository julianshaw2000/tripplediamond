# C#/.NET – Coding Rules (ASP.NET Core, .NET 9+)

These rules are mandatory for all C# code in this repository.

---

## 1. Project & Language Settings

- Target `.NET 9` (or latest stable). Use `<Nullable>enable</Nullable>` and treat warnings as errors.
- Enable `<ImplicitUsings>enable</ImplicitUsings>` and `<LangVersion>latest</LangVersion>`.
- File-scoped namespaces everywhere: `namespace MyApp.Features.Orders;`
- One public type per file. Filename matches type name.
- Enforce formatting via `.editorconfig` and `dotnet format` in CI.
- Use `dotnet build`, `dotnet test`, `dotnet format --verify-no-changes` as the CI gate.

---

## 2. Naming & Style

- `PascalCase`: types, public members, methods, properties, constants.
- `camelCase`: parameters, local variables, private fields (no `_` prefix unless disambiguation is needed).
- Prefer `record` for DTOs, commands, queries, and value objects.
- Use primary constructors (C# 12+) for services with simple injection:
  ```csharp
  public class OrderService(IOrderRepository repo, ILogger<OrderService> logger) { ... }
  ```
- Use `required` on DTO/record properties instead of constructor validation where appropriate.
- No vague names: `Helper`, `Manager`, `Utils`, `Data`, `Stuff`.

---

## 3. Architecture — Vertical Slice (default)

Organise code by **feature**, not by layer. Each slice owns everything it needs.

```
src/
  Features/
    Orders/
      CreateOrder.cs       ← command + handler + validator + endpoint in one file (or split if large)
      GetOrder.cs          ← query + handler + response + endpoint
      OrderRepository.cs   ← if the slice needs a custom query
  Common/
    Behaviours/            ← MediatR pipeline behaviours (validation, logging, perf)
    Exceptions/
    Extensions/
  Infrastructure/
    Persistence/           ← DbContext, migrations, global configs
    ExternalServices/
  Program.cs
```

- A slice may have its own sub-folder when it grows (`Features/Orders/Commands/`, `Features/Orders/Queries/`).
- **Cross-slice sharing** goes into `Common/` or `Shared/` — never import one feature slice into another directly.
- Use **Clean Architecture** (Domain / Application / Infrastructure / Web layers) only when domain complexity warrants it (complex invariants, rich domain model). Default to Vertical Slice.

---

## 4. CQRS with MediatR

Every use case is a `IRequest<TResponse>` handled by a single `IRequestHandler<TRequest, TResponse>`.

```csharp
// Command
public record CreateOrderCommand(Guid CustomerId, List<OrderLineDto> Lines) : IRequest<Result<Guid>>;

// Handler
public class CreateOrderHandler(AppDbContext db, ILogger<CreateOrderHandler> logger)
    : IRequestHandler<CreateOrderCommand, Result<Guid>>
{
    public async Task<Result<Guid>> Handle(CreateOrderCommand cmd, CancellationToken ct)
    {
        // business logic here
    }
}
```

- **Commands** mutate state and return `Result<T>` or `Result`.
- **Queries** are read-only and return `Result<TResponse>`. Project directly to the response DTO — do not return domain entities.
- Register MediatR pipeline behaviours for: validation, logging, performance tracking.
- Do not put business logic in endpoints or controllers — delegate to MediatR.

---

## 5. Result Pattern (no exception-driven flow)

Use a `Result<T>` / `Result` type (e.g., `ErrorOr`, `FluentResults`, or a custom `Result`) for all expected failure paths. Reserve exceptions for truly exceptional conditions.

```csharp
// Returning errors
if (order is null)
    return Result.Failure<Order>(Error.NotFound("Order.NotFound", $"Order {id} not found"));

// Caller maps errors to HTTP
return result.Match(
    onSuccess: order => TypedResults.Ok(order),
    onFailure: errors => errors.ToProblemResult());
```

- **Never throw** for domain validation, not-found, or business rule violations.
- Map `Result` → HTTP in the endpoint/controller layer only. The application layer never knows about HTTP.

---

## 6. Minimal APIs (preferred over Controllers)

- Default to Minimal APIs. Use `IEndpointRouteBuilder` extension methods to group endpoints by feature:
  ```csharp
  public static class OrderEndpoints
  {
      public static IEndpointRouteBuilder MapOrderEndpoints(this IEndpointRouteBuilder app)
      {
          var group = app.MapGroup("/api/orders").RequireAuthorization();
          group.MapPost("/", CreateOrder);
          group.MapGet("/{id:guid}", GetOrder);
          return app;
      }
  }
  ```
- Use `TypedResults` (not `Results<T>`) for compile-time checked responses and accurate OpenAPI.
- Annotate endpoints with `.WithName()`, `.WithSummary()`, `.WithTags()`, `.Produces<T>()` for OpenAPI.
- Use `[AsParameters]` to bind complex query parameter objects.
- Use Controllers only if the team is more comfortable with them or if the project already uses them — do not mix.

---

## 7. Validation

- Use **FluentValidation** with a MediatR `ValidationBehaviour<TRequest, TResponse>` pipeline behaviour.
- Validators live next to their request type (same file or `CreateOrder.Validator.cs`).
- The pipeline behaviour collects all failures and returns a `Result` containing a `ValidationError` — it does not throw.
- For simple input binding validation (required fields, length), also use DataAnnotation attributes on request records.

```csharp
public class CreateOrderValidator : AbstractValidator<CreateOrderCommand>
{
    public CreateOrderValidator()
    {
        RuleFor(x => x.CustomerId).NotEmpty();
        RuleFor(x => x.Lines).NotEmpty().ForEach(line => line.SetValidator(new OrderLineValidator()));
    }
}
```

---

## 8. Dependency Injection

- Register services by interface. Group registrations in `IServiceCollection` extension methods per feature/layer.
- Use **keyed services** (`AddKeyedScoped`, `[FromKeyedServices]`) for multiple implementations of the same interface.
- Lifetimes:
  - `Scoped`: DbContext, per-request services, repositories.
  - `Singleton`: stateless services, caches, HTTP clients via `IHttpClientFactory`.
  - `Transient`: lightweight, stateless, short-lived only.
- No service locator (`IServiceProvider` injected into business logic).
- Use `Scrutor` for assembly scanning / decoration if registrations become repetitive.

---

## 9. Async & Cancellation

- All I/O paths async end-to-end. No `.Result`, `.Wait()`, `GetAwaiter().GetResult()`.
- Pass and forward `CancellationToken ct` through every async call chain: endpoints → handlers → repos → EF Core / HttpClient.
- Use `Task.WhenAll` for independent parallel I/O; avoid over-parallelisation in request handlers.
- `ValueTask<T>` only where benchmarks prove it reduces allocations on a hot path — default to `Task<T>`.

---

## 10. Data Access (EF Core)

- `DbContext` is `Scoped`. Never use it as `Singleton`.
- `AsNoTracking()` on all read-only queries. Only track entities you intend to update/delete.
- Project to response DTOs directly in queries — never return `IQueryable<Entity>` from a repository or handler.
- Use **compiled queries** (`EF.CompileAsyncQuery`) for hot read paths.
- Use **split queries** (`AsSplitQuery()`) to avoid cartesian-explosion on collection includes.
- Keep migrations in `Infrastructure/Persistence/Migrations/`. Apply via `dotnet ef database update` or a startup migration runner.
- Use **interceptors** (`ISaveChangesInterceptor`) for cross-cutting concerns (soft-delete, audit fields, domain events dispatch).
- Do not expose `DbContext` outside the Infrastructure layer. Wrap complex queries in repository methods or query objects.

---

## 11. API Design & Problem Details

- Return **RFC 7807 Problem Details** for all error responses. Register `builder.Services.AddProblemDetails()` and `app.UseExceptionHandler()`.
- Use a global `IExceptionHandler` implementation to map unhandled exceptions to Problem Details — do not catch-and-rethrow everywhere.
- HTTP status codes:
  - `200 OK` / `201 Created` / `204 No Content` for success.
  - `400 Bad Request` for validation failures.
  - `401 Unauthorized` / `403 Forbidden` for auth.
  - `404 Not Found` for missing resources.
  - `409 Conflict` for domain rule violations (duplicate, wrong state).
  - `422 Unprocessable Entity` for semantic validation failures.
- Keep response models (`*Response`, `*Dto`) stable and explicit. Never return EF entities directly.
- Version APIs via URL segment (`/api/v1/...`) or header — decide once and be consistent.

---

## 12. Design Patterns

### Repository
- Use repositories to abstract data access **when** the slice has complex query logic or you need testability without the DB.
- Keep repositories focused: one aggregate root per repository.
- Do not wrap simple CRUD — inject `DbContext` directly in handlers for trivial cases.

### Specification
- Encapsulate reusable query predicates in `Specification<T>` objects to avoid duplicating `.Where(...)` expressions.

### Decorator
- Use the Decorator pattern (via Scrutor or manual registration) for cross-cutting concerns on services: caching, logging, retry.

### Outbox Pattern
- For reliable event publishing (e.g., via MassTransit or a custom outbox), write domain events to an `OutboxMessages` table inside the same transaction, then process asynchronously via a background service.

### Domain Events
- Raise domain events as value objects inside the aggregate; collect them on `DbContext.SaveChanges` via an interceptor and dispatch via MediatR `INotification`.

### Options / Strategy
- Use the Options pattern for all configuration. Use the Strategy pattern (registered via keyed DI) to swap algorithms or integrations at runtime without conditionals.

---

## 13. Logging & Observability

- `ILogger<T>` only. Structured logging via Serilog (or built-in) with property enrichment.
- Use **log scopes** or enrichers to include `CorrelationId`, `UserId`, `TenantId` on every log entry in the request pipeline.
- Use **`LoggerMessage.Define`** (or `[LoggerMessage]` source generator) for high-frequency log paths — avoid `LogInformation("Processing {Id}", id)` in hot loops.
- Add **OpenTelemetry** traces and metrics for production observability (`AddOpenTelemetry()`, `ActivitySource`).
- Never log secrets, PII, or full request bodies by default.

---

## 14. Security

- HTTPS only; redirect HTTP → HTTPS.
- Use standard auth: OIDC / JWT bearer. No hand-rolled auth.
- Enforce authorization via **policies** (`RequireAuthenticatedUser`, `RequireRole`, `RequireClaim`); no inline `User.IsInRole()` checks in handlers.
- Apply built-in **rate limiting** (`AddRateLimiter`) on public and auth endpoints.
- Use **CORS** explicitly; no wildcard origins in production.
- Validate file uploads (type, size, virus scan hook). Never trust `Content-Type` header alone.
- Bind config secrets from environment variables or a secrets manager (Azure Key Vault, AWS Secrets Manager) — never from `appsettings.json` checked into source control.

---

## 15. Testing

- **Unit tests**: xUnit + NSubstitute (or Moq). Test handlers, validators, domain logic in isolation.
- **Integration tests**: `WebApplicationFactory<Program>` + **Testcontainers** for a real database. Test full request → DB round-trips.
- **Naming**: `MethodName_StateUnderTest_ExpectedBehaviour`.
- Test auth failures, validation failures, and not-found paths — not just happy paths.
- No test should share mutable state with another test. Each test owns its data.
- CI gate: `dotnet build && dotnet test && dotnet format --verify-no-changes`.

---

## 16. Code Review Enforcement

Flag and refuse to merge code that:
- Puts business logic in endpoints, controllers, or `Program.cs`.
- Returns domain entities directly from an API.
- Uses `.Result`, `.Wait()`, or any blocking async call.
- Throws exceptions for expected domain failures (use `Result`).
- Reads raw config strings outside of typed options classes.
- Mixes feature slices (one feature importing another's internals).
- Uses `IServiceProvider` as a service locator in application or domain code.

Prefer **simple, explicit, boring solutions** over clever abstractions.
