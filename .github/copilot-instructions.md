# Copilot Instructions — Building with Trellis

This project uses the **Trellis** framework (.NET 10). Trellis combines Railway-Oriented Programming (ROP) with Domain-Driven Design (DDD). Follow these patterns exactly.

## Core Principles

1. **Errors are values, not exceptions.** Use `Result<T>` for expected failures. Never throw for business logic. Never use try/catch in Domain or Application layers.
2. **Make illegal states unrepresentable.** Every domain concept is a value object with `TryCreate`. If it exists, it's valid.
3. **No primitive obsession.** No raw `Guid`, `string`, or `int` in domain method signatures. Use typed value objects everywhere.
4. **Optional values use `Maybe<T>`, never null.** `Maybe<PhoneNumber>`, not `PhoneNumber?`.

## Architecture

```
Api → Application → Domain
Api → Infrastructure → Application → Domain
```

| Layer | Depends On | Contains |
|-------|-----------|----------|
| **Domain** | Trellis packages only (Results, Primitives, DDD, Stateless, Authorization) | Aggregates, entities, value objects, domain events, specifications, permission constants |
| **Application** | Domain, Trellis.Mediator | Commands, queries, handlers, repository interfaces |
| **Infrastructure** | Application, Trellis.EntityFrameworkCore, EF Core provider | DbContext, entity configurations, repository implementations, migrations |
| **Api** | Application, Infrastructure, Trellis.Asp, Mediator.SourceGenerator | Endpoints, DTOs, Program.cs (composition root), IActorProvider implementation |

**Rules:**
- Domain has ZERO infrastructure dependencies (no EF Core, no ASP.NET, no Mediator).
- Repository interfaces live in Application, implementations in Infrastructure.
- `Mediator.SourceGenerator` is installed ONLY in the Api project.
- Each layer has one `DependencyInjection.cs` with `Add{Layer}()` extension method.

## Value Objects

Use `Trellis.Primitives` base classes with the source generator (`Trellis.Primitives.Generator`):

| Base Class | Backing Type | Example |
|-----------|-------------|---------|
| `RequiredGuid<T>` | `Guid` | `CustomerId`, `OrderId`, `ProductId`, `LineItemId` |
| `RequiredString<T>` | `string` | `FirstName`, `LastName`, `ProductName`, `Sku` |
| `RequiredInt<T>` | `int` | `Quantity`, `StockQuantity` |
| `RequiredDecimal<T>` | `decimal` | `UnitPrice` |
| `RequiredEnum<T>` | `string` (smart enum) | `OrderStatus` |

**Identity types** always use `RequiredGuid` with `Guid.CreateVersion7()` for time-sortable IDs:
```csharp
public sealed class OrderId : RequiredGuid<OrderId>
{
    public static OrderId NewUnique() => Create(Guid.CreateVersion7());
}
```

**Built-in value objects** from `Trellis.Primitives` namespace: `EmailAddress`, `PhoneNumber`, `Url`, `Money`, `Currency`, `NonEmptyString`, `FirstName`, `LastName`, `Country`.

**Two factory methods** on every value object:
- `TryCreate(value)` → `Result<T>` — use for untrusted input
- `Create(value)` → `T` (throws) — use in tests and known-valid contexts

**Smart enums** via `RequiredEnum<T>`:
```csharp
public sealed class OrderStatus : RequiredEnum<OrderStatus>
{
    public static readonly OrderStatus Draft = new("Draft");
    public static readonly OrderStatus Submitted = new("Submitted");
    // ... etc
}
```

## Result\<T\> and ROP Chains

Every operation returns `Result<T>`. Chain with Bind/Map/Tap/Ensure:

```csharp
public async Task<Result<Order>> CreateDraftOrderAsync(...)
    => await CustomerId.TryCreate(customerId)
        .BindAsync(id => _customerRepo.GetByIdAsync(id))
        .BindAsync(customer => CreateOrderWithLineItems(customer, lineItems))
        .TapAsync(order => _orderRepo.SaveAsync(order));
```

**Key methods:**

| Method | Purpose | When to Use |
|--------|---------|-------------|
| `Bind` / `BindAsync` | Transform value, may fail (returns Result) | Calling another operation that can fail |
| `Map` / `MapAsync` | Transform value, cannot fail (returns raw value) | Simple transformations |
| `Tap` / `TapAsync` | Side effect, doesn't change value | Saving, logging, sending events |
| `Ensure` / `EnsureAsync` | Validate, fail if predicate is false | Adding guard conditions |
| `Combine` | Combine multiple Results into a tuple | Validating multiple inputs together |
| `Match` / `MatchError` | Pattern match on success/failure | Converting to HTTP responses |
| `ParallelAsync` | Run independent async Results in parallel | Fetching multiple entities concurrently |

**Combine pattern** for validating multiple inputs:
```csharp
FirstName.TryCreate(firstName)
    .Combine(LastName.TryCreate(lastName))
    .Combine(EmailAddress.TryCreate(email))
    .Bind((first, last, email) => Customer.TryCreate(first, last, email));
```

**ParallelAsync** for concurrent fetches:
```csharp
var result = await _customerRepo.GetByIdAsync(customerId)
    .ParallelAsync(_productRepo.GetByIdsAsync(productIds))
    .WhenAllAsync()
    .MapAsync((customer, products) => CreateOrder(customer, products));
```

## Maybe\<T\>

Use for optional values. Never null in the domain model.

```csharp
// Domain property
public Maybe<PhoneNumber> PhoneNumber { get; }

// Repository returns Maybe for lookups
Maybe<Customer> customer = await _context.Customers
    .FirstOrDefaultMaybeAsync(c => c.Id == customerId, ct);

// Convert to Result when entity must exist
Result<Customer> result = customer.ToResult(Error.NotFound("Customer", customerId));
```

## Error Types

| Error Type | HTTP Status | When Used |
|-----------|-------------|-----------|
| `ValidationError` | 400 | Invalid input, invalid state transition, insufficient stock |
| `NotFoundError` | 404 | Entity not found |
| `ConflictError` | 409 | Duplicate email/SKU, concurrency conflict |
| `ForbiddenError` | 403 | Missing permission, resource authorization failure |
| `DomainError` | 422 | Business rule violation |

Create errors via factory methods: `Error.Validation(message, field)`, `Error.NotFound(type, id)`, `Error.Conflict(message)`, `Error.Forbidden(message)`.

## DDD Building Blocks

```csharp
// Aggregate
public sealed class Order : Aggregate<OrderId> { ... }

// Entity (owned by an aggregate)
public sealed class LineItem : Entity<LineItemId> { ... }

// Value Object (multi-field, no identity)
public sealed class ShippingAddress : ValueObject
{
    protected override IEnumerable<IComparable> GetEqualityComponents() { ... }
}

// Specification (composable query filter)
public sealed class OverdueOrderSpec : Specification<Order>
{
    public override Expression<Func<Order, bool>> ToExpression() =>
        order => order.Status == OrderStatus.Submitted
              && order.SubmittedAt < DateTime.UtcNow.AddDays(-7);
}
```

## State Machine (Trellis.Stateless)

Use Stateless library with Trellis integration. Transitions return `Result<T>`:

```csharp
_machine = new StateMachine<OrderStatus, Trigger>(() => Status, s => Status = s);
_machine.Configure(OrderStatus.Draft)
    .Permit(Trigger.Submit, OrderStatus.Submitted)
    .Permit(Trigger.Cancel, OrderStatus.Cancelled);

// Fire returns Result<T>, not void/throw
public Result<Order> Submit() => _machine.FireResult(Trigger.Submit)
    .Map(() => { SubmittedAt = DateTime.UtcNow; return this; });
```

## CQRS with Mediator (Trellis.Mediator)

All operations are Commands or Queries using martinothamar/Mediator:

```csharp
// Command with validation and authorization
public sealed record CreateCustomerCommand(string FirstName, string LastName, string Email)
    : ICommand<Result<Customer>>, IValidate, IAuthorize
{
    public IReadOnlyList<string> RequiredPermissions => [Permissions.CustomersCreate];

    public IResult Validate() =>
        string.IsNullOrWhiteSpace(FirstName)
            ? Result.Failure(Error.Validation("Required", "firstName"))
            : Result.Success();
}

// Resource-based authorization
public sealed record CancelOrderCommand(OrderId OrderId, string CreatedByActorId)
    : ICommand<Result<Order>>, IAuthorize, IAuthorizeResource
{
    public IReadOnlyList<string> RequiredPermissions => [Permissions.OrdersCancel];

    public IResult Authorize(Actor actor) =>
        actor.Id == CreatedByActorId || actor.HasPermission(Permissions.OrdersReadAll)
            ? Result.Success()
            : Result.Failure(Error.Forbidden("Only the order creator or admin can cancel"));
}

// Query
public sealed record GetOrderQuery(OrderId OrderId)
    : IQuery<Result<Order>>, IAuthorize
{
    public IReadOnlyList<string> RequiredPermissions => [Permissions.OrdersRead];
}
```

**Pipeline behavior order** (registered via `ServiceCollectionExtensions.PipelineBehaviors`):
```
Request → ExceptionBehavior → TracingBehavior → LoggingBehavior
  → AuthorizationBehavior → ResourceAuthorizationBehavior
    → ValidationBehavior → Handler → Result<T>
```

**Registration:**
```csharp
services.AddMediator(options =>
{
    options.Assemblies = [typeof(CreateCustomerCommand).Assembly];
    options.PipelineBehaviors = ServiceCollectionExtensions.PipelineBehaviors;
});
```

## Authorization (Trellis.Authorization)

**Permissions** are string constants in the Domain layer:
```csharp
public static class Permissions
{
    public const string CustomersCreate = "customers:create";
    public const string OrdersCreate = "orders:create";
    public const string OrdersApprove = "orders:approve";
    // etc.
}
```

**Actor** represents the authenticated user:
```csharp
var actor = Actor.Create("user-42", new HashSet<string> { "orders:create", "orders:read" });
actor.HasPermission("orders:create");  // true
```

**IActorProvider** resolves the current actor. Implement in the Api layer:
```csharp
public class TestActorProvider(IHttpContextAccessor accessor) : IActorProvider
{
    public Actor GetCurrentActor()
    {
        var header = accessor.HttpContext?.Request.Headers["X-Test-Actor"].FirstOrDefault();
        if (header is null) return Actor.Create("admin", allPermissions); // default admin
        var payload = JsonSerializer.Deserialize<ActorPayload>(header);
        return Actor.Create(payload.Id, payload.Permissions.ToHashSet());
    }
}
```

## EF Core (Trellis.EntityFrameworkCore)

**Convention-based value converters** — one line replaces all `HasConversion()` boilerplate:
```csharp
protected override void ConfigureConventions(ModelConfigurationBuilder configurationBuilder)
{
    configurationBuilder.ApplyTrellisConventions(typeof(CustomerId).Assembly);
    // Scans for IScalarValue<TSelf, TPrimitive> and RequiredEnum<TSelf>
    // Trellis.Primitives assembly is always included automatically
}
```

**NEVER write `HasConversion()`.** ApplyTrellisConventions handles all Trellis value objects.

**OnModelCreating** — configure keys, indexes, constraints only:
```csharp
modelBuilder.Entity<Customer>(b =>
{
    b.HasKey(c => c.Id);
    b.Property(c => c.Name).HasMaxLength(100).IsRequired();
    b.HasIndex(c => c.Email).IsUnique();
    // NO HasConversion() — already handled by conventions
});
```

**Result-returning SaveChanges:**
```csharp
var result = await context.SaveChangesResultAsync(ct);
// DbUpdateConcurrencyException → ConflictError
// Duplicate key → ConflictError
// FK violation → DomainError
```

**Maybe-returning queries:**
```csharp
Maybe<Customer> customer = await context.Customers
    .FirstOrDefaultMaybeAsync(c => c.Id == customerId, ct);

Result<Customer> customer = await context.Customers
    .FirstOrDefaultResultAsync(c => c.Id == customerId, Error.NotFound("Customer", id), ct);
```

**Specification queries:**
```csharp
var overdueOrders = await context.Orders
    .Where(new OverdueOrderSpec())
    .ToListAsync(ct);
```

## Minimal API Endpoints (Trellis.Asp)

Endpoints are thin — send command via Mediator, convert Result to HTTP:

```csharp
// Standard error mapping
app.MapPost("/api/orders/{id}/submission", async (Guid id, IMediator mediator) =>
    await mediator.Send(new SubmitOrderCommand(OrderId.Create(id)))
        .ToMinimalApiResult());

// 201 Created with Location header via MatchError
app.MapPost("/api/customers", async (CreateCustomerRequest request, IMediator mediator) =>
    await mediator.Send(new CreateCustomerCommand(request.FirstName, request.LastName, request.Email))
        .MatchError(
            customer => Results.Created($"/api/customers/{customer.Id}", customer.ToResponse()),
            error => error.ToMinimalApiResult()));
```

**Every endpoint must have:**
- `.AddServiceLevelIndicator()` chained after the route handler
- API versioning via query parameter `?api-version=2026-11-12`
- Error responses as RFC 9457 Problem Details

**Request/Response DTOs** live in `Api/Contracts/`. Never expose domain types directly.

## Testing (Trellis.Testing)

Use Trellis.Testing assertion extensions with xUnit + FluentAssertions:

```csharp
// Result assertions
result.ShouldBeSuccess();
result.ShouldBeSuccess().Value.Should().Be(expected);
result.ShouldBeFailure();
result.ShouldBeFailure().WithError<ValidationError>();
result.ShouldBeFailure().WithError<ForbiddenError>();
result.ShouldBeFailure().WithError<NotFoundError>();

// Maybe assertions
maybe.ShouldBeSome();
maybe.ShouldBeSome().Value.Should().Be(expected);
maybe.ShouldBeNone();
```

**Domain tests:** Pure unit tests, no infrastructure. Test value object TryCreate, aggregate rules, state machine transitions, specifications.

**Application tests:** Mock repository interfaces. Test handler logic, authorization checks, error mapping.

**API integration tests:** Use `WebApplicationFactory<Program>` with SQLite/InMemory replacing SQL Server. Test HTTP round-trips, status codes, Problem Details, authorization enforcement via `X-Test-Actor` header.
