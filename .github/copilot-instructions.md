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
Api → Acl → Application → Domain
```

| Layer | Depends On | Contains |
|-------|-----------|----------|
| **Domain** | Trellis packages only (Results, Primitives, DDD, Stateless, Authorization) | Aggregates, entities, value objects, domain events, specifications, permission constants |
| **Application** | Domain, Mediator, Trellis.Mediator | Commands, queries, handlers, repository interfaces |
| **Acl** | Application, Trellis.EntityFrameworkCore, EF Core provider | DbContext, entity configurations, repository implementations, migrations |
| **Api** | Application, Acl, Trellis.Asp | Endpoints, DTOs, Program.cs (composition root), IActorProvider implementation |

> **Why "Acl"?** ACL stands for Anti-Corruption Layer. This avoids confusion with actual infrastructure (servers, databases, cloud services). The Acl layer adapts external systems (SQL Server, message queues, etc.) to the domain model through repository implementations and EF Core.

**Rules:**
- Domain has ZERO external dependencies (no EF Core, no ASP.NET, no Mediator).
- Repository interfaces live in Application, implementations in Acl.
- `Mediator.SourceGenerator` is installed in the **Application** project (where commands and queries are defined).
- Each layer has one `DependencyInjection.cs` with an `Add{Layer}()` extension method.

## Project Layout

Uses the `.slnx` (XML-based) solution format.

```
{ServiceName}/
├── {ServiceName}.slnx
├── Directory.Build.props
├── Directory.Packages.props
├── global.json
├── .editorconfig
├── .runsettings
├── build/
│   └── test.props
├── Domain/
│   ├── src/
│   │   └── Domain.csproj
│   └── tests/
│       └── Domain.Tests.csproj
├── Application/
│   ├── src/
│   │   └── Application.csproj
│   └── tests/
│       └── Application.Tests.csproj
├── Acl/
│   ├── src/
│   │   └── AntiCorruptionLayer.csproj
│   └── tests/
│       └── AntiCorruptionLayer.Tests.csproj
└── Api/
    ├── src/
    │   └── Api.csproj
    └── tests/
        └── Api.Tests.csproj
```

Project names are short (`Domain.csproj`, not `{ServiceName}.Domain.csproj`). The `RootNamespace` is set automatically in `Directory.Build.props`:

```xml
<PropertyGroup Condition=" '$(IsTestProject)' == 'false' ">
  <RootNamespace>{ServiceName}.$(MSBuildProjectName.Replace(" ", "_"))</RootNamespace>
  <AssemblyName>{ServiceName}.$(MSBuildProjectName)</AssemblyName>
</PropertyGroup>
```

## Build System

Use Central Package Management (`Directory.Packages.props`) and shared build props. **Never duplicate configuration across projects.**

### Directory.Build.props

Shared by ALL projects. Key features:
- Sets `TargetFramework`, `Nullable`, `ImplicitUsings`, `LangVersion`.
- Adds `<Using Include="Trellis" />` and `Trellis.Analyzers` to every project.
- Detects test projects via `IsTestProject` and auto-imports `build/test.props`.
- Sets `RootNamespace` and `AssemblyName` using the service name prefix for non-test projects.

```xml
<Project xmlns="http://schemas.microsoft.com/developer/msbuild/2003">
  <Sdk Name="DotNet.ReproducibleBuilds.Isolated" Version="1.1.1" />
  <PropertyGroup Label="General">
    <Authors>{Author}</Authors>
    <Company>$(Authors)</Company>
    <SolutionDir Condition=" '$(SolutionDir)' == '' OR '$(SolutionDir)' == '*Undefined*' ">$(MSBuildThisFileDirectory)</SolutionDir>
    <IsTestProject>$(MSBuildProjectName.EndsWith('.Tests'))</IsTestProject>
    <TargetFramework>net10.0</TargetFramework>
    <Nullable>enable</Nullable>
    <ImplicitUsings>enable</ImplicitUsings>
    <LangVersion>Latest</LangVersion>
    <AnalysisLevel>latest-Recommended</AnalysisLevel>
  </PropertyGroup>
  <PropertyGroup Label="Build">
    <TreatWarningsAsErrors>true</TreatWarningsAsErrors>
    <ManagePackageVersionsCentrally>true</ManagePackageVersionsCentrally>
    <EnforceCodeStyleInBuild>true</EnforceCodeStyleInBuild>
  </PropertyGroup>
  <ItemGroup>
    <InternalsVisibleTo Include="$(MSBuildProjectName).Tests" />
    <Using Include="Trellis" />
    <PackageReference Include="Trellis.Analyzers" />
  </ItemGroup>
  <PropertyGroup Condition=" '$(IsTestProject)' == 'false' ">
    <RootNamespace>{ServiceName}.$(MSBuildProjectName.Replace(" ", "_"))</RootNamespace>
    <AssemblyName>{ServiceName}.$(MSBuildProjectName)</AssemblyName>
  </PropertyGroup>
  <ImportGroup Condition=" '$(IsTestProject)' == 'true' ">
    <Import Project="$(MSBuildThisFileDirectory)build/test.props"/>
  </ImportGroup>
</Project>
```

### build/test.props

Shared by ALL test projects (auto-imported via `Directory.Build.props` — no manual import needed in each test `.csproj`):

```xml
<?xml version="1.0" encoding="utf-8"?>
<Project xmlns="http://schemas.microsoft.com/developer/msbuild/2003">
  <PropertyGroup>
    <OutputType>Exe</OutputType>
    <UseMicrosoftTestingPlatformRunner>true</UseMicrosoftTestingPlatformRunner>
    <TestingPlatformDotnetTestSupport>true</TestingPlatformDotnetTestSupport>
    <NoWarn>$(NoWarn),CA1707</NoWarn>
    <EnableAotAnalyzer>false</EnableAotAnalyzer>
    <EnableTrimAnalyzer>false</EnableTrimAnalyzer>
  </PropertyGroup>
  <ItemGroup>
    <Using Include="FluentAssertions" />
    <Using Include="Xunit" />
    <Using Include="Trellis" />
  </ItemGroup>
  <ItemGroup>
    <PackageReference Include="xunit.v3" />
    <PackageReference Include="FluentAssertions" />
    <PackageReference Include="Microsoft.Testing.Extensions.CodeCoverage" />
    <PackageReference Include="Microsoft.Testing.Extensions.TrxReport" />
  </ItemGroup>
</Project>
```

**Do NOT** create `GlobalUsings.cs` files in test projects. The global usings come from `build/test.props` via `<Using Include="..." />`.

Test-specific packages (e.g., `Trellis.Testing`, `Microsoft.AspNetCore.Mvc.Testing`) go in the individual test project that needs them, not in `build/test.props`.

### Project Dependencies (.csproj examples)

**Domain.csproj** — no project references, only Trellis packages:
```xml
<Project Sdk="Microsoft.NET.Sdk">
  <ItemGroup>
    <PackageReference Include="Trellis.DomainDrivenDesign" />
    <PackageReference Include="Trellis.Primitives" />
    <PackageReference Include="Trellis.Primitives.Generator" OutputItemType="Analyzer" ReferenceOutputAssembly="false" />
    <PackageReference Include="Trellis.Results" />
    <PackageReference Include="Trellis.Stateless" />
    <PackageReference Include="Trellis.Authorization" />
  </ItemGroup>
</Project>
```

**Application.csproj** — references Domain, has Mediator + SourceGenerator:
```xml
<Project Sdk="Microsoft.NET.Sdk">
  <ItemGroup>
    <PackageReference Include="Mediator.Abstractions" />
    <PackageReference Include="Mediator.SourceGenerator">
      <PrivateAssets>all</PrivateAssets>
      <IncludeAssets>runtime; build; native; contentfiles; analyzers; buildtransitive</IncludeAssets>
    </PackageReference>
    <PackageReference Include="Trellis.Mediator" />
  </ItemGroup>
  <ItemGroup>
    <ProjectReference Include="..\..\Domain\src\Domain.csproj" />
  </ItemGroup>
</Project>
```

**AntiCorruptionLayer.csproj** — references Application, has EF Core:
```xml
<Project Sdk="Microsoft.NET.Sdk">
  <ItemGroup>
    <PackageReference Include="Trellis.EntityFrameworkCore" />
    <PackageReference Include="Microsoft.EntityFrameworkCore.SqlServer" />
  </ItemGroup>
  <ItemGroup>
    <ProjectReference Include="..\..\Application\src\Application.csproj" />
  </ItemGroup>
</Project>
```

**Api.csproj** — Web SDK, references Application + Acl:
```xml
<Project Sdk="Microsoft.NET.Sdk.Web">
  <ItemGroup>
    <PackageReference Include="Trellis.Asp" />
    <PackageReference Include="Asp.Versioning.Mvc.ApiExplorer" />
    <PackageReference Include="ServiceLevelIndicators.Asp.ApiVersioning" />
  </ItemGroup>
  <ItemGroup>
    <ProjectReference Include="..\..\Application\src\Application.csproj" />
    <ProjectReference Include="..\..\Acl\src\AntiCorruptionLayer.csproj" />
  </ItemGroup>
</Project>
```

## Value Objects

Use `Trellis.Primitives` base classes with source generator (`Trellis.Primitives.Generator`):

| Base Class | Backing Type | Example |
|-----------|-------------|---------|
| `RequiredGuid<T>` | `Guid` | Identity types (`UserId`, `ProductId`) |
| `RequiredString<T>` | `string` | Text values (`FirstName`, `LastName`, `Sku`) |
| `RequiredInt<T>` | `int` | Integer values (`Quantity`, `StockQuantity`) |
| `RequiredDecimal<T>` | `decimal` | Decimal values (`Percentage`, `Weight`) |
| `RequiredEnum<T>` | `string` (smart enum) | Status types (`OrderStatus`) |

**Identity types** always use `RequiredGuid` with `Guid.CreateVersion7()` for time-sortable IDs:
```csharp
public sealed partial class UserId : RequiredGuid<UserId>
{
    public static UserId NewUnique() => Create(Guid.CreateVersion7());
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
}
```

**Multi-field value objects** extend `ValueObject`:
```csharp
public sealed class Address : ValueObject
{
    public string Street { get; }
    public string City { get; }
    // ...
    protected override IEnumerable<IComparable> GetEqualityComponents()
    {
        yield return Street;
        yield return City;
    }
}
```

**Custom validation** with `FluentValidation` and `Trellis.FluentValidation`:
```csharp
public class ZipCode : ScalarValueObject<ZipCode, string>, IScalarValue<ZipCode, string>
{
    private ZipCode(string value) : base(value) { }

    public static Result<ZipCode> TryCreate(string value, string? fieldName = null)
    {
        var zipCode = new ZipCode(value);
        return s_validation.ValidateToResult(zipCode);
    }

    static readonly InlineValidator<ZipCode> s_validation = new()
    {
        v => v.RuleFor(x => x.Value).NotEmpty().Matches(@"^\d{5}$")
    };
}
```

## Result\<T\> and ROP Chains

Every operation returns `Result<T>`. Chain with Bind/Map/Tap/Ensure:

```csharp
public async Task<Result<Order>> Handle(CreateDraftOrderCommand cmd, CancellationToken ct)
    => await _customerRepo.GetByIdAsync(cmd.CustomerId, ct)
        .BindAsync(customer => CreateOrderWithItems(customer, cmd.LineItems))
        .TapAsync(order => _orderRepo.SaveAsync(order, ct));
```

Note: Commands receive value object types (e.g., `CustomerId`, not `Guid`). Handlers never call `TryCreate` on command properties — scalar value binding already validated them at the API layer.

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

**Combine pattern** for validating multiple inputs (used in domain factory methods and tests — not in handlers, since commands already contain value objects):
```csharp
FirstName.TryCreate(firstName)
    .Combine(LastName.TryCreate(lastName))
    .Combine(EmailAddress.TryCreate(email))
    .Bind((first, last, email) => User.TryCreate(first, last, email));
```

**ParallelAsync** for concurrent fetches:
```csharp
var result = await _customerRepo.GetByIdAsync(customerId, ct)
    .ParallelAsync(_productRepo.GetByIdsAsync(productIds, ct))
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
// Aggregate — root entity with identity
public sealed class User : Aggregate<UserId>
{
    public FirstName FirstName { get; }
    public LastName LastName { get; }
    public EmailAddress Email { get; }

    public static Result<User> TryCreate(FirstName firstName, LastName lastName, EmailAddress email)
    {
        var user = new User(firstName, lastName, email);
        return validator.ValidateToResult(user);
    }

    private User(FirstName firstName, LastName lastName, EmailAddress email)
        : base(UserId.NewUnique()) { ... }
}

// Entity (owned by an aggregate)
public sealed class LineItem : Entity<LineItemId> { ... }

// Specification (composable query filter)
public sealed class ActiveUsersSpec : Specification<User>
{
    public override Expression<Func<User, bool>> ToExpression() =>
        user => user.IsActive && user.CreatedAt > DateTime.UtcNow.AddDays(-30);
}
```

**Domain events** — record types for cross-aggregate communication:
```csharp
public sealed record UserCreatedEvent(UserId UserId, EmailAddress Email, DateTime CreatedAt);
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
// Command with authorization — no IValidate needed, value objects self-validate via TryCreate
public sealed record CreateUserCommand(FirstName FirstName, LastName LastName, EmailAddress Email)
    : ICommand<Result<User>>, IAuthorize
{
    public IReadOnlyList<string> RequiredPermissions => [Permissions.UsersCreate];
}

// IValidate — only for cross-field or collection validation that value objects cannot capture
public sealed record CreateDraftOrderCommand(CustomerId CustomerId, IReadOnlyList<LineItemInput> LineItems)
    : ICommand<Result<Order>>, IAuthorize, IValidate
{
    public IReadOnlyList<string> RequiredPermissions => [Permissions.OrdersCreate];

    public IResult Validate() =>
        LineItems.Count == 0
            ? Result.Failure(Error.Validation("At least one line item is required", "lineItems"))
            : Result.Success();
}

// Resource-based authorization
public sealed record DeleteUserCommand(UserId UserId)
    : ICommand<Result<Unit>>, IAuthorize, IAuthorizeResource
{
    public IReadOnlyList<string> RequiredPermissions => [Permissions.UsersDelete];

    public IResult Authorize(Actor actor) =>
        actor.Id == UserId.Value.ToString() || actor.HasPermission(Permissions.Admin)
            ? Result.Success()
            : Result.Failure(Error.Forbidden("Only the owner or admin can delete"));
}

// Query
public sealed record GetUserQuery(UserId UserId)
    : IQuery<Result<User>>, IAuthorize
{
    public IReadOnlyList<string> RequiredPermissions => [Permissions.UsersRead];
}
```

**Pipeline behavior order** (registered via `ServiceCollectionExtensions.PipelineBehaviors`):
```
Request → ExceptionBehavior → TracingBehavior → LoggingBehavior
  → AuthorizationBehavior → ResourceAuthorizationBehavior
    → ValidationBehavior → Handler → Result<T>
```

**Registration in Application/DependencyInjection.cs:**
```csharp
public static IServiceCollection AddApplication(this IServiceCollection services)
{
    services.AddMediator(options =>
    {
        options.ServiceLifetime = ServiceLifetime.Scoped;
        options.PipelineBehaviors = ServiceCollectionExtensions.PipelineBehaviors;
    });
    return services;
}
```

## Authorization (Trellis.Authorization)

**Permissions** are string constants in the Domain layer:
```csharp
public static class Permissions
{
    public const string UsersCreate = "users:create";
    public const string UsersRead = "users:read";
    public const string UsersDelete = "users:delete";
    public const string Admin = "admin";
}
```

**Actor** represents the authenticated user:
```csharp
var actor = Actor.Create("user-42", new HashSet<string> { "users:create", "users:read" });
actor.HasPermission("users:create");  // true
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
    configurationBuilder.ApplyTrellisConventions(typeof(UserId).Assembly);
    // Scans for IScalarValue<TSelf, TPrimitive> and RequiredEnum<TSelf>
    // Trellis.Primitives assembly is always included automatically
}
```

**NEVER write `HasConversion()`.** ApplyTrellisConventions handles all Trellis value objects.

**OnModelCreating** — configure keys, indexes, constraints only:
```csharp
modelBuilder.Entity<User>(b =>
{
    b.HasKey(c => c.Id);
    b.Property(c => c.FirstName).HasMaxLength(100).IsRequired();
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
Maybe<User> user = await context.Users
    .FirstOrDefaultMaybeAsync(c => c.Id == userId, ct);

Result<User> user = await context.Users
    .FirstOrDefaultResultAsync(c => c.Id == userId, Error.NotFound("User", id), ct);
```

**Specification queries:**
```csharp
var activeUsers = await context.Users
    .Where(new ActiveUsersSpec())
    .ToListAsync(ct);
```

## MVC Controllers (Trellis.Asp)

Controllers inherit `ControllerBase` with `[ApiController]`. Actions are thin — send command via Mediator, chain `.ToActionResult(this)` or `.ToActionResultAsync(this)`:

```csharp
[ApiController]
[Route("api/[controller]")]
[ApiVersion("2026-11-12")]
[Consumes("application/json")]
[Produces("application/json")]
[ServiceLevelIndicator]
public class UsersController(ISender sender) : ControllerBase
{
    // Value object types directly in action parameters — Trellis binds automatically
    [HttpGet("{id:guid}")]
    public async Task<ActionResult<UserResponse>> Get(UserId id, CancellationToken ct)
        => await sender.Send(new GetUserQuery(id), ct)
            .MapAsync(user => user.ToResponse())
            .ToActionResultAsync(this);

    // 201 Created with Location header
    [HttpPost]
    public async Task<ActionResult<UserResponse>> Create(
        CreateUserRequest request, CancellationToken ct)
        => await sender.Send(new CreateUserCommand(...), ct)
            .ToCreatedAtActionResult(this, nameof(Get),
                user => new { id = user.Id },
                user => user.ToResponse());
}
```

**Every controller must have:**
- `[ApiController]` attribute and inherit `ControllerBase`
- `[ApiVersion("2026-11-12")]` at class level
- `[ServiceLevelIndicator]` at class level
- `[Route("api/[controller]")]` at class level
- `[Consumes("application/json")]` and `[Produces("application/json")]` at class level
- Error responses as RFC 9457 Problem Details (handled by `ToActionResult`)

### Automatic Scalar Value Binding

**Use value object types — not primitives — in controller action parameters.** Trellis automatically converts route parameters, query parameters, and JSON body properties via model binding and JSON converters. Never call `.Create()` or `.TryCreate()` manually in controllers.

**How it works:**
- **Route/query parameters:** `ScalarValueModelBinderProvider` detects any type implementing `IScalarValue<TSelf, TPrimitive>` and binds the raw value through `TryCreate`. If validation fails, the error is collected (not thrown).
- **JSON body:** `ValidatingJsonConverterFactory` handles deserialization of scalar value types in request bodies. All validation errors across all properties are collected before returning a 400 response.
- **Maybe\<T\> parameters:** Route/query parameters of type `Maybe<T>` are supported — absent values produce `Maybe.None`, present values are validated through `TryCreate`.
- **Error accumulation:** Trellis collects ALL scalar value validation failures across the entire request (route + query + body) and returns them together as a single Problem Details 400 response. This gives callers complete feedback in one round-trip.

**Registration** — add scalar value validation to the MVC pipeline in `Api/src/DependencyInjection.cs`:
```csharp
services.AddControllers().AddScalarValueValidation();
```
And activate the middleware in `Program.cs`:
```csharp
app.UseScalarValueValidation();
```

**Request/Response DTOs** live in `Api/src/Contracts/`. Never expose domain types directly. Request DTOs can use scalar value object types as properties — they will be validated automatically via the JSON converter.

**SLI resource IDs** — use `[CustomerResourceId]` on route parameters that identify the customer resource for Service Level Indicators.

**API Versioning** — use date-based versions (e.g., `2026-11-12`). Controllers are organized in versioned folders under `Api/src/` (e.g., `Api/src/2026-11-12/Controllers/`).

## Testing (Trellis.Testing)

Use Trellis.Testing assertion extensions with xUnit + FluentAssertions:

```csharp
// Result assertions
result.Should().BeSuccess();
result.Should().BeSuccess().Which.Should().Be(expected);
result.Should().BeFailure();
result.Should().BeFailureOfType<ValidationError>();
result.Should().BeFailureOfType<ForbiddenError>();
result.Should().BeFailureOfType<NotFoundError>();

// Maybe assertions
maybe.Should().HaveValue();
maybe.Should().HaveValue().Which.Should().Be(expected);
maybe.Should().BeNone();
```

**Domain tests:** Pure unit tests, no external dependencies. Test value object TryCreate, aggregate rules, state machine transitions, specifications.

**Application tests:** Mock repository interfaces. Test handler logic, authorization checks, error mapping. Use `Xunit.DependencyInjection` for test DI with a `Startup.cs` that registers Mediator and mock services.

**API integration tests:** Use `WebApplicationFactory<Program>` with SQLite/InMemory replacing the real database. Test HTTP round-trips, status codes, Problem Details, authorization enforcement. Use `MartinCostello.Logging.XUnit.v3` for test logging.

```csharp
// Test .csproj for API integration tests (no manual test.props import needed):
<Project Sdk="Microsoft.NET.Sdk">
  <ItemGroup>
    <ProjectReference Include="..\src\Api.csproj" />
  </ItemGroup>
  <ItemGroup>
    <PackageReference Include="MartinCostello.Logging.XUnit.v3" />
    <PackageReference Include="Microsoft.AspNetCore.Mvc.Testing" />
    <PackageReference Include="Trellis.Testing" />
  </ItemGroup>
</Project>
```

## Trellis Feedback

While building with Trellis, **actively track friction points, workarounds, and missing capabilities.** At the end of the project (or at any significant milestone), generate a `TRELLIS_FEEDBACK.md` file in the repository root.

This feedback helps the Trellis team identify gaps in the framework and prioritize future improvements. **Generate this file proactively** — do not wait to be asked.

### When to Record Feedback

- You had to write boilerplate that Trellis should have handled
- You worked around a missing pattern or building block
- A Trellis API was confusing or required reading source code to understand
- You wished a base class, interface, or extension method existed but it didn't
- The copilot instructions were ambiguous or missing guidance for a scenario you encountered
- An error message from Trellis was unhelpful or misleading
- You had to make an architectural decision that Trellis should have constrained
- A common .NET pattern (middleware, DI, configuration) wasn't covered by Trellis conventions

### Feedback File Format

Generate `TRELLIS_FEEDBACK.md` with this structure:

```markdown
# Trellis Feedback — {ServiceName}

> Generated by AI while building {ServiceName} on {date}.
> Trellis version: {version from Directory.Packages.props}
> AI model: {model name}

## Summary

{1-2 sentence overall assessment of the development experience with Trellis}

## Friction Points

### FP-1: {Short title}
- **Category:** Missing Building Block | Workaround Required | Ambiguous API | Missing Documentation | Error Message | Architectural Gap
- **Severity:** High (blocked progress) | Medium (slowed progress) | Low (minor inconvenience)
- **Context:** {What were you trying to do?}
- **What happened:** {What went wrong or was harder than expected?}
- **Workaround used:** {What you did instead, if anything}
- **Suggested improvement:** {What Trellis could add or change}

### FP-2: ...

## What Worked Well

{List of Trellis features that were particularly effective or easy to use. This helps the team know what NOT to change.}

## Suggested New Features

### SF-1: {Feature name}
- **Use case:** {When would this be useful?}
- **Proposed API:** {Sketch of what the API could look like}

### SF-2: ...

## Copilot Instructions Feedback

{Any sections of the copilot instructions that were unclear, missing, or led to incorrect code generation. Be specific about which section and what was confusing.}
```

### Rules

- **Be specific.** Include the exact code you wrote as a workaround. Vague feedback like "EF Core was hard" is not actionable.
- **One friction point per entry.** Don't combine unrelated issues.
- **Include severity.** This helps the Trellis team prioritize.
- **Credit what works.** The "What Worked Well" section is equally important — it prevents regressions.
- **If nothing went wrong, say so.** A feedback file with zero friction points and a strong "What Worked Well" section is valuable data.
