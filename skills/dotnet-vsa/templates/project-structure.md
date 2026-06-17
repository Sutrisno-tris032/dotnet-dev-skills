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
│   │   │   └── Interfaces/
│   │   │       ├── IApplicationDbContext.cs
│   │   │       └── ICurrentUserService.cs
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

### `{ProjectName}.Domain.csproj`
```xml
<Project Sdk="Microsoft.NET.Sdk">
  <PropertyGroup>
    <TargetFramework>{TFM}</TargetFramework>
    <Nullable>enable</Nullable>
    <ImplicitUsings>enable</ImplicitUsings>
    <TreatWarningsAsErrors>true</TreatWarningsAsErrors>
  </PropertyGroup>
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

public abstract class BaseEvent : INotification;
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
global using Microsoft.EntityFrameworkCore;
global using {ProjectName}.Domain.Common;
global using {ProjectName}.Application.Common.Interfaces;
global using {ProjectName}.Application.Common.Exceptions;
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

### `Application/Common/Exceptions/NotFoundException.cs`
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
        var errors = validators
            .Select(v => v.Validate(context))
            .Where(r => !r.IsValid)
            .SelectMany(r => r.Errors)
            .ToList();

        if (errors.Count != 0)
            throw new ValidationException(errors);

        return await next();
    }
}
```

### `Application/Common/Behaviors/LoggingBehavior.cs`
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
    <!-- Gunakan provider sesuai pilihan user: SqlServer / Npgsql / Sqlite -->
    <PackageReference Include="Microsoft.EntityFrameworkCore.SqlServer" Version="{VER_EFCore}" />
    <PackageReference Include="Microsoft.EntityFrameworkCore.Tools" Version="{VER_EFCore}">
      <PrivateAssets>all</PrivateAssets>
      <IncludeAssets>runtime; build; native; contentfiles; analyzers</IncludeAssets>
    </PackageReference>
    <PackageReference Include="Microsoft.EntityFrameworkCore.Design" Version="{VER_EFCore}">
      <PrivateAssets>all</PrivateAssets>
      <IncludeAssets>runtime; build; native; contentfiles; analyzers</IncludeAssets>
    </PackageReference>
    <PackageReference Include="Microsoft.AspNetCore.Http.Abstractions" Version="2.2.*" />
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
            options.UseSqlServer(configuration.GetConnectionString("DefaultConnection"));
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
  </PropertyGroup>

  <ItemGroup>
    <PackageReference Include="Swashbuckle.AspNetCore" Version="{VER_Swashbuckle}" />
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
```csharp
using {ProjectName}.Application;
using {ProjectName}.Infrastructure;
using {ProjectName}.WebApi.Middleware;

var builder = WebApplication.CreateBuilder(args);

builder.Services.AddApplicationServices();
builder.Services.AddInfrastructureServices(builder.Configuration);

builder.Services.AddControllers();
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

### `WebApi/Middleware/ExceptionHandlingMiddleware.cs`
```csharp
using System.Net;
using System.Text.Json;
using {ProjectName}.Application.Common.Exceptions;

namespace {ProjectName}.WebApi.Middleware;

public class ExceptionHandlingMiddleware(RequestDelegate next, ILogger<ExceptionHandlingMiddleware> logger)
{
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
        context.Response.ContentType = "application/problem+json";

        var (statusCode, title, errors) = exception switch
        {
            ValidationException ve => (HttpStatusCode.BadRequest, "Validation Error", ve.Errors),
            NotFoundException => (HttpStatusCode.NotFound, "Not Found", (IDictionary<string, string[]>?)null),
            _ => (HttpStatusCode.InternalServerError, "Server Error", null)
        };

        context.Response.StatusCode = (int)statusCode;

        var response = new
        {
            type = "https://tools.ietf.org/html/rfc7807",
            title,
            status = (int)statusCode,
            detail = exception.Message,
            errors
        };

        await context.Response.WriteAsync(JsonSerializer.Serialize(response,
            new JsonSerializerOptions { PropertyNamingPolicy = JsonNamingPolicy.CamelCase }));
    }
}
```

### `WebApi/appsettings.json`
```json
{
  "ConnectionStrings": {
    "DefaultConnection": "Server=(localdb)\\mssqllocaldb;Database={ProjectName}Db;Trusted_Connection=True;"
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

### `.gitignore`
```
## .NET
bin/
obj/
*.user
*.suo
.vs/
*.DotSettings.user

## Migrations (opsional — uncomment jika tidak ingin commit migrations)
# **/Migrations/

## Environment
appsettings.*.local.json
*.env
.env

## OS
.DS_Store
Thumbs.db
```

### Solution file `{ProjectName}.sln`
Jalankan perintah berikut untuk membuat solution:
```bash
dotnet new sln -n {ProjectName}
dotnet sln add src/{ProjectName}.Domain
dotnet sln add src/{ProjectName}.Application
dotnet sln add src/{ProjectName}.Infrastructure
dotnet sln add src/{ProjectName}.WebApi
```
