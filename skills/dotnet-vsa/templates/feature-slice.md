# Template Feature Slice — CRUD Lengkap

Template ini digunakan saat menjalankan `new-feature`. Ganti:
- `{ProjectName}` → nama solution
- `{Feature}` → nama fitur/modul (plural, PascalCase) — contoh: `Orders`, `Products`
- `{Entity}` → nama entity (singular, PascalCase) — contoh: `Order`, `Product`
- `{entity}` → nama entity (singular, camelCase) — contoh: `order`, `product`
- `{feature}` → nama fitur/modul (plural, camelCase/lowercase) — contoh: `orders`, `products`
  (**bukan** route parameter ASP.NET — ini substitution variable yang diganti langsung ke nilai seperti `orders`)

---

## 1. Domain Entity

### `src/{ProjectName}.Domain/Entities/{Entity}.cs`
```csharp
namespace {ProjectName}.Domain.Entities;

public class {Entity} : AuditableEntity
{
    // Gunakan private constructor + factory method untuk menjaga invariant
    private {Entity}() { }

    public static {Entity} Create(/* parameter sesuai kebutuhan */)
    {
        var {entity} = new {Entity}
        {
            // set properties
        };

        // Opsional: raise domain event
        // {entity}.AddDomainEvent(new {Entity}CreatedEvent({entity}));

        return {entity};
    }

    // Properties — gunakan private set untuk field yang tidak boleh berubah langsung
    public string Name { get; private set; } = string.Empty;
    // tambahkan property sesuai kebutuhan domain

    // Update — method domain agar mutasi terpusat dan tidak dapat diakses langsung dari luar
    public void Update(string name)
    {
        Name = name;
        // update property lain sesuai kebutuhan
    }
}
```

---

## 2. EF Core Configuration

### `src/{ProjectName}.Infrastructure/Persistence/Configurations/{Entity}Configuration.cs`
```csharp
using Microsoft.EntityFrameworkCore;
using Microsoft.EntityFrameworkCore.Metadata.Builders;
using {ProjectName}.Domain.Entities;

namespace {ProjectName}.Infrastructure.Persistence.Configurations;

public class {Entity}Configuration : IEntityTypeConfiguration<{Entity}>
{
    public void Configure(EntityTypeBuilder<{Entity}> builder)
    {
        builder.ToTable("{Feature}");  // nama tabel, biasanya plural

        builder.HasKey(e => e.Id);

        builder.Property(e => e.Name)
            .IsRequired()
            .HasMaxLength(200);

        // tambahkan konfigurasi kolom lainnya
    }
}
```

---

## 3. Update IApplicationDbContext & ApplicationDbContext

### Di `Application/Common/Interfaces/IApplicationDbContext.cs`, tambahkan:
```csharp
DbSet<{Entity}> {Feature} { get; }
```

### Di `Infrastructure/Persistence/ApplicationDbContext.cs`, tambahkan:
```csharp
public DbSet<{Entity}> {Feature} => Set<{Entity}>();
```

---

## 4. Queries

### `Application/Features/{Feature}/Queries/Get{Entity}ById/Get{Entity}ByIdQuery.cs`
```csharp
namespace {ProjectName}.Application.Features.{Feature}.Queries.Get{Entity}ById;

public record Get{Entity}ByIdQuery(int Id) : IRequest<{Entity}Response>;
```

### `Application/Features/{Feature}/Queries/Get{Entity}ById/{Entity}Response.cs`
```csharp
namespace {ProjectName}.Application.Features.{Feature}.Queries.Get{Entity}ById;

public record {Entity}Response(
    int Id,
    string Name
    // tambahkan property sesuai entity
);
```

### `Application/Features/{Feature}/Queries/Get{Entity}ById/Get{Entity}ByIdQueryHandler.cs`
```csharp
namespace {ProjectName}.Application.Features.{Feature}.Queries.Get{Entity}ById;

public class Get{Entity}ByIdQueryHandler(IApplicationDbContext context)
    : IRequestHandler<Get{Entity}ByIdQuery, {Entity}Response>
{
    public async Task<{Entity}Response> Handle(Get{Entity}ByIdQuery request, CancellationToken ct)
    {
        var {entity} = await context.{Feature}
            .AsNoTracking()
            .Where(x => x.Id == request.Id)
            .Select(x => new {Entity}Response(x.Id, x.Name))
            .FirstOrDefaultAsync(ct)
            ?? throw new NotFoundException(nameof({Entity}), request.Id);

        return {entity};
    }
}
```

### `Application/Features/{Feature}/Queries/Get{Entity}List/Get{Entity}ListQuery.cs`
```csharp
namespace {ProjectName}.Application.Features.{Feature}.Queries.Get{Entity}List;

public record Get{Entity}ListQuery(
    int PageNumber = 1,
    int PageSize = 10,
    string? SearchTerm = null
) : IRequest<List<{Entity}ListResponse>>;
```

### `Application/Features/{Feature}/Queries/Get{Entity}List/{Entity}ListResponse.cs`
```csharp
namespace {ProjectName}.Application.Features.{Feature}.Queries.Get{Entity}List;

public record {Entity}ListResponse(int Id, string Name);
```

### `Application/Features/{Feature}/Queries/Get{Entity}List/Get{Entity}ListQueryHandler.cs`
```csharp
namespace {ProjectName}.Application.Features.{Feature}.Queries.Get{Entity}List;

public class Get{Entity}ListQueryHandler(IApplicationDbContext context)
    : IRequestHandler<Get{Entity}ListQuery, List<{Entity}ListResponse>>
{
    public async Task<List<{Entity}ListResponse>> Handle(Get{Entity}ListQuery request, CancellationToken ct)
    {
        return await context.{Feature}
            .AsNoTracking()
            .Where(x => request.SearchTerm == null || x.Name.Contains(request.SearchTerm))
            .OrderBy(x => x.Name)
            .Skip((request.PageNumber - 1) * request.PageSize)
            .Take(request.PageSize)
            .Select(x => new {Entity}ListResponse(x.Id, x.Name))
            .ToListAsync(ct);
    }
}
```

---

## 5. Commands — Create

### `Application/Features/{Feature}/Commands/Create{Entity}/Create{Entity}Command.cs`
```csharp
namespace {ProjectName}.Application.Features.{Feature}.Commands.Create{Entity};

public record Create{Entity}Command(string Name) : IRequest<int>;
```

### `Application/Features/{Feature}/Commands/Create{Entity}/Create{Entity}CommandValidator.cs`
```csharp
namespace {ProjectName}.Application.Features.{Feature}.Commands.Create{Entity};

public class Create{Entity}CommandValidator : AbstractValidator<Create{Entity}Command>
{
    public Create{Entity}CommandValidator()
    {
        RuleFor(x => x.Name)
            .NotEmpty().WithMessage("Name is required.")
            .MaximumLength(200).WithMessage("Name must not exceed 200 characters.");
    }
}
```

### `Application/Features/{Feature}/Commands/Create{Entity}/Create{Entity}CommandHandler.cs`
```csharp
namespace {ProjectName}.Application.Features.{Feature}.Commands.Create{Entity};

public class Create{Entity}CommandHandler(IApplicationDbContext context)
    : IRequestHandler<Create{Entity}Command, int>
{
    public async Task<int> Handle(Create{Entity}Command request, CancellationToken ct)
    {
        var {entity} = {Entity}.Create(request.Name);

        context.{Feature}.Add({entity});
        await context.SaveChangesAsync(ct);

        return {entity}.Id;
    }
}
```

---

## 6. Commands — Update

### `Application/Features/{Feature}/Commands/Update{Entity}/Update{Entity}Command.cs`
```csharp
namespace {ProjectName}.Application.Features.{Feature}.Commands.Update{Entity};

public record Update{Entity}Command(int Id, string Name) : IRequest;
```

### `Application/Features/{Feature}/Commands/Update{Entity}/Update{Entity}CommandValidator.cs`
```csharp
namespace {ProjectName}.Application.Features.{Feature}.Commands.Update{Entity};

public class Update{Entity}CommandValidator : AbstractValidator<Update{Entity}Command>
{
    public Update{Entity}CommandValidator()
    {
        RuleFor(x => x.Id).GreaterThan(0);
        RuleFor(x => x.Name)
            .NotEmpty().WithMessage("Name is required.")
            .MaximumLength(200).WithMessage("Name must not exceed 200 characters.");
    }
}
```

### `Application/Features/{Feature}/Commands/Update{Entity}/Update{Entity}CommandHandler.cs`
```csharp
namespace {ProjectName}.Application.Features.{Feature}.Commands.Update{Entity};

public class Update{Entity}CommandHandler(IApplicationDbContext context)
    : IRequestHandler<Update{Entity}Command>
{
    public async Task Handle(Update{Entity}Command request, CancellationToken ct)
    {
        // .NET 6/7: ganti FindAsync([request.Id], ct) → FindAsync(new object[] { request.Id }, ct)
        var {entity} = await context.{Feature}.FindAsync([request.Id], ct)
            ?? throw new NotFoundException(nameof({Entity}), request.Id);

        {entity}.Update(request.Name);

        await context.SaveChangesAsync(ct);
    }
}
```

---

## 7. Commands — Delete

### `Application/Features/{Feature}/Commands/Delete{Entity}/Delete{Entity}Command.cs`
```csharp
namespace {ProjectName}.Application.Features.{Feature}.Commands.Delete{Entity};

public record Delete{Entity}Command(int Id) : IRequest;
```

### `Application/Features/{Feature}/Commands/Delete{Entity}/Delete{Entity}CommandHandler.cs`
```csharp
namespace {ProjectName}.Application.Features.{Feature}.Commands.Delete{Entity};

public class Delete{Entity}CommandHandler(IApplicationDbContext context)
    : IRequestHandler<Delete{Entity}Command>
{
    public async Task Handle(Delete{Entity}Command request, CancellationToken ct)
    {
        // .NET 6/7: ganti FindAsync([request.Id], ct) → FindAsync(new object[] { request.Id }, ct)
        var {entity} = await context.{Feature}.FindAsync([request.Id], ct)
            ?? throw new NotFoundException(nameof({Entity}), request.Id);

        context.{Feature}.Remove({entity});
        await context.SaveChangesAsync(ct);
    }
}
```

---

## 8A. Endpoint — Controller

### `WebApi/Controllers/{Feature}Controller.cs`
```csharp
using MediatR;
using Microsoft.AspNetCore.Mvc;
using {ProjectName}.Application.Common.Models;
using {ProjectName}.Application.Features.{Feature}.Commands.Create{Entity};
using {ProjectName}.Application.Features.{Feature}.Commands.Update{Entity};
using {ProjectName}.Application.Features.{Feature}.Commands.Delete{Entity};
using {ProjectName}.Application.Features.{Feature}.Queries.Get{Entity}ById;
using {ProjectName}.Application.Features.{Feature}.Queries.Get{Entity}List;

namespace {ProjectName}.WebApi.Controllers;

[ApiController]
[Route("api/[controller]")]
public class {Feature}Controller(ISender sender) : ControllerBase
{
    [HttpGet]
    public async Task<ActionResult<ApiResponse<List<{Entity}ListResponse>>>> GetList(
        [FromQuery] int pageNumber = 1,
        [FromQuery] int pageSize = 10,
        [FromQuery] string? searchTerm = null,
        CancellationToken ct = default)
    {
        var result = await sender.Send(new Get{Entity}ListQuery(pageNumber, pageSize, searchTerm), ct);
        return Ok(ApiResponse<List<{Entity}ListResponse>>.Ok(result));
    }

    [HttpGet("{id:int}")]
    public async Task<ActionResult<ApiResponse<{Entity}Response>>> GetById(int id, CancellationToken ct)
    {
        var result = await sender.Send(new Get{Entity}ByIdQuery(id), ct);
        return Ok(ApiResponse<{Entity}Response>.Ok(result));
    }

    [HttpPost]
    public async Task<ActionResult<ApiResponse<int>>> Create(Create{Entity}Command command, CancellationToken ct)
    {
        var id = await sender.Send(command, ct);
        return CreatedAtAction(nameof(GetById), new { id }, ApiResponse<int>.Ok(id));
    }

    [HttpPut("{id:int}")]
    public async Task<ActionResult<ApiResponse<object?>>> Update(int id, Update{Entity}Command command, CancellationToken ct)
    {
        await sender.Send(command with { Id = id }, ct);
        return Ok(ApiResponse<object?>.Ok(null, "{Entity} updated successfully."));
    }

    [HttpDelete("{id:int}")]
    public async Task<ActionResult<ApiResponse<object?>>> Delete(int id, CancellationToken ct)
    {
        await sender.Send(new Delete{Entity}Command(id), ct);
        return Ok(ApiResponse<object?>.Ok(null, "{Entity} deleted successfully."));
    }
}
```

---

## 8B. Endpoint — Carter Minimal API

### `WebApi/Endpoints/{Feature}Endpoints.cs`
```csharp
using Carter;
using MediatR;
using {ProjectName}.Application.Common.Models;
using {ProjectName}.Application.Features.{Feature}.Commands.Create{Entity};
using {ProjectName}.Application.Features.{Feature}.Commands.Update{Entity};
using {ProjectName}.Application.Features.{Feature}.Commands.Delete{Entity};
using {ProjectName}.Application.Features.{Feature}.Queries.Get{Entity}ById;
using {ProjectName}.Application.Features.{Feature}.Queries.Get{Entity}List;

namespace {ProjectName}.WebApi.Endpoints;

public class {Feature}Endpoints : ICarterModule
{
    public void AddRoutes(IEndpointRouteBuilder app)
    {
        var group = app.MapGroup("/api/{feature}").WithTags("{Feature}");

        group.MapGet("/", async (
            [AsParameters] Get{Entity}ListQuery query,
            ISender sender,
            CancellationToken ct) =>
        {
            var result = await sender.Send(query, ct);
            return Results.Ok(ApiResponse<List<{Entity}ListResponse>>.Ok(result));
        }).Produces<ApiResponse<List<{Entity}ListResponse>>>();

        group.MapGet("/{id:int}", async (int id, ISender sender, CancellationToken ct) =>
        {
            var result = await sender.Send(new Get{Entity}ByIdQuery(id), ct);
            return Results.Ok(ApiResponse<{Entity}Response>.Ok(result));
        }).WithName("Get{Entity}ById")   // diperlukan agar CreatedAtRoute di MapPost bisa resolve
          .Produces<ApiResponse<{Entity}Response>>()
          .ProducesProblem(StatusCodes.Status404NotFound);

        group.MapPost("/", async (Create{Entity}Command command, ISender sender, CancellationToken ct) =>
        {
            var id = await sender.Send(command, ct);
            return Results.CreatedAtRoute("Get{Entity}ById", new { id }, ApiResponse<int>.Ok(id));
        }).Produces<ApiResponse<int>>(StatusCodes.Status201Created)
          .ProducesValidationProblem();

        group.MapPut("/{id:int}", async (int id, Update{Entity}Command command, ISender sender, CancellationToken ct) =>
        {
            await sender.Send(command with { Id = id }, ct);
            return Results.Ok(ApiResponse<object?>.Ok(null, "{Entity} updated successfully."));
        }).Produces<ApiResponse<object?>>()
          .ProducesValidationProblem()
          .ProducesProblem(StatusCodes.Status404NotFound);

        group.MapDelete("/{id:int}", async (int id, ISender sender, CancellationToken ct) =>
        {
            await sender.Send(new Delete{Entity}Command(id), ct);
            return Results.Ok(ApiResponse<object?>.Ok(null, "{Entity} deleted successfully."));
        }).Produces<ApiResponse<object?>>()
          .ProducesProblem(StatusCodes.Status404NotFound);
    }
}
```

---

## 9. Migration Command

Setelah semua file dibuat, jalankan:
```bash
# Dari root solution
dotnet ef migrations add Add{Entity} \
  --project src/{ProjectName}.Infrastructure \
  --startup-project src/{ProjectName}.WebApi

dotnet ef database update \
  --project src/{ProjectName}.Infrastructure \
  --startup-project src/{ProjectName}.WebApi
```

---

## 10. Unit Test Template

> **PENTING:** Unit test handler harus mock `IApplicationDbContext` dengan NSubstitute.
> **JANGAN** inject `ApplicationDbContext` atau buat `new ApplicationDbContext(options)` langsung di test —
> itu melanggar Clean Architecture (Application layer tidak boleh bergantung ke EF Core concrete).

### `tests/{ProjectName}.Application.Tests/Features/{Feature}/Commands/Create{Entity}CommandHandlerTests.cs`

```csharp
using NSubstitute;
using FluentAssertions;
using {ProjectName}.Application.Common.Interfaces;
using {ProjectName}.Application.Features.{Feature}.Commands.Create{Entity};
using {ProjectName}.Domain.Entities;
using Microsoft.EntityFrameworkCore;

namespace {ProjectName}.Application.Tests.Features.{Feature}.Commands;

public class Create{Entity}CommandHandlerTests
{
    private readonly IApplicationDbContext _context;
    private readonly DbSet<{Entity}> _{entity}Set;
    private readonly Create{Entity}CommandHandler _handler;

    public Create{Entity}CommandHandlerTests()
    {
        _context = Substitute.For<IApplicationDbContext>();
        _{entity}Set = Substitute.For<DbSet<{Entity}>>();
        _context.{Feature}.Returns(_{entity}Set);

        _handler = new Create{Entity}CommandHandler(_context);
    }

    [Fact]
    public async Task Handle_ValidCommand_AddsEntityAndReturnsId()
    {
        // Arrange
        var command = new Create{Entity}Command("Test Name");
        _context.SaveChangesAsync(Arg.Any<CancellationToken>()).Returns(1);

        // Act
        var result = await _handler.Handle(command, CancellationToken.None);

        // Assert
        _{entity}Set.Received(1).Add(Arg.Is<{Entity}>(e => e.Name == "Test Name"));
        await _context.Received(1).SaveChangesAsync(Arg.Any<CancellationToken>());
    }
}
```

### `tests/{ProjectName}.Application.Tests/Features/{Feature}/Queries/Get{Entity}ByIdQueryHandlerTests.cs`

Untuk query handler yang membaca data, gunakan helper `DbSet` mock dengan data in-memory:

```csharp
using NSubstitute;
using FluentAssertions;
using {ProjectName}.Application.Common.Exceptions;
using {ProjectName}.Application.Common.Interfaces;
using {ProjectName}.Application.Features.{Feature}.Queries.Get{Entity}ById;
using {ProjectName}.Domain.Entities;
using Microsoft.EntityFrameworkCore;

namespace {ProjectName}.Application.Tests.Features.{Feature}.Queries;

public class Get{Entity}ByIdQueryHandlerTests
{
    private readonly IApplicationDbContext _context;
    private readonly Get{Entity}ByIdQueryHandler _handler;

    public Get{Entity}ByIdQueryHandlerTests()
    {
        _context = Substitute.For<IApplicationDbContext>();
        _handler = new Get{Entity}ByIdQueryHandler(_context);
    }

    [Fact]
    public async Task Handle_ExistingId_Returns{Entity}Response()
    {
        // Arrange — buat DbSet mock dari list
        var {entity}List = new List<{Entity}>
        {
            {Entity}.Create("Test Name") // gunakan factory method
        }.AsQueryable();

        var mock{Feature} = {entity}List.BuildMockDbSet(); // helper extension (lihat di bawah)
        _context.{Feature}.Returns(mock{Feature});

        // Act — entity baru punya Id=0 (belum di-persist), jadi query dengan Id=0
        var result = await _handler.Handle(new Get{Entity}ByIdQuery(0), CancellationToken.None);

        // Assert
        result.Should().NotBeNull();
        result.Name.Should().Be("Test Name");
    }

    [Fact]
    public async Task Handle_NonExistingId_ThrowsNotFoundException()
    {
        // Arrange
        var emptyList = new List<{Entity}>().AsQueryable().BuildMockDbSet();
        _context.{Feature}.Returns(emptyList);

        // Act & Assert
        await _handler.Invoking(h => h.Handle(new Get{Entity}ByIdQuery(999), CancellationToken.None))
            .Should().ThrowAsync<NotFoundException>();
    }
}
```

> Untuk `BuildMockDbSet()`, tambahkan package `MockQueryable.NSubstitute` ke test project
> atau buat helper yang meng-wrap `IQueryable` ke `DbSet` mock yang mendukung async LINQ.
>
> **Alternatif lebih sederhana:** untuk integration test dengan real database,
> gunakan `Testcontainers` — lihat `packages.md` bagian Integration Tests.
