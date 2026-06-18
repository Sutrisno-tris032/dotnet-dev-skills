# NuGet Packages

## Compatibility Matrix

**WAJIB dibaca saat `new-project`**. Setelah user memilih versi .NET, gunakan versi package dari baris yang sesuai. **Jangan gunakan `*` wildcard** — tulis versi eksak. Versi di bawah adalah versi stabil terbaru yang sudah divalidasi.

| Package | .NET 6 (C# 10) | .NET 7 (C# 11) | .NET 8 (C# 12) | .NET 9 (C# 13) |
|---|---|---|---|---|
| `TargetFramework` | `net6.0` | `net7.0` | `net8.0` | `net9.0` |
| **MediatR** | `12.2.0` | `12.2.0` | `12.4.1` | `12.4.1` |
| **MediatR.Contracts** | `2.0.1` | `2.0.1` | `2.0.1` | `2.0.1` |
| **FluentValidation** | `11.9.2` | `11.9.2` | `11.11.0` | `11.11.0` |
| **FluentValidation.DependencyInjectionExtensions** | `11.9.2` | `11.9.2` | `11.11.0` | `11.11.0` |
| **Microsoft.EntityFrameworkCore** | `6.0.36` | `7.0.20` | `8.0.11` | `9.0.4` |
| **Microsoft.EntityFrameworkCore.SqlServer** | `6.0.36` | `7.0.20` | `8.0.11` | `9.0.4` |
| **Npgsql.EntityFrameworkCore.PostgreSQL** | `6.0.22` | `7.0.18` | `8.0.11` | `9.0.4` |
| **Microsoft.EntityFrameworkCore.Sqlite** | `6.0.36` | `7.0.20` | `8.0.11` | `9.0.4` |
| **Microsoft.EntityFrameworkCore.Design** | `6.0.36` | `7.0.20` | `8.0.11` | `9.0.4` |
| **Microsoft.EntityFrameworkCore.Tools** | `6.0.36` | `7.0.20` | `8.0.11` | `9.0.4` |
| **Swashbuckle.AspNetCore** | `6.5.0` | `6.5.0` | `6.9.0` | `7.2.0` |
| **Carter** | `7.2.0` | `7.2.0` | `8.2.1` | `9.0.0` |
| **Mapster** | `7.3.0` | `7.3.0` | `7.4.0` | `7.4.0` |
| **Mapster.DependencyInjection** | `1.0.0` | `1.0.0` | `1.0.0` | `1.0.0` |
| **Serilog.AspNetCore** | `6.1.0` | `7.0.0` | `8.0.3` | `9.0.0` |
| **Serilog.Sinks.Console** | `4.1.0` | `4.1.0` | `5.0.0` | `6.0.0` |
| **Serilog.Sinks.File** | `5.0.0` | `5.0.0` | `6.0.0` | `6.0.0` |
| **Microsoft.AspNetCore.Http.Abstractions** | `2.2.0` | `2.2.0` | `2.2.0` | `2.2.0` |
| **Microsoft.NET.Test.Sdk** | `17.8.0` | `17.8.0` | `17.11.1` | `17.12.0` |
| **xunit** | `2.6.6` | `2.6.6` | `2.9.2` | `2.9.2` |
| **xunit.runner.visualstudio** | `2.5.8` | `2.5.8` | `2.8.2` | `2.8.2` |
| **FluentAssertions** | `6.12.2` | `6.12.2` | `6.12.2` | `6.12.2` |
| **NSubstitute** | `5.1.0` | `5.1.0` | `5.1.0` | `5.3.0` |
| **MockQueryable.NSubstitute** | `7.0.0` | `7.0.0` | `7.0.0` | `7.0.1` |
| **Respawn** | `6.0.2` | `6.0.2` | `6.2.1` | `6.2.1` |
| **Microsoft.AspNetCore.Mvc.Testing** | `6.0.36` | `7.0.20` | `8.0.11` | `9.0.4` |

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
- OpenAPI built-in (`AddOpenApi`/`MapOpenApi`) tersedia — **HANYA .NET 9+**, jangan pakai di .NET 6/7/8
- Swashbuckle 7.2.0 tetap bisa dipakai di .NET 9 dan direkomendasikan untuk kompatibilitas

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

Hanya boleh referensi `MediatR.Contracts` (interface untuk domain event: `INotification`).
Tidak boleh ada dependency ke Application, Infrastructure, atau package EF Core.

```xml
<ItemGroup>
  <PackageReference Include="MediatR.Contracts" Version="2.0.1" />
</ItemGroup>
```

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
  <!-- Swagger — gunakan untuk SEMUA versi .NET (6/7/8/9) -->
  <!-- JANGAN gunakan AddOpenApi/MapOpenApi untuk .NET 6/7/8 — itu fitur .NET 9+ saja -->
  <PackageReference Include="Swashbuckle.AspNetCore" Version="{versi dari tabel}" />

  <!-- EF Design — diperlukan agar dotnet ef migrations bisa jalan dengan startup-project WebApi -->
  <PackageReference Include="Microsoft.EntityFrameworkCore.Design" Version="{versi dari tabel}">
    <PrivateAssets>all</PrivateAssets>
    <IncludeAssets>runtime; build; native; contentfiles; analyzers</IncludeAssets>
  </PackageReference>

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

**Prasyarat migrations bisa berjalan:** `Microsoft.EntityFrameworkCore.Design` harus ada di project WebApi (startup project). Sudah disertakan di template WebApi.csproj di atas.

---

## Testing Packages (project terpisah)

Gunakan versi dari Compatibility Matrix.

```xml
<!-- {ProjectName}.Application.Tests.csproj -->
<Project Sdk="Microsoft.NET.Sdk">
  <PropertyGroup>
    <TargetFramework>{TFM}</TargetFramework>
    <IsPackable>false</IsPackable>
  </PropertyGroup>

  <ItemGroup>
    <PackageReference Include="Microsoft.NET.Test.Sdk" Version="{versi dari tabel}" />
    <PackageReference Include="xunit" Version="{versi dari tabel}" />
    <PackageReference Include="xunit.runner.visualstudio" Version="{versi dari tabel}" />
    <PackageReference Include="FluentAssertions" Version="{versi dari tabel}" />
    <PackageReference Include="NSubstitute" Version="{versi dari tabel}" />
    <!-- MockQueryable diperlukan agar DbSet mock mendukung async LINQ (ToListAsync, FirstOrDefaultAsync) -->
    <PackageReference Include="MockQueryable.NSubstitute" Version="{versi dari tabel}" />
  </ItemGroup>

  <ItemGroup>
    <ProjectReference Include="..\..\src\{ProjectName}.Application\{ProjectName}.Application.csproj" />
  </ItemGroup>
</Project>

<!-- {ProjectName}.Integration.Tests.csproj -->
<!-- Gunakan Testcontainers sesuai database provider yang dipilih user di step 1c: -->
<!--   SQL Server  → Testcontainers.MsSql     -->
<!--   PostgreSQL  → Testcontainers.PostgreSql -->
<!--   SQLite      → tidak perlu Testcontainers (file-based) -->
<Project Sdk="Microsoft.NET.Sdk">
  <PropertyGroup>
    <TargetFramework>{TFM}</TargetFramework>
    <IsPackable>false</IsPackable>
  </PropertyGroup>

  <ItemGroup>
    <PackageReference Include="Microsoft.NET.Test.Sdk" Version="{versi dari tabel}" />
    <PackageReference Include="xunit" Version="{versi dari tabel}" />
    <PackageReference Include="xunit.runner.visualstudio" Version="{versi dari tabel}" />
    <PackageReference Include="FluentAssertions" Version="{versi dari tabel}" />
    <PackageReference Include="Microsoft.AspNetCore.Mvc.Testing" Version="{versi dari tabel}" />
    <!-- Pilih sesuai provider: -->
    <PackageReference Include="Testcontainers.MsSql" Version="3.9.0" />
    <!-- <PackageReference Include="Testcontainers.PostgreSql" Version="3.9.0" /> -->
    <PackageReference Include="Respawn" Version="{versi dari tabel}" />
  </ItemGroup>

  <ItemGroup>
    <ProjectReference Include="..\..\src\{ProjectName}.WebApi\{ProjectName}.WebApi.csproj" />
  </ItemGroup>
</Project>
```
