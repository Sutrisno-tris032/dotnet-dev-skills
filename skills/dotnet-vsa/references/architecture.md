# Arsitektur: Clean Architecture + Vertical Slice Architecture

## Mengapa menggabungkan CA dan VSA?

**Clean Architecture (CA)** memisahkan kode berdasarkan *lapisan tanggung jawab* (dependency rule: semua panah mengarah ke dalam). Masalahnya: ketika proyek besar, navigasi antar lapisan untuk satu fitur tersebar di banyak folder.

**Vertical Slice Architecture (VSA)** memisahkan kode berdasarkan *fitur bisnis*. Setiap fitur (slice) berdiri sendiri. Masalahnya: tanpa struktur, kode antar fitur bisa saling bergantung secara sembarangan.

**Gabungan keduanya:**
- Dependency rule CA tetap ditegakkan (Domain ← Application ← Infrastructure ← WebApi)
- Di dalam Application layer, kode diorganisir per fitur (VSA), bukan per tipe (Commands/, Queries/)
- Domain layer tetap *shared* karena entity dan value object memang dipakai lintas fitur

---

## Struktur Lapisan

### Domain Layer (`{ProjectName}.Domain`)

Pure C# class library. Hanya boleh referensi `MediatR.Contracts` (untuk `INotification` di `BaseEvent`). Tidak boleh ada EF Core, FluentValidation, atau package Application/Infrastructure lainnya.

```
Domain/
├── Common/
│   ├── BaseEntity.cs          ← Id, DomainEvents collection
│   ├── AuditableEntity.cs     ← CreatedAt, UpdatedAt, CreatedBy, UpdatedBy
│   └── ValueObject.cs         ← base class untuk value object
├── Entities/
│   └── {Entity}.cs
├── ValueObjects/
│   └── {ValueObject}.cs
├── Events/
│   └── {Entity}{Action}Event.cs
└── Exceptions/
    └── DomainException.cs
```

**BaseEntity:**
```csharp
public abstract class BaseEntity
{
    public int Id { get; protected set; }

    private readonly List<BaseEvent> _domainEvents = [];
    public IReadOnlyCollection<BaseEvent> DomainEvents => _domainEvents.AsReadOnly();

    public void AddDomainEvent(BaseEvent domainEvent) => _domainEvents.Add(domainEvent);
    public void RemoveDomainEvent(BaseEvent domainEvent) => _domainEvents.Remove(domainEvent);
    public void ClearDomainEvents() => _domainEvents.Clear();
}
```

**Aturan entity:**
- Field yang tidak boleh berubah sembarangan → gunakan `private set` atau constructor parameter
- Tidak ada method yang bergantung pada infrastruktur (DB, HTTP, dll.)
- Business rule ada di entity, bukan di handler

---

### Application Layer (`{ProjectName}.Application`)

Orchestration layer. Bergantung ke Domain. Mendefinisikan interface yang diimplementasi Infrastructure.

```
Application/
├── Common/
│   ├── Behaviors/
│   │   ├── ValidationBehavior.cs
│   │   └── LoggingBehavior.cs
│   ├── Exceptions/
│   │   ├── ValidationException.cs
│   │   └── NotFoundException.cs
│   ├── Interfaces/
│   │   ├── IApplicationDbContext.cs
│   │   └── ICurrentUserService.cs
│   └── Models/
│       └── ApiResponse.cs
├── Features/
│   └── {Feature}/
│       ├── Commands/
│       │   └── {Action}{Entity}/
│       │       ├── {Action}{Entity}Command.cs
│       │       ├── {Action}{Entity}CommandHandler.cs
│       │       └── {Action}{Entity}CommandValidator.cs
│       └── Queries/
│           └── Get{Entity}{Scope}/
│               ├── Get{Entity}{Scope}Query.cs
│               ├── Get{Entity}{Scope}QueryHandler.cs
│               └── {Entity}Response.cs
└── DependencyInjection.cs
```

**Pola setiap slice:**

```csharp
// Command — record, immutable
public record CreateOrderCommand(string CustomerId, List<OrderItemDto> Items) : IRequest<int>;

// Handler — satu per command/query
public class CreateOrderCommandHandler(IApplicationDbContext context)
    : IRequestHandler<CreateOrderCommand, int>
{
    public async Task<int> Handle(CreateOrderCommand request, CancellationToken ct)
    {
        var order = Order.Create(request.CustomerId, request.Items);
        context.Orders.Add(order);
        await context.SaveChangesAsync(ct);
        return order.Id;
    }
}

// Validator — FluentValidation
public class CreateOrderCommandValidator : AbstractValidator<CreateOrderCommand>
{
    public CreateOrderCommandValidator()
    {
        RuleFor(x => x.CustomerId).NotEmpty();
        RuleFor(x => x.Items).NotEmpty().ForEach(item => item.ChildRules(i =>
        {
            i.RuleFor(x => x.ProductId).NotEmpty();
            i.RuleFor(x => x.Quantity).GreaterThan(0);
        }));
    }
}
```

**DependencyInjection.cs:**
```csharp
public static class DependencyInjection
{
    public static IServiceCollection AddApplicationServices(this IServiceCollection services)
    {
        services.AddMediatR(cfg =>
        {
            cfg.RegisterServicesFromAssembly(Assembly.GetExecutingAssembly());
            cfg.AddBehavior(typeof(IPipelineBehavior<,>), typeof(LoggingBehavior<,>));
            cfg.AddBehavior(typeof(IPipelineBehavior<,>), typeof(ValidationBehavior<,>));
        });

        services.AddValidatorsFromAssembly(Assembly.GetExecutingAssembly());

        return services;
    }
}
```

---

### Infrastructure Layer (`{ProjectName}.Infrastructure`)

Implementasi semua interface dari Application. Bergantung ke Application dan Domain.

```
Infrastructure/
├── Persistence/
│   ├── ApplicationDbContext.cs
│   ├── Configurations/
│   │   └── {Entity}Configuration.cs
│   ├── Migrations/
│   └── Interceptors/
│       └── AuditableEntityInterceptor.cs
├── Services/
│   └── CurrentUserService.cs
└── DependencyInjection.cs
```

**IApplicationDbContext (di Application layer):**
```csharp
public interface IApplicationDbContext
{
    DbSet<Order> Orders { get; }
    DbSet<Product> Products { get; }
    Task<int> SaveChangesAsync(CancellationToken cancellationToken);
}
```

**ApplicationDbContext:**
```csharp
public class ApplicationDbContext(DbContextOptions<ApplicationDbContext> options)
    : DbContext(options), IApplicationDbContext
{
    public DbSet<Order> Orders => Set<Order>();
    public DbSet<Product> Products => Set<Product>();

    protected override void OnModelCreating(ModelBuilder builder)
    {
        builder.ApplyConfigurationsFromAssembly(Assembly.GetExecutingAssembly());
        base.OnModelCreating(builder);
    }
}
```

---

### WebApi Layer (`{ProjectName}.WebApi`)

Titik masuk aplikasi. Hanya wiring DI, middleware, dan endpoint. Tidak ada business logic.

Pilih salah satu pola endpoint:

**Option A — Controllers (familiar, mudah di-test dengan WebApplicationFactory):**
```csharp
[ApiController]
[Route("api/[controller]")]
public class OrdersController(ISender sender) : ControllerBase
{
    [HttpPost]
    public async Task<ActionResult<ApiResponse<int>>> Create(CreateOrderCommand command, CancellationToken ct)
    {
        var id = await sender.Send(command, ct);
        return Ok(ApiResponse<int>.Ok(id));
    }

    [HttpGet("{id:int}")]
    public async Task<ActionResult<ApiResponse<OrderResponse>>> GetById(int id, CancellationToken ct)
    {
        var result = await sender.Send(new GetOrderByIdQuery(id), ct);
        return Ok(ApiResponse<OrderResponse>.Ok(result));
    }
}
```

**Option B — Carter Minimal API (lebih minimal, endpoint lebih explicit):**

Carter memerlukan `AddCarter()` dan `MapCarter()` di `Program.cs` agar endpoint terdaftar.

```csharp
public class OrderEndpoints : ICarterModule
{
    public void AddRoutes(IEndpointRouteBuilder app)
    {
        var group = app.MapGroup("/api/orders").WithTags("Orders");

        group.MapPost("/", async (CreateOrderCommand command, ISender sender, CancellationToken ct) =>
        {
            var id = await sender.Send(command, ct);
            return Results.Ok(ApiResponse<int>.Ok(id));
        }).Produces<ApiResponse<int>>(StatusCodes.Status201Created);

        group.MapGet("/{id:int}", async (int id, ISender sender, CancellationToken ct) =>
        {
            var result = await sender.Send(new GetOrderByIdQuery(id), ct);
            return Results.Ok(ApiResponse<OrderResponse>.Ok(result));
        }).Produces<ApiResponse<OrderResponse>>()
          .ProducesProblem(StatusCodes.Status404NotFound);
    }
}
```

---

## Pipeline Behavior

Urutan pipeline behavior yang direkomendasikan:

```
Request → LoggingBehavior → ValidationBehavior → Handler → Response
```

**ValidationBehavior:**
```csharp
public class ValidationBehavior<TRequest, TResponse>(IEnumerable<IValidator<TRequest>> validators)
    : IPipelineBehavior<TRequest, TResponse>
    where TRequest : notnull
{
    public async Task<TResponse> Handle(TRequest request, RequestHandlerDelegate<TResponse> next, CancellationToken ct)
    {
        if (!validators.Any()) return await next();

        var context = new ValidationContext<TRequest>(request);
        var errors = new List<FluentValidation.Results.ValidationFailure>();

        // Sekuensial — DbContext tidak thread-safe; Task.WhenAll bisa menyebabkan race condition
        foreach (var validator in validators)
        {
            var result = await validator.ValidateAsync(context, ct);
            errors.AddRange(result.Errors);
        }

        if (errors.Count != 0)
            throw new Exceptions.ValidationException(errors);

        return await next();
    }
}
```

---

## Keputusan Desain Penting

| Pertanyaan | Jawaban yang direkomendasikan |
|---|---|
| Pakai Repository pattern? | Tidak — `IApplicationDbContext` sudah cukup sebagai abstraksi. Repository di atas EF Core sering over-engineering. |
| Pakai AutoMapper atau Mapster? | Mapster lebih cepat dan lebih explicit. Tapi jika tim sudah familiar AutoMapper, tidak masalah. Alternatif: manual mapping di handler (paling transparan). |
| Domain Event vs Integration Event? | Domain Event: di-dispatch dalam transaksi yang sama (MediatR `INotification`). Integration Event: lintas bounded context / service (pakai message broker). |
| Generic repository? | Hindari — `IRepository<T>` sering bocor abstraksi dan susah di-query secara LINQ-spesifik. |
| Async semua? | Ya — semua handler, semua query ke DB harus async. |
