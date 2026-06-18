---
name: dotnet-vsa
description: >-
  Scaffold, extend, or review a .NET Web API project using Clean Architecture
  combined with Vertical Slice Architecture (VSA). Invoke with: new-project,
  new-feature, or review.
---

# .NET Web API — Clean Architecture + Vertical Slice Architecture

Skill ini membantu membangun, memperluas, dan mereview proyek **ASP.NET Core Web API** yang menerapkan **Clean Architecture** (CA) digabung dengan **Vertical Slice Architecture** (VSA). Hasilkan kode yang production-ready, konsisten, dan langsung bisa di-build — bukan boilerplate kosong.

## Cara Invoke

```
/dotnet-vsa new-project [ProjectName]
/dotnet-vsa new-feature [FeatureName] [EntityName]
/dotnet-vsa review
```

Jika argumen tidak diberikan, tanyakan yang diperlukan sebelum memulai.

---

## Prinsip Arsitektur

Baca `references/architecture.md` untuk penjelasan lengkap. Ringkasan:

| Lapisan | Tanggung Jawab | Boleh Bergantung Ke |
|---|---|---|
| **Domain** | Entity, Value Object, Domain Event, Exception domain | Tidak ada (pure) |
| **Application** | Use case (Command/Query), Handler, Validator, Interface | Domain |
| **Infrastructure** | EF Core, Repository impl, External service | Application, Domain |
| **WebApi** | Endpoint, Middleware, DI wiring | Application |

**VSA diterapkan di Application layer:** setiap fitur punya folder sendiri yang berisi Command/Query + Handler + Validator + Response — tidak ada folder horizontal `Commands/` atau `Queries/` di root Application.

---

## Perintah: `new-project`

### Langkah eksekusi

1. Tanyakan (jika belum ada di args) — **tanyakan semua poin ini sebelum membuat file apapun**:

   **a. Nama proyek** (mis. `OrderManagement`)

   **b. Versi .NET** — tampilkan pilihan berikut dan minta user memilih:
   ```
   Pilih versi .NET:
   [1] .NET 6  (LTS, support s/d Nov 2024)
   [2] .NET 7  (STS, support s/d May 2024)
   [3] .NET 8  (LTS, support s/d Nov 2026) ← direkomendasikan jika butuh LTS
   [4] .NET 9  (STS, support s/d May 2026) ← terbaru, fitur terbanyak
   ```
   Setelah user memilih, baca tabel versi package di `references/packages.md` bagian
   **"Compatibility Matrix"** dan gunakan versi package yang sesuai untuk SEMUA
   `PackageReference` yang dihasilkan. Jangan hardcode versi — selalu lookup dari tabel.

   **c. Database provider** (default: **SQL Server**; opsi: PostgreSQL, SQLite)

   **d. Pola endpoint** — Controllers (default) atau Carter Minimal API

2. Baca `references/packages.md` → bagian **Compatibility Matrix** untuk mendapatkan
   versi eksak semua package berdasarkan .NET version yang dipilih user.

3. Baca `templates/project-structure.md` untuk struktur folder dan isi file.

4. Buat **seluruh** file berikut (isi dengan kode nyata, bukan placeholder).
   Ganti SEMUA placeholder versi dengan versi eksak dari tabel Compatibility Matrix:
   - `Directory.Build.props` (di root solution — properti shared untuk semua project)
   - Solution file (`.sln`)
   - Empat project `.csproj` dengan `PackageReference` sesuai versi yang dipilih
   - `Domain/Common/BaseEntity.cs`, `AuditableEntity.cs`, `BaseEvent.cs`
   - `Domain/Exceptions/DomainException.cs`
   - `Application/Common/Behaviors/ValidationBehavior.cs`, `LoggingBehavior.cs`
   - `Application/Common/Exceptions/ValidationException.cs`, `NotFoundException.cs`
   - `Application/Common/Interfaces/IApplicationDbContext.cs`, `ICurrentUserService.cs`
   - `Application/Common/Models/ApiResponse.cs`
   - `Infrastructure/Persistence/ApplicationDbContext.cs`
   - `Infrastructure/Persistence/Interceptors/AuditableEntityInterceptor.cs`
   - `Infrastructure/Services/CurrentUserService.cs`
   - `Infrastructure/DependencyInjection.cs`
   - `Application/DependencyInjection.cs`
   - `WebApi/Program.cs` (wiring DI + middleware + Swagger; gunakan Template A untuk Controllers, Template B untuk Carter)
   - `WebApi/Middleware/ExceptionHandlingMiddleware.cs`
   - `WebApi/appsettings.json` (connection string sesuai database provider yang dipilih)
   - `WebApi/appsettings.Development.json` (development overrides dengan log level Debug)
   - `.gitignore` (standar .NET)

5. Di awal output, tampilkan ringkasan pilihan yang digunakan:
   ```
   Target Framework : net8.0
   MediatR          : 12.4.1
   FluentValidation : 11.11.0
   EF Core          : 8.0.x
   ...
   ```
6. Tampilkan ringkasan struktur folder yang dibuat.
7. Tunjukkan perintah berikutnya: `dotnet build` dan cara menambah fitur pertama.

### Aturan wajib saat scaffold

- Semua project harus bisa **`dotnet build`** tanpa error.
- `TargetFramework` di semua `.csproj` harus konsisten sesuai pilihan user (mis. `net8.0`).
- Versi package WAJIB diambil dari tabel Compatibility Matrix di `references/packages.md` —
  **jangan gunakan `*` wildcard** di output akhir; tuliskan versi eksak (mis. `12.4.1`).
- Untuk .NET 6/7: hindari fitur C# 12 (`primary constructor`, collection expression `[]`) —
  gunakan sintaks yang kompatibel (.NET 6 = C# 10, .NET 7 = C# 11, .NET 8/9 = C# 12/13).
- Gunakan **MediatR 12+** (bukan versi lama dengan namespace `MediatR.Extensions.*`).
- **Domain.csproj** harus referensi `MediatR.Contracts` (bukan `MediatR`) — hanya interface `INotification`.
- Gunakan **FluentValidation.DependencyInjectionExtensions** — daftarkan validator via `AddValidatorsFromAssembly`.
- `ApplicationDbContext` implement `IApplicationDbContext` yang didefinisikan di Application.
- `Program.cs` harus memanggil `builder.Services.AddApplicationServices()` dan `builder.Services.AddInfrastructureServices(builder.Configuration)`.
- **OpenAPI/Swagger:** Untuk .NET 6/7/8 WAJIB gunakan Swashbuckle (`AddEndpointsApiExplorer` + `AddSwaggerGen`).
  `AddOpenApi()` dan `MapOpenApi()` adalah **fitur .NET 9+ saja** — jangan generate untuk .NET < 9.
- **`Microsoft.EntityFrameworkCore.Design` WAJIB ada di WebApi.csproj** (`PrivateAssets=all`) agar
  `dotnet ef migrations add --startup-project WebApi` bisa berjalan.
- **Database provider** — sertakan HANYA package dan DI call yang sesuai pilihan user (step 1c):
  - SQL Server → `Microsoft.EntityFrameworkCore.SqlServer` + `options.UseSqlServer(connStr)`
  - PostgreSQL → `Npgsql.EntityFrameworkCore.PostgreSQL` + `options.UseNpgsql(connStr)`
  - SQLite → `Microsoft.EntityFrameworkCore.Sqlite` + `options.UseSqlite(connStr)`
  Jangan sertakan package provider yang tidak dipilih user.
- **Clean Architecture:** Handler TIDAK BOLEH inject `ApplicationDbContext` atau `DbContext` langsung —
  selalu gunakan `IApplicationDbContext`. WebApi TIDAK BOLEH mengakses DbContext sama sekali.
- **Carter:** Jika user memilih Carter, `Program.cs` WAJIB memanggil `builder.Services.AddCarter()` dan `app.MapCarter()` — tanpa keduanya semua endpoint Carter tidak akan terdaftar (404).

---

## Perintah: `new-feature`

### Langkah eksekusi

1. Tanyakan (jika belum ada di args):
   - Nama fitur/modul (mis. `Orders`, `Products`)
   - Nama entity utama (mis. `Order`, `Product`)
   - Operasi yang dibutuhkan: Create, Read (GetById, GetList), Update, Delete (centang semua yang relevan)
   - Apakah entity perlu kolom audit (`CreatedAt`, `UpdatedAt`, `CreatedBy`)
2. Baca `templates/feature-slice.md` untuk pola lengkap tiap slice.
3. Buat file-file berikut sesuai operasi yang dipilih:

#### Struktur folder satu fitur

```
Domain/
└── Entities/
    └── {Entity}.cs

Application/
└── Features/
    └── {Feature}/
        ├── Commands/
        │   ├── Create{Entity}/
        │   │   ├── Create{Entity}Command.cs
        │   │   ├── Create{Entity}CommandHandler.cs
        │   │   └── Create{Entity}CommandValidator.cs
        │   ├── Update{Entity}/
        │   │   ├── Update{Entity}Command.cs
        │   │   ├── Update{Entity}CommandHandler.cs
        │   │   └── Update{Entity}CommandValidator.cs
        │   └── Delete{Entity}/
        │       ├── Delete{Entity}Command.cs
        │       └── Delete{Entity}CommandHandler.cs
        └── Queries/
            ├── Get{Entity}ById/
            │   ├── Get{Entity}ByIdQuery.cs
            │   ├── Get{Entity}ByIdQueryHandler.cs
            │   └── {Entity}Response.cs
            └── Get{Entity}List/
                ├── Get{Entity}ListQuery.cs
                ├── Get{Entity}ListQueryHandler.cs
                └── {Entity}ListResponse.cs

Infrastructure/
└── Persistence/
    └── Configurations/
        └── {Entity}Configuration.cs

WebApi/
└── Endpoints/           ← jika pakai Carter
    └── {Feature}Endpoints.cs
    -- ATAU --
└── Controllers/         ← jika pakai Controllers
    └── {Feature}Controller.cs
```

4. Daftarkan `DbSet<{Entity}>` di `IApplicationDbContext` dan `ApplicationDbContext`.
5. Tampilkan migration command: `dotnet ef migrations add Add{Entity} --project Infrastructure --startup-project WebApi`.

### Aturan wajib saat new-feature

- Setiap Command/Query adalah `record` (bukan class).
- **Handler inject `IApplicationDbContext` (interface) — TIDAK BOLEH inject `ApplicationDbContext`, `DbContext`,
  atau class konkret EF Core apapun.** Ini adalah aturan CA yang tidak boleh dilanggar.
- Validator menggunakan `AbstractValidator<TCommand>`.
- Response/DTO adalah `record` — tidak boleh mengekspos entity Domain langsung ke endpoint.
- `DeleteCommand` hanya berisi `Id`; handler lempar `NotFoundException` jika tidak ditemukan.
- Endpoint mengembalikan `Results<Ok<T>, NotFound, BadRequest<...>>` (typed results) jika pakai minimal API.
- **Semua endpoint WAJIB membungkus response dalam `ApiResponse<T>`** — gunakan `ApiResponse<T>.Ok(data)` untuk sukses. Untuk operasi tanpa return data (Update/Delete), kembalikan `ApiResponse<object?>.Ok(null, "pesan")` dengan HTTP 200.
- `ExceptionHandlingMiddleware` sudah menangani error dengan format `ApiResponse<object?>.Fail(message, errors)` — endpoint tidak perlu menangkap exception.

---

## Perintah: `review`

### Langkah eksekusi

1. Minta user menunjukkan file atau folder yang akan direview (atau baca semua file `.cs` di working directory).
2. Periksa setiap item dalam checklist di bawah.
3. Laporkan temuan dengan format:

```
## Review: Clean Architecture + VSA

### Pelanggaran Dependency Rule
- [file]: [masalah] → [rekomendasi]

### Pelanggaran VSA Pattern
- [file]: [masalah] → [rekomendasi]

### Masalah Kualitas Kode
- [file:baris]: [masalah] → [rekomendasi]

### Hal yang sudah benar ✓
- ...
```

### Checklist review

**Dependency Rule (CA)**
- [ ] Domain tidak import namespace dari Application/Infrastructure/WebApi
- [ ] Application tidak import namespace dari Infrastructure/WebApi
- [ ] Infrastructure hanya import Application dan Domain
- [ ] WebApi tidak akses repository/DbContext langsung (harus lewat MediatR)

**VSA Pattern**
- [ ] Setiap fitur punya folder sendiri di `Features/`
- [ ] Tidak ada handler di luar folder fiturnya sendiri
- [ ] Tidak ada "shared" handler yang melayani banyak fitur
- [ ] Command dan Query adalah `record`

**CQRS & MediatR**
- [ ] Command mengubah state; Query hanya membaca (tidak ada side-effect di query)
- [ ] Setiap Command/Query punya satu Handler
- [ ] `ValidationBehavior` terdaftar sebagai pipeline behavior

**Entity & Domain**
- [ ] Entity punya private setter atau value object untuk field yang critical
- [ ] Tidak ada business logic di handler (logic domain ada di entity/domain service)
- [ ] `BaseEntity` digunakan konsisten

**Persistence**
- [ ] Repository tidak di-inject langsung di handler — gunakan `IApplicationDbContext`
- [ ] Setiap entity punya `IEntityTypeConfiguration<T>`
- [ ] Tidak ada raw SQL kecuali di query kompleks dengan `FromSqlRaw` yang aman

**API / Endpoint**
- [ ] Tidak ada business logic di controller/endpoint
- [ ] Endpoint hanya: terima request → kirim ke MediatR → kembalikan response
- [ ] Error handling terpusat di middleware, bukan try-catch per endpoint
- [ ] Semua response dibungkus `ApiResponse<T>` (sukses maupun error)

---

## Konvensi penamaan

Baca `references/conventions.md` untuk detail lengkap. Ringkasan cepat:

| Artifact | Pola |
|---|---|
| Command | `Create{Entity}Command`, `Update{Entity}Command`, `Delete{Entity}Command` |
| Query | `Get{Entity}ByIdQuery`, `Get{Entity}ListQuery` |
| Handler | `{CommandOrQuery}Handler` (di file terpisah) |
| Validator | `{Command}Validator` |
| Response | `{Entity}Response`, `{Entity}ListResponse` |
| Endpoint (Carter) | `{Feature}Endpoints : ICarterModule` |
| Controller | `{Feature}Controller : ControllerBase` |

---

## Berkas pendukung

- `references/architecture.md` — penjelasan mendalam CA + VSA, trade-off, dan keputusan desain
- `references/conventions.md` — naming convention, file structure, code style
- `references/packages.md` — daftar NuGet package yang dipakai dan versi yang direkomendasikan
- `templates/project-structure.md` — template scaffold project lengkap dengan semua isi file
- `templates/feature-slice.md` — template kode lengkap untuk satu feature slice (CRUD)
