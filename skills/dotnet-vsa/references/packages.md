# NuGet Packages

## Compatibility Matrix

**WAJIB dibaca saat `new-project`**. Setelah user memilih versi .NET, gunakan versi package dari baris yang sesuai. Jangan hardcode `*` wildcard di output akhir — tulis versi eksak mayor.minor.

| Package | .NET 6 (C# 10) | .NET 7 (C# 11) | .NET 8 (C# 12) | .NET 9 (C# 13) |
|---|---|---|---|---|
| `TargetFramework` | `net6.0` | `net7.0` | `net8.0` | `net9.0` |
| **MediatR** | `12.2.*` | `12.2.*` | `12.4.*` | `12.4.*` |
| **FluentValidation** | `11.9.*` | `11.9.*` | `11.11.*` | `11.11.*` |
| **FluentValidation.DependencyInjectionExtensions** | `11.9.*` | `11.9.*` | `11.11.*` | `11.11.*` |
| **Microsoft.EntityFrameworkCore** | `6.0.*` | `7.0.*` | `8.0.*` | `9.0.*` |
| **Microsoft.EntityFrameworkCore.SqlServer** | `6.0.*` | `7.0.*` | `8.0.*` | `9.0.*` |
| **Npgsql.EntityFrameworkCore.PostgreSQL** | `6.0.*` | `7.0.*` | `8.0.*` | `9.0.*` |
| **Microsoft.EntityFrameworkCore.Sqlite** | `6.0.*` | `7.0.*` | `8.0.*` | `9.0.*` |
| **Microsoft.EntityFrameworkCore.Design** | `6.0.*` | `7.0.*` | `8.0.*` | `9.0.*` |
| **Microsoft.EntityFrameworkCore.Tools** | `6.0.*` | `7.0.*` | `8.0.*` | `9.0.*` |
| **Swashbuckle.AspNetCore** | `6.5.*` | `6.5.*` | `6.9.*` | `7.2.*` |
| **Carter** | `7.2.*` | `7.2.*` | `8.0.*` | `8.1.*` |
| **Mapster** | `7.3.*` | `7.3.*` | `7.4.*` | `7.4.*` |
| **Mapster.DependencyInjection** | `1.0.*` | `1.0.*` | `1.0.*` | `1.0.*` |
| **Serilog.AspNetCore** | `6.1.*` | `7.0.*` | `8.0.*` | `9.0.*` |
| **Serilog.Sinks.Console** | `4.1.*` | `4.1.*` | `5.0.*` | `6.0.*` |
| **Serilog.Sinks.File** | `5.0.*` | `5.0.*` | `5.0.*` | `6.0.*` |
| **Microsoft.AspNetCore.Http.Abstractions** | `2.2.*` | `2.2.*` | `2.2.*` | `2.2.*` |
| **Microsoft.NET.Test.Sdk** | `17.8.*` | `17.8.*` | `17.11.*` | `17.12.*` |
| **xunit** | `2.6.*` | `2.6.*` | `2.9.*` | `2.9.*` |
| **xunit.runner.visualstudio** | `2.5.*` | `2.5.*` | `2.8.*` | `2.8.*` |
| **FluentAssertions** | `6.12.*` | `6.12.*` | `6.12.*` | `6.12.*` |
| **NSubstitute** | `5.1.*` | `5.1.*` | `5.1.*` | `5.3.*` |
| **Respawn** | `6.0.*` | `6.0.*` | `6.2.*` | `6.2.*` |
| **Microsoft.AspNetCore.Mvc.Testing** | `6.0.*` | `7.0.*` | `8.0.*` | `9.0.*` |

### Catatan Penting per Versi

**NET 6:**
- Gunakan sintaks C# 10: tidak ada `primary constructor`, tidak ada collection expression `[]`
- `new List<T>()` bukan `[]`; constructor via `public MyClass(IFoo foo) { _foo = foo; }` bukan `(IFoo foo)`
- `global using` sudah tersedia (C# 10)
- `record` tersedia; `record struct` tersedia
- Minimal API tersedia tapi Carter belum sematur versi 8+

**NET 7:**
- C# 11: `required` modifier, raw string literal, generic attributes
- Constructor masih belum primary constructor (itu C# 12)
- Pattern `INumber<T>` generic math tersedia

**NET 8:**
- C# 12: **primary constructor** tersedia untuk semua class (bukan hanya record)
- Collection expression `[]` tersedia
- `FrozenDictionary`, `FrozenSet` tersedia
- Keyed DI (`AddKeyedScoped`) tersedia

**NET 9:**
- C# 13: `params` span, `field` keyword (preview)
- `HybridCache` tersedia
- OpenAPI built-in (`Microsoft.AspNetCore.OpenApi`) — Swashbuckle masih bisa dipakai tapi bersaing dengan built-in

---

## Sintaks C# per Versi — Quick Reference

Saat generate kode, sesuaikan sintaks dengan .NET version yang dipilih:

```csharp
// Primary constructor — hanya .NET 8+ (C# 12)
public class CreateOrderHandler(IApplicationDbContext context) : IRequestHandler<...> { }

// .NET 6/7 — gunakan constructor biasa
public class CreateOrderHandler : IRequestHandler<...>
{
    private readonly IApplicationDbContext _context;
    public CreateOrderHandler(IApplicationDbContext context) => _context = context;
}

// Collection expression [] — hanya .NET 8+ (C# 12)
private readonly List<BaseEvent> _domainEvents = [];

// .NET 6/7
private readonly List<BaseEvent> _domainEvents = new();

// FindAsync dengan array literal — .NET 8+
await context.Orders.FindAsync([request.Id], ct);

// .NET 6/7
await context.Orders.FindAsync(new object[] { request.Id }, ct);
```

---

## Package Per Layer

### Domain (`{ProjectName}.Domain`)

Tidak ada NuGet package eksternal. Domain harus pure C#.

### Application (`{ProjectName}.Application`)

Gunakan versi dari Compatibility Matrix sesuai .NET target.

```xml
<ItemGroup>
  <PackageReference Include="MediatR" Version="{versi dari tabel}" />
  <PackageReference Include="FluentValidation" Version="{versi dari tabel}" />
  <PackageReference Include="FluentValidation.DependencyInjectionExtensions" Version="{versi dari tabel}" />
  <PackageReference Include="Microsoft.EntityFrameworkCore" Version="{versi dari tabel}" />

  <!-- Opsional: mapping -->
  <PackageReference Include="Mapster" Version="{versi dari tabel}" />
  <PackageReference Include="Mapster.DependencyInjection" Version="{versi dari tabel}" />
</ItemGroup>

<ItemGroup>
  <ProjectReference Include="..\{ProjectName}.Domain\{ProjectName}.Domain.csproj" />
</ItemGroup>
```

### Infrastructure (`{ProjectName}.Infrastructure`)

```xml
<ItemGroup>
  <PackageReference Include="Microsoft.EntityFrameworkCore" Version="{versi dari tabel}" />
  <PackageReference Include="Microsoft.EntityFrameworkCore.Tools" Version="{versi dari tabel}">
    <PrivateAssets>all</PrivateAssets>
    <IncludeAssets>runtime; build; native; contentfiles; analyzers</IncludeAssets>
  </PackageReference>
  <PackageReference Include="Microsoft.EntityFrameworkCore.Design" Version="{versi dari tabel}">
    <PrivateAssets>all</PrivateAssets>
    <IncludeAssets>runtime; build; native; contentfiles; analyzers</IncludeAssets>
  </PackageReference>

  <!-- Database Provider — pilih sesuai pilihan user -->
  <!-- SQL Server -->
  <PackageReference Include="Microsoft.EntityFrameworkCore.SqlServer" Version="{versi dari tabel}" />
  <!-- PostgreSQL -->
  <PackageReference Include="Npgsql.EntityFrameworkCore.PostgreSQL" Version="{versi dari tabel}" />
  <!-- SQLite -->
  <PackageReference Include="Microsoft.EntityFrameworkCore.Sqlite" Version="{versi dari tabel}" />

  <PackageReference Include="Microsoft.AspNetCore.Http.Abstractions" Version="{versi dari tabel}" />
</ItemGroup>

<ItemGroup>
  <ProjectReference Include="..\{ProjectName}.Application\{ProjectName}.Application.csproj" />
</ItemGroup>
```

### WebApi (`{ProjectName}.WebApi`)

```xml
<ItemGroup>
  <!-- Swagger -->
  <PackageReference Include="Swashbuckle.AspNetCore" Version="{versi dari tabel}" />

  <!-- Opsional: Carter (jika user pilih minimal API) -->
  <PackageReference Include="Carter" Version="{versi dari tabel}" />

  <!-- Opsional: Serilog -->
  <PackageReference Include="Serilog.AspNetCore" Version="{versi dari tabel}" />
  <PackageReference Include="Serilog.Sinks.Console" Version="{versi dari tabel}" />
  <PackageReference Include="Serilog.Sinks.File" Version="{versi dari tabel}" />
</ItemGroup>

<ItemGroup>
  <ProjectReference Include="..\{ProjectName}.Application\{ProjectName}.Application.csproj" />
  <ProjectReference Include="..\{ProjectName}.Infrastructure\{ProjectName}.Infrastructure.csproj" />
</ItemGroup>
```

---

## EF Core Migrations

```bash
# Tambah migration
dotnet ef migrations add {MigrationName} \
  --project src/{ProjectName}.Infrastructure \
  --startup-project src/{ProjectName}.WebApi

# Apply migration
dotnet ef database update \
  --project src/{ProjectName}.Infrastructure \
  --startup-project src/{ProjectName}.WebApi

# Rollback
dotnet ef database update {MigrationName} \
  --project src/{ProjectName}.Infrastructure \
  --startup-project src/{ProjectName}.WebApi
```

---

## Testing Packages (project terpisah)

Gunakan versi dari Compatibility Matrix.

```xml
<!-- {ProjectName}.Application.Tests.csproj -->
<ItemGroup>
  <PackageReference Include="Microsoft.NET.Test.Sdk" Version="{versi dari tabel}" />
  <PackageReference Include="xunit" Version="{versi dari tabel}" />
  <PackageReference Include="xunit.runner.visualstudio" Version="{versi dari tabel}" />
  <PackageReference Include="FluentAssertions" Version="{versi dari tabel}" />
  <PackageReference Include="NSubstitute" Version="{versi dari tabel}" />
  <PackageReference Include="Microsoft.EntityFrameworkCore.InMemory" Version="{versi EF dari tabel}" />
</ItemGroup>

<!-- {ProjectName}.Integration.Tests.csproj -->
<ItemGroup>
  <PackageReference Include="Microsoft.AspNetCore.Mvc.Testing" Version="{versi dari tabel}" />
  <PackageReference Include="Testcontainers.MsSql" Version="3.*" />
  <PackageReference Include="Respawn" Version="{versi dari tabel}" />
</ItemGroup>
```
