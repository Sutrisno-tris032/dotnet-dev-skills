# Template Scaffold: Project Lengkap

Gunakan template ini saat menjalankan `new-project`. Ganti:
- `{ProjectName}` → nama proyek yang diberikan user
- `{TFM}` → target framework moniker dari pilihan user: `net6.0` / `net7.0` / `net8.0` / `net9.0`
- `{VER_*}` → versi package dari tabel **Compatibility Matrix** di `references/packages.md`

**Jangan tulis `net9.0` atau versi package secara hardcode** — selalu substitusi dari pilihan user.

---

## Struktur Folder

```
{ProjectName}/
├── src/
│   ├── {ProjectName}.Domain/
│   │   ├── Common/
│   │   │   ├── BaseEntity.cs
│   │   │   ├── AuditableEntity.cs
│   │   │   └── BaseEvent.cs
│   │   └── {ProjectName}.Domain.csproj
│   │
│   ├── {ProjectName}.Application/
│   │   ├── Common/
│   │   │   ├── Behaviors/
│   │   │   │   ├── LoggingBehavior.cs
│   │   │   │   └── ValidationBehavior.cs
│   │   │   ├── Exceptions/
│   │   │   │   ├── ValidationException.cs
│   │   │   │   └── NotFoundException.cs
│   │   │   ├── Interfaces/
│   │   │   │   ├── IApplicationDbContext.cs
│   │   │   │   └── ICurrentUserService.cs
│   │   │   └── Models/
│   │   │       └── ApiResponse.cs
│   │   ├── Features/          ← fitur ditambahkan di sini
│   │   ├── GlobalUsings.cs
│   │   ├── DependencyInjection.cs
│   │   └── {ProjectName}.Application.csproj
│   │
│   ├── {ProjectName}.Infrastructure/
│   │   ├── Persistence/
│   │   │   ├── ApplicationDbContext.cs
│   │   │   ├── Configurations/    ← IEntityTypeConfiguration per entity
│   │   │   └── Interceptors/
│   │   │       └── AuditableEntityInterceptor.cs
│   │   ├── Services/
│   │   │   └── CurrentUserService.cs
│   │   ├── GlobalUsings.cs
│   │   ├── DependencyInjection.cs
│   │   └── {ProjectName}.Infrastructure.csproj
│   │
│   └── {ProjectName}.WebApi/
│       ├── Endpoints/    ← jika Carter; atau Controllers/ jika MVC
│       ├── Middleware/
│       │   └── ExceptionHandlingMiddleware.cs
│       ├── GlobalUsings.cs
│       ├── appsettings.json
│       ├── appsettings.Development.json
│       ├── Program.cs
│       └── {ProjectName}.WebApi.csproj
│
├── tests/
│   ├── {ProjectName}.Application.Tests/
│   └── {ProjectName}.Integration.Tests/
│
├── .gitignore
└── {ProjectName}.sln
```

---

## File Contents

### `Directory.Build.props` (di root solution)

Letakkan di folder `{ProjectName}/` (sejajar dengan `.sln`). Properti ini berlaku untuk semua project di bawahnya sehingga tidak perlu di-repeat di tiap `.csproj`.

```xml
<Project>
  <PropertyGroup>
    <Nullable>enable</Nullable>
    <ImplicitUsings>enable</ImplicitUsings>
    <TreatWarningsAsErrors>true</TreatWarningsAsErrors>
    <LangVersion>default</LangVersion>
  </PropertyGroup>
</Project>
```

### `{ProjectName}.Domain.csproj`

Domain hanya boleh referensi `MediatR.Contracts` (untuk `INotification` di `BaseEvent`). Tidak boleh ada EF Core atau package lain.

```xml
<Project Sdk="Microsoft.NET.Sdk">
  <PropertyGroup>
    <TargetFramework>{TFM}</TargetFramework>
    <Nullable>enable</Nullable>
    <ImplicitUsings>enable</ImplicitUsings>
    <TreatWarningsAsErrors>true</TreatWarningsAsErrors>
  </PropertyGroup>

  <ItemGroup>
    <!-- Hanya interfaces (INotification). MediatR runtime ada di Application, bukan di sini. -->
    <PackageReference Include="MediatR.Contracts" Version="2.0.1" />
  </ItemGroup>
</Project>
```

### `Domain/Common/BaseEntity.cs`

> Kode di bawah menggunakan collection expression `[]` (C# 12 / .NET 8+).
> Untuk .NET 6/7, ganti `= []` dengan `= new()`.

```csharp
namespace {ProjectName}.Domain.Common;

public abstract class BaseEntity
{
    public int Id { get; protected set; }

    // .NET 8+: private readonly List<BaseEvent> _domainEvents = [];
    // .NET 6/7: private readonly List<BaseEvent> _domainEvents = new();
    private readonly List<BaseEvent> _domainEvents = [];
    public IReadOnlyCollection<BaseEvent> DomainEvents => _domainEvents.AsReadOnly();

    public void AddDomainEvent(BaseEvent domainEvent) => _domainEvents.Add(domainEvent);
    public void ClearDomainEvents() => _domainEvents.Clear();
}
```

### `Domain/Common/AuditableEntity.cs`
```csharp
namespace {ProjectName}.Domain.Common;

public abstract class AuditableEntity : BaseEntity
{
    public DateTimeOffset CreatedAt { get; set; }
    public string? CreatedBy { get; set; }
    public DateTimeOffset? UpdatedAt { get; set; }
    public string? UpdatedBy { get; set; }
}
```

### `Domain/Common/BaseEvent.cs`
```csharp
using MediatR;

namespace {ProjectName}.Domain.Common;

public abstract class BaseEvent : INotification { }
```

### `Domain/Exceptions/DomainException.cs`

Lempar exception ini dari entity/domain service ketika business rule dilanggar. Middleware menangkapnya dan mengembalikan HTTP 422.

```csharp
namespace {ProjectName}.Domain.Exceptions;

public class DomainException : Exception
{
    public DomainException(string message) : base(message) { }
}
```

### `{ProjectName}.Application.csproj`
```xml
<Project Sdk="Microsoft.NET.Sdk">
  <PropertyGroup>
    <TargetFramework>{TFM}</TargetFramework>
    <Nullable>enable</Nullable>
    <ImplicitUsings>enable</ImplicitUsings>
    <TreatWarningsAsErrors>true</TreatWarningsAsErrors>
  </PropertyGroup>

  <ItemGroup>
    <PackageReference Include="MediatR" Version="{VER_MediatR}" />
    <PackageReference Include="FluentValidation" Version="{VER_FluentValidation}" />
    <PackageReference Include="FluentValidation.DependencyInjectionExtensions" Version="{VER_FluentValidation}" />
    <PackageReference Include="Microsoft.EntityFrameworkCore" Version="{VER_EFCore}" />
  </ItemGroup>

  <ItemGroup>
    <ProjectReference Include="..\{ProjectName}.Domain\{ProjectName}.Domain.csproj" />
  </ItemGroup>
</Project>
```

### `Application/GlobalUsings.cs`
```csharp
global using MediatR;
global using FluentValidation;
global using FluentValidation.Results;
global using Microsoft.EntityFrameworkCore;
global using {ProjectName}.Domain.Common;
global using {ProjectName}.Domain.Entities;
global using {ProjectName}.Application.Common.Interfaces;
global using {ProjectName}.Application.Common.Exceptions;
global using {ProjectName}.Application.Common.Models;
```

### `Application/Common/Interfaces/IApplicationDbContext.cs`
```csharp
namespace {ProjectName}.Application.Common.Interfaces;

public interface IApplicationDbContext
{
    Task<int> SaveChangesAsync(CancellationToken cancellationToken);
}
```

### `Application/Common/Interfaces/ICurrentUserService.cs`
```csharp
namespace {ProjectName}.Application.Common.Interfaces;

public interface ICurrentUserService
{
    string? UserId { get; }
}
```

### `Application/Common/Models/ApiResponse.cs`

Response envelope standar untuk semua endpoint. `Data` membawa payload sukses; `Errors` membawa detail kegagalan.

```csharp
namespace {ProjectName}.Application.Common.Models;

public record ApiResponse<T>
{
    public bool Success { get; init; }
    public T? Data { get; init; }
    public string? Message { get; init; }
    public object? Errors { get; init; }

    public static ApiResponse<T> Ok(T data, string? message = null)
        => new() { Success = true, Data = data, Message = message };

    public static ApiResponse<T> Fail(string message, object? errors = null)
        => new() { Success = false, Message = message, Errors = errors };
}
```

### `Application/Common/Exceptions/NotFoundException.cs`

> Primary constructor digunakan di bawah (C# 12 / .NET 8+).
> Untuk .NET 6/7, gunakan: `public NotFoundException(string name, object key) : base($"Entity '{name}' ({key}) was not found.") { }`

```csharp
namespace {ProjectName}.Application.Common.Exceptions;

public class NotFoundException(string name, object key)
    : Exception($"Entity '{name}' ({key}) was not found.");
```

### `Application/Common/Exceptions/ValidationException.cs`
```csharp
using FluentValidation.Results;

namespace {ProjectName}.Application.Common.Exceptions;

public class ValidationException : Exception
{
    public IDictionary<string, string[]> Errors { get; }

    public ValidationException(IEnumerable<ValidationFailure> failures)
        : base("One or more validation failures have occurred.")
    {
        Errors = failures
            .GroupBy(f => f.PropertyName, f => f.ErrorMessage)
            .ToDictionary(g => g.Key, g => g.ToArray());
    }
}
```

### `Application/Common/Behaviors/ValidationBehavior.cs`

> Primary constructor digunakan di bawah (C# 12 / .NET 8+).
> Untuk .NET 6/7, gunakan constructor biasa:
> `private readonly IEnumerable<IValidator<TRequest>> _validators;`
> `public ValidationBehavior(IEnumerable<IValidator<TRequest>> validators) => _validators = validators;`
> lalu ganti `validators` dengan `_validators` di dalam method `Handle`.

```csharp
namespace {ProjectName}.Application.Common.Behaviors;

public class ValidationBehavior<TRequest, TResponse>(IEnumerable<IValidator<TRequest>> validators)
    : IPipelineBehavior<TRequest, TResponse>
    where TRequest : notnull
{
    public async Task<TResponse> Handle(TRequest request, RequestHandlerDelegate<TResponse> next, CancellationToken ct)
    {
        if (!validators.Any())
            return await next();

        var context = new ValidationContext<TRequest>(request);
        var errors = new List<ValidationFailure>();

        // Jalankan validator secara sekuensial — DbContext tidak thread-safe untuk concurrent access
        foreach (var validator in validators)
        {
            var result = await validator.ValidateAsync(context, ct);
            errors.AddRange(result.Errors);
        }

        if (errors.Count != 0)
            throw new ValidationException(errors);

        return await next();
    }
}
```

### `Application/Common/Behaviors/LoggingBehavior.cs`

> Primary constructor digunakan di bawah (C# 12 / .NET 8+).
> Untuk .NET 6/7, gunakan constructor biasa:
> `private readonly ILogger<...> _logger; public LoggingBehavior(ILogger<...> logger) => _logger = logger;`
> lalu ganti `logger` dengan `_logger` di dalam `Handle`.

```csharp
using Microsoft.Extensions.Logging;

namespace {ProjectName}.Application.Common.Behaviors;

public class LoggingBehavior<TRequest, TResponse>(ILogger<LoggingBehavior<TRequest, TResponse>> logger)
    : IPipelineBehavior<TRequest, TResponse>
    where TRequest : notnull
{
    public async Task<TResponse> Handle(TRequest request, RequestHandlerDelegate<TResponse> next, CancellationToken ct)
    {
        var requestName = typeof(TRequest).Name;
        logger.LogInformation("Handling {RequestName}", requestName);

        var response = await next();

        logger.LogInformation("Handled {RequestName}", requestName);
        return response;
    }
}
```

### `Application/DependencyInjection.cs`
```csharp
using System.Reflection;
using Microsoft.Extensions.DependencyInjection;
using {ProjectName}.Application.Common.Behaviors;

namespace {ProjectName}.Application;

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

### `{ProjectName}.Infrastructure.csproj`
```xml
<Project Sdk="Microsoft.NET.Sdk">
  <PropertyGroup>
    <TargetFramework>{TFM}</TargetFramework>
    <Nullable>enable</Nullable>
    <ImplicitUsings>enable</ImplicitUsings>
    <TreatWarningsAsErrors>true</TreatWarningsAsErrors>
  </PropertyGroup>

  <ItemGroup>
    <PackageReference Include="Microsoft.EntityFrameworkCore" Version="{VER_EFCore}" />

    <!--
      Sertakan HANYA satu provider sesuai pilihan user (step 1c):
        SQL Server  → Microsoft.EntityFrameworkCore.SqlServer  (versi {VER_EFCore})
        PostgreSQL  → Npgsql.EntityFrameworkCore.PostgreSQL    (versi dari tabel, kolom Npgsql)
        SQLite      → Microsoft.EntityFrameworkCore.Sqlite     (versi {VER_EFCore})
    -->
    <!-- SQL Server (hapus dua baris di bawah jika bukan SQL Server): -->
    <PackageReference Include="Microsoft.EntityFrameworkCore.SqlServer" Version="{VER_EFCore}" />
    <!-- PostgreSQL (hapus komentar jika PostgreSQL; versi dari tabel kolom "Npgsql" di packages.md):
    <PackageReference Include="Npgsql.EntityFrameworkCore.PostgreSQL" Version="{VER_Npgsql}" /> -->
    <!-- SQLite (hapus komentar jika SQLite):
    <PackageReference Include="Microsoft.EntityFrameworkCore.Sqlite" Version="{VER_EFCore}" /> -->

    <PackageReference Include="Microsoft.EntityFrameworkCore.Tools" Version="{VER_EFCore}">
      <PrivateAssets>all</PrivateAssets>
      <IncludeAssets>runtime; build; native; contentfiles; analyzers</IncludeAssets>
    </PackageReference>
    <PackageReference Include="Microsoft.EntityFrameworkCore.Design" Version="{VER_EFCore}">
      <PrivateAssets>all</PrivateAssets>
      <IncludeAssets>runtime; build; native; contentfiles; analyzers</IncludeAssets>
    </PackageReference>
    <PackageReference Include="Microsoft.AspNetCore.Http.Abstractions" Version="2.2.0" />
  </ItemGroup>

  <ItemGroup>
    <ProjectReference Include="..\{ProjectName}.Application\{ProjectName}.Application.csproj" />
  </ItemGroup>
</Project>
```

### `Infrastructure/Persistence/ApplicationDbContext.cs`

> Primary constructor digunakan di bawah (C# 12 / .NET 8+).
> Untuk .NET 6/7 tambahkan constructor eksplisit: `public ApplicationDbContext(DbContextOptions<ApplicationDbContext> options) : base(options) { }`

```csharp
using System.Reflection;
using Microsoft.EntityFrameworkCore;
using {ProjectName}.Application.Common.Interfaces;

namespace {ProjectName}.Infrastructure.Persistence;

public class ApplicationDbContext(DbContextOptions<ApplicationDbContext> options)
    : DbContext(options), IApplicationDbContext
{
    // DbSet<{Entity}> akan ditambahkan saat new-feature

    protected override void OnModelCreating(ModelBuilder builder)
    {
        builder.ApplyConfigurationsFromAssembly(Assembly.GetExecutingAssembly());
        base.OnModelCreating(builder);
    }
}
```

### `Infrastructure/Persistence/Interceptors/AuditableEntityInterceptor.cs`
```csharp
using Microsoft.EntityFrameworkCore;
using Microsoft.EntityFrameworkCore.Diagnostics;
using {ProjectName}.Application.Common.Interfaces;
using {ProjectName}.Domain.Common;

namespace {ProjectName}.Infrastructure.Persistence.Interceptors;

public class AuditableEntityInterceptor(ICurrentUserService currentUserService) : SaveChangesInterceptor
{
    public override InterceptionResult<int> SavingChanges(DbContextEventData eventData, InterceptionResult<int> result)
    {
        UpdateAuditFields(eventData.Context);
        return base.SavingChanges(eventData, result);
    }

    public override ValueTask<InterceptionResult<int>> SavingChangesAsync(
        DbContextEventData eventData, InterceptionResult<int> result, CancellationToken ct = default)
    {
        UpdateAuditFields(eventData.Context);
        return base.SavingChangesAsync(eventData, result, ct);
    }

    private void UpdateAuditFields(DbContext? context)
    {
        if (context is null) return;

        foreach (var entry in context.ChangeTracker.Entries<AuditableEntity>())
        {
            if (entry.State == EntityState.Added)
            {
                entry.Entity.CreatedBy = currentUserService.UserId;
                entry.Entity.CreatedAt = DateTimeOffset.UtcNow;
            }

            if (entry.State is EntityState.Added or EntityState.Modified)
            {
                entry.Entity.UpdatedBy = currentUserService.UserId;
                entry.Entity.UpdatedAt = DateTimeOffset.UtcNow;
            }
        }
    }
}
```

### `Infrastructure/Services/CurrentUserService.cs`
```csharp
using System.Security.Claims;
using Microsoft.AspNetCore.Http;
using {ProjectName}.Application.Common.Interfaces;

namespace {ProjectName}.Infrastructure.Services;

public class CurrentUserService(IHttpContextAccessor httpContextAccessor) : ICurrentUserService
{
    public string? UserId => httpContextAccessor.HttpContext?.User.FindFirstValue(ClaimTypes.NameIdentifier);
}
```

### `Infrastructure/DependencyInjection.cs`
```csharp
using Microsoft.EntityFrameworkCore;
using Microsoft.Extensions.Configuration;
using Microsoft.Extensions.DependencyInjection;
using {ProjectName}.Application.Common.Interfaces;
using {ProjectName}.Infrastructure.Persistence;
using {ProjectName}.Infrastructure.Persistence.Interceptors;
using {ProjectName}.Infrastructure.Services;

namespace {ProjectName}.Infrastructure;

public static class DependencyInjection
{
    public static IServiceCollection AddInfrastructureServices(
        this IServiceCollection services,
        IConfiguration configuration)
    {
        services.AddScoped<AuditableEntityInterceptor>();

        services.AddDbContext<ApplicationDbContext>((sp, options) =>
        {
            var connStr = configuration.GetConnectionString("DefaultConnection")!;
            // Aktifkan SATU baris sesuai provider yang dipilih user (step 1c):
            options.UseSqlServer(connStr);    // SQL Server
            // options.UseNpgsql(connStr);    // PostgreSQL
            // options.UseSqlite(connStr);    // SQLite

            options.AddInterceptors(sp.GetRequiredService<AuditableEntityInterceptor>());
        });

        services.AddScoped<IApplicationDbContext>(sp => sp.GetRequiredService<ApplicationDbContext>());

        services.AddHttpContextAccessor();
        services.AddScoped<ICurrentUserService, CurrentUserService>();

        return services;
    }
}
```

### `{ProjectName}.WebApi.csproj`
```xml
<Project Sdk="Microsoft.NET.Sdk.Web">
  <PropertyGroup>
    <TargetFramework>{TFM}</TargetFramework>
    <Nullable>enable</Nullable>
    <ImplicitUsings>enable</ImplicitUsings>
    <TreatWarningsAsErrors>true</TreatWarningsAsErrors>
  </PropertyGroup>

  <ItemGroup>
    <!-- Swagger — gunakan untuk SEMUA versi .NET. AddOpenApi/MapOpenApi adalah .NET 9+ ONLY. -->
    <PackageReference Include="Swashbuckle.AspNetCore" Version="{VER_Swashbuckle}" />

    <!-- WAJIB: diperlukan agar `dotnet ef migrations add --startup-project WebApi` bisa berjalan -->
    <PackageReference Include="Microsoft.EntityFrameworkCore.Design" Version="{VER_EFCore}">
      <PrivateAssets>all</PrivateAssets>
      <IncludeAssets>runtime; build; native; contentfiles; analyzers</IncludeAssets>
    </PackageReference>

    <!-- Tambahkan Carter jika user pilih Minimal API -->
    <!-- <PackageReference Include="Carter" Version="{VER_Carter}" /> -->
  </ItemGroup>

  <ItemGroup>
    <ProjectReference Include="..\{ProjectName}.Application\{ProjectName}.Application.csproj" />
    <ProjectReference Include="..\{ProjectName}.Infrastructure\{ProjectName}.Infrastructure.csproj" />
  </ItemGroup>
</Project>
```

### `WebApi/Program.cs`

> **Penting:** `AddOpenApi()` dan `MapOpenApi()` adalah fitur bawaan **ASP.NET Core 9+ saja**.
> Untuk .NET 6/7/8, **WAJIB** gunakan Swashbuckle di bawah. Jangan gunakan `AddOpenApi` untuk .NET < 9.

**Template A — Controllers (untuk .NET 6 / 7 / 8 / 9 dengan Swashbuckle):**
```csharp
using Microsoft.AspNetCore.Mvc;
using {ProjectName}.Application;
using {ProjectName}.Application.Common.Models;
using {ProjectName}.Infrastructure;
using {ProjectName}.WebApi.Middleware;

var builder = WebApplication.CreateBuilder(args);

builder.Services.AddApplicationServices();
builder.Services.AddInfrastructureServices(builder.Configuration);

builder.Services.AddControllers()
    .ConfigureApiBehaviorOptions(options =>
    {
        // Normalisasi model-binding errors ke ApiResponse agar konsisten dengan middleware
        options.InvalidModelStateResponseFactory = ctx =>
        {
            var errors = ctx.ModelState
                .Where(e => e.Value?.Errors.Count > 0)
                .ToDictionary(
                    e => e.Key,
                    e => e.Value!.Errors.Select(x => x.ErrorMessage).ToArray());

            return new BadRequestObjectResult(
                ApiResponse<object?>.Fail("Validation failed.", errors));
        };
    });

builder.Services.AddEndpointsApiExplorer();
builder.Services.AddSwaggerGen();

var app = builder.Build();

app.UseMiddleware<ExceptionHandlingMiddleware>();

if (app.Environment.IsDevelopment())
{
    app.UseSwagger();
    app.UseSwaggerUI();
}

app.UseHttpsRedirection();
app.UseAuthorization();
app.MapControllers();

app.Run();
```

**Template B — Carter Minimal API (untuk .NET 6 / 7 / 8 / 9 dengan Swashbuckle):**

Aktifkan package Carter di `WebApi.csproj` (uncomment baris Carter), lalu gunakan Program.cs ini:

```csharp
using Carter;
using {ProjectName}.Application;
using {ProjectName}.Infrastructure;
using {ProjectName}.WebApi.Middleware;

var builder = WebApplication.CreateBuilder(args);

builder.Services.AddApplicationServices();
builder.Services.AddInfrastructureServices(builder.Configuration);

builder.Services.AddCarter();
builder.Services.AddEndpointsApiExplorer();
builder.Services.AddSwaggerGen();

var app = builder.Build();

app.UseMiddleware<ExceptionHandlingMiddleware>();

if (app.Environment.IsDevelopment())
{
    app.UseSwagger();
    app.UseSwaggerUI();
}

app.UseHttpsRedirection();
app.UseAuthorization();
app.MapCarter();

app.Run();
```

> Jika user memilih .NET 9 dan ingin built-in OpenAPI, ganti blok Swagger (`AddEndpointsApiExplorer` + `AddSwaggerGen` + `UseSwagger` + `UseSwaggerUI`) dengan:
> `builder.Services.AddOpenApi();` dan `app.MapOpenApi();` — tapi pastikan hapus Swashbuckle dari .csproj.

### `WebApi/Middleware/ExceptionHandlingMiddleware.cs`
```csharp
using System.Net;
using System.Text.Json;
using {ProjectName}.Application.Common.Exceptions;
using {ProjectName}.Application.Common.Models;
using {ProjectName}.Domain.Exceptions;

namespace {ProjectName}.WebApi.Middleware;

public class ExceptionHandlingMiddleware(RequestDelegate next, ILogger<ExceptionHandlingMiddleware> logger)
{
    // Static readonly agar tidak alokasi JsonSerializerOptions baru tiap request
    private static readonly JsonSerializerOptions _jsonOptions =
        new() { PropertyNamingPolicy = JsonNamingPolicy.CamelCase };

    public async Task InvokeAsync(HttpContext context)
    {
        try
        {
            await next(context);
        }
        catch (Exception ex)
        {
            logger.LogError(ex, "An unhandled exception occurred");
            await HandleExceptionAsync(context, ex);
        }
    }

    private static async Task HandleExceptionAsync(HttpContext context, Exception exception)
    {
        context.Response.ContentType = "application/json";

        var (statusCode, message, errors) = exception switch
        {
            ValidationException ve  => (HttpStatusCode.BadRequest, "Validation failed.", (object?)ve.Errors),
            NotFoundException       => (HttpStatusCode.NotFound, exception.Message, (object?)null),
            DomainException         => (HttpStatusCode.UnprocessableEntity, exception.Message, (object?)null),
            _                       => (HttpStatusCode.InternalServerError, "An unexpected error occurred.", (object?)null)
        };

        context.Response.StatusCode = (int)statusCode;

        var response = ApiResponse<object?>.Fail(message, errors);
        await context.Response.WriteAsync(JsonSerializer.Serialize(response, _jsonOptions));
    }
}
```

### `WebApi/appsettings.json`

Gunakan template sesuai database provider yang dipilih user:

**SQL Server (default):**
```json
{
  "ConnectionStrings": {
    "DefaultConnection": "Server=(localdb)\\mssqllocaldb;Database={ProjectName}Db;Trusted_Connection=True;MultipleActiveResultSets=true"
  },
  "Logging": {
    "LogLevel": {
      "Default": "Information",
      "Microsoft.AspNetCore": "Warning"
    }
  },
  "AllowedHosts": "*"
}
```

**PostgreSQL:**
```json
{
  "ConnectionStrings": {
    "DefaultConnection": "Host=localhost;Port=5432;Database={ProjectName}Db;Username=postgres;Password=your_password;Search Path=public"
  },
  "Logging": {
    "LogLevel": {
      "Default": "Information",
      "Microsoft.AspNetCore": "Warning"
    }
  },
  "AllowedHosts": "*"
}
```

> `Search Path=public` memastikan query menggunakan schema `public` secara default di PostgreSQL.
> Ganti `public` dengan nama schema custom jika dibutuhkan.

**SQLite:**
```json
{
  "ConnectionStrings": {
    "DefaultConnection": "Data Source={ProjectName}.db"
  },
  "Logging": {
    "LogLevel": {
      "Default": "Information",
      "Microsoft.AspNetCore": "Warning"
    }
  },
  "AllowedHosts": "*"
}
```

### `WebApi/appsettings.Development.json`

Gunakan template sesuai database provider yang dipilih user (sama dengan pilihan di step 1c):

**SQL Server (default):**
```json
{
  "ConnectionStrings": {
    "DefaultConnection": "Server=(localdb)\\mssqllocaldb;Database={ProjectName}DevDb;Trusted_Connection=True;MultipleActiveResultSets=true"
  },
  "Logging": {
    "LogLevel": {
      "Default": "Debug",
      "Microsoft.AspNetCore": "Information",
      "Microsoft.EntityFrameworkCore.Database.Command": "Information"
    }
  }
}
```

**PostgreSQL:**
```json
{
  "ConnectionStrings": {
    "DefaultConnection": "Host=localhost;Port=5432;Database={ProjectName}DevDb;Username=postgres;Password=your_password;Search Path=public"
  },
  "Logging": {
    "LogLevel": {
      "Default": "Debug",
      "Microsoft.AspNetCore": "Information",
      "Microsoft.EntityFrameworkCore.Database.Command": "Information"
    }
  }
}
```

**SQLite:**
```json
{
  "ConnectionStrings": {
    "DefaultConnection": "Data Source={ProjectName}Dev.db"
  },
  "Logging": {
    "LogLevel": {
      "Default": "Debug",
      "Microsoft.AspNetCore": "Information",
      "Microsoft.EntityFrameworkCore.Database.Command": "Information"
    }
  }
}
```

### `.gitignore`
```
## Build output
bin/
obj/
publish/
*.nupkg
*.snupkg

## Visual Studio
*.user
*.suo
.vs/
*.DotSettings.user
*.sln.iml

## Test results
TestResults/
coverage/
*.coverage
*.coveragexml

## Migrations (opsional — uncomment jika tidak ingin commit migrations)
# **/Migrations/

## Secrets & env config overrides
appsettings.*.local.json
secrets.json
*.env
.env

## OS
.DS_Store
Thumbs.db
desktop.ini

## JetBrains Rider
.idea/

## VS Code
.vscode/
!.vscode/extensions.json
!.vscode/settings.json

## Docker artifacts
docker-compose.override.yml
```

### Solution file `{ProjectName}.sln`
Jalankan perintah berikut untuk membuat solution:
```bash
dotnet new sln -n {ProjectName}
dotnet sln add src/{ProjectName}.Domain
dotnet sln add src/{ProjectName}.Application
dotnet sln add src/{ProjectName}.Infrastructure
dotnet sln add src/{ProjectName}.WebApi
dotnet sln add tests/{ProjectName}.Application.Tests
dotnet sln add tests/{ProjectName}.Integration.Tests
```

### `tests/{ProjectName}.Application.Tests/{ProjectName}.Application.Tests.csproj`

Sesuaikan versi package dari Compatibility Matrix berdasarkan .NET version yang dipilih user.

```xml
<Project Sdk="Microsoft.NET.Sdk">
  <PropertyGroup>
    <TargetFramework>{TFM}</TargetFramework>
    <IsPackable>false</IsPackable>
  </PropertyGroup>

  <ItemGroup>
    <PackageReference Include="Microsoft.NET.Test.Sdk" Version="{VER_TestSdk}" />
    <PackageReference Include="xunit" Version="{VER_xunit}" />
    <PackageReference Include="xunit.runner.visualstudio" Version="{VER_xunitRunner}" />
    <PackageReference Include="FluentAssertions" Version="{VER_FluentAssertions}" />
    <PackageReference Include="NSubstitute" Version="{VER_NSubstitute}" />
    <!-- MockQueryable diperlukan agar DbSet mock mendukung async LINQ (ToListAsync, FirstOrDefaultAsync) -->
    <PackageReference Include="MockQueryable.NSubstitute" Version="{VER_MockQueryable}" />
  </ItemGroup>

  <ItemGroup>
    <ProjectReference Include="..\..\src\{ProjectName}.Application\{ProjectName}.Application.csproj" />
  </ItemGroup>
</Project>
```

### `tests/{ProjectName}.Integration.Tests/{ProjectName}.Integration.Tests.csproj`

Gunakan Testcontainers sesuai provider yang dipilih user di step 1c:
- SQL Server → `Testcontainers.MsSql`
- PostgreSQL → `Testcontainers.PostgreSql`
- SQLite → tidak perlu Testcontainers (file-based)

```xml
<Project Sdk="Microsoft.NET.Sdk">
  <PropertyGroup>
    <TargetFramework>{TFM}</TargetFramework>
    <IsPackable>false</IsPackable>
  </PropertyGroup>

  <ItemGroup>
    <PackageReference Include="Microsoft.NET.Test.Sdk" Version="{VER_TestSdk}" />
    <PackageReference Include="xunit" Version="{VER_xunit}" />
    <PackageReference Include="xunit.runner.visualstudio" Version="{VER_xunitRunner}" />
    <PackageReference Include="FluentAssertions" Version="{VER_FluentAssertions}" />
    <PackageReference Include="Microsoft.AspNetCore.Mvc.Testing" Version="{VER_MvcTesting}" />
    <!-- Pilih provider Testcontainers sesuai step 1c: -->
    <PackageReference Include="Testcontainers.MsSql" Version="3.9.0" />
    <!-- <PackageReference Include="Testcontainers.PostgreSql" Version="3.9.0" /> -->
    <PackageReference Include="Respawn" Version="{VER_Respawn}" />
  </ItemGroup>

  <ItemGroup>
    <ProjectReference Include="..\..\src\{ProjectName}.WebApi\{ProjectName}.WebApi.csproj" />
  </ItemGroup>
</Project>
```
