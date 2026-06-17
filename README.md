# dotnet-dev-skills

Claude Code skill plugin untuk pengembangan **.NET Web API** dengan **Clean Architecture + Vertical Slice Architecture (VSA)**.

## Skill yang Tersedia

### `dotnet-vsa`

Scaffold, extend, atau review proyek .NET Web API menggunakan Clean Architecture dikombinasikan dengan Vertical Slice Architecture.

**Perintah:**

```
/dotnet-vsa new-project OrderManagement
/dotnet-vsa new-feature Orders Order
/dotnet-vsa review
```

| Perintah | Fungsi |
|---|---|
| `new-project [NamaProyek]` | Scaffold project baru dari nol (4 layer, semua boilerplate) |
| `new-feature [Fitur] [Entity]` | Tambah satu vertical slice lengkap (CRUD) ke project yang ada |
| `review` | Review kode existing terhadap aturan Clean Architecture + VSA |

Saat `new-project`, skill akan menanyakan:
- Versi .NET (.NET 6 / 7 / 8 / 9) — versi package NuGet disesuaikan otomatis
- Database provider (SQL Server / PostgreSQL / SQLite)
- Pola endpoint (Controllers / Carter Minimal API)

## Stack Teknologi

- **.NET 6 / 7 / 8 / 9** — dipilih saat `new-project`
- **ASP.NET Core Web API**
- **MediatR 12** — CQRS pattern
- **FluentValidation 11** — input validation
- **Entity Framework Core** — ORM (versi menyesuaikan .NET target)
- **Carter** (opsional) — Minimal API endpoint grouping

## Instalasi

```
/plugin marketplace add Sutrisno-tris032/dotnet-dev-skills
/plugin install dotnet-vsa@dotnet-dev-skills
```

Perintah pertama mendaftarkan repositori GitHub ini sebagai marketplace; perintah kedua mengaktifkan
plugin tersebut dan mengaktifkan kelima skill sekaligus (skill-skill tersebut tidak diinstal
secara terpisah). Untuk memperbarui nanti, jalankan kembali perintah instalasi setelah
`/plugin marketplace update-plugin dotnet-vsa`.

## Struktur Skill

```
dotnet-dev-skills/
├── .claude-plugin/
│   ├── plugin.json           ← registrasi plugin
│   └── marketplace.json
├── skills/dotnet-vsa/
│   ├── SKILL.md              ← entry point, definisi semua perintah
│   ├── references/
│   │   ├── architecture.md   ← penjelasan CA + VSA, code patterns per layer
│   │   ├── conventions.md    ← naming convention & code style
│   │   └── packages.md       ← NuGet packages + compatibility matrix per .NET version
│   └── templates/
│       ├── project-structure.md  ← scaffold project lengkap dengan isi semua file
│       └── feature-slice.md      ← template CRUD lengkap per feature slice
└── README.md
```
