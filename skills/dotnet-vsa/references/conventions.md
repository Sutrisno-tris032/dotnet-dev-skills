# Konvensi Kode & Penamaan

## Konvensi Umum C#

- **File-scoped namespace** — gunakan `namespace {ProjectName}.{Layer}.{...};` (satu baris, tanpa kurung kurawal)
- **Primary constructor** — gunakan untuk inject dependency di class sederhana (C# 12+)
- **Record** untuk semua Command, Query, dan Response — immutable by default, value equality gratis
- **Collection expression** `[]` sebagai pengganti `new List<T>()` (C# 12+)
- **Null-forgiving operator** `!` hanya jika benar-benar yakin tidak null; lebih baik gunakan nullable check

## Penamaan File

| Tipe | Pola Nama File | Contoh |
|---|---|---|
| Entity | `{Entity}.cs` | `Order.cs` |
| Command | `{Action}{Entity}Command.cs` | `CreateOrderCommand.cs` |
| Command Handler | `{Action}{Entity}CommandHandler.cs` | `CreateOrderCommandHandler.cs` |
| Command Validator | `{Action}{Entity}CommandValidator.cs` | `CreateOrderCommandValidator.cs` |
| Query | `Get{Entity}{Scope}Query.cs` | `GetOrderByIdQuery.cs`, `GetOrderListQuery.cs` |
| Query Handler | `Get{Entity}{Scope}QueryHandler.cs` | `GetOrderByIdQueryHandler.cs` |
| Response/DTO | `{Entity}Response.cs` | `OrderResponse.cs` |
| List Response | `{Entity}ListResponse.cs` | `OrderListResponse.cs` |
| EF Configuration | `{Entity}Configuration.cs` | `OrderConfiguration.cs` |
| Controller | `{Feature}Controller.cs` | `OrdersController.cs` |
| Carter Module | `{Feature}Endpoints.cs` | `OrderEndpoints.cs` |
| Interface | `I{Name}.cs` | `IApplicationDbContext.cs` |

## Penamaan Symbol

```csharp
// Namespace: PascalCase, matching folder structure
namespace OrderManagement.Application.Features.Orders.Commands.CreateOrder;

// Record Command: PascalCase, suffix Command
public record CreateOrderCommand(string CustomerId, decimal TotalAmount) : IRequest<int>;

// Record Query: PascalCase, prefix Get, suffix Query
public record GetOrderByIdQuery(int OrderId) : IRequest<OrderResponse>;

// Response: PascalCase, suffix Response
public record OrderResponse(int Id, string CustomerId, string Status, decimal TotalAmount);

// Handler: sama persis dengan command/query + suffix Handler
public class CreateOrderCommandHandler : IRequestHandler<CreateOrderCommand, int> { }

// Validator: sama persis dengan command + suffix Validator (tanpa Command)
public class CreateOrderCommandValidator : AbstractValidator<CreateOrderCommand> { }

// Entity: PascalCase, singular
public class Order : AuditableEntity { }

// Private fields: _camelCase
private readonly IApplicationDbContext _context;

// Constants: PascalCase (bukan ALL_CAPS)
public const string DefaultStatus = "Pending";
```

## Struktur Folder per Feature

```
Application/Features/Orders/          ← nama fitur: PascalCase, plural
    Commands/
        CreateOrder/                  ← nama operasi: PascalCase
            CreateOrderCommand.cs
            CreateOrderCommandHandler.cs
            CreateOrderCommandValidator.cs
        UpdateOrder/
            UpdateOrderCommand.cs
            UpdateOrderCommandHandler.cs
            UpdateOrderCommandValidator.cs
        DeleteOrder/
            DeleteOrderCommand.cs
            DeleteOrderCommandHandler.cs
    Queries/
        GetOrderById/
            GetOrderByIdQuery.cs
            GetOrderByIdQueryHandler.cs
            OrderResponse.cs
        GetOrderList/
            GetOrderListQuery.cs
            GetOrderListQueryHandler.cs
            OrderListResponse.cs
```

## Aturan `using`

- Urutkan: System → Microsoft → third-party → project internal
- Gunakan **global using** di file `GlobalUsings.cs` per project untuk namespace yang sering dipakai:

```csharp
// Application/GlobalUsings.cs
global using MediatR;
global using FluentValidation;
global using Microsoft.EntityFrameworkCore;
global using {ProjectName}.Domain.Entities;
global using {ProjectName}.Application.Common.Interfaces;
global using {ProjectName}.Application.Common.Exceptions;
```

## Return Type Endpoint

### Controller
```csharp
// GET by id — bisa NotFound
public async Task<ActionResult<OrderResponse>> GetById(int id, CancellationToken ct)
{
    var result = await _sender.Send(new GetOrderByIdQuery(id), ct);
    return Ok(result);  // NotFoundException dihandle middleware
}

// POST — kembalikan 201 Created
public async Task<ActionResult<int>> Create(CreateOrderCommand command, CancellationToken ct)
{
    var id = await _sender.Send(command, ct);
    return CreatedAtAction(nameof(GetById), new { id }, id);
}

// PUT/PATCH — 204 No Content
public async Task<IActionResult> Update(int id, UpdateOrderCommand command, CancellationToken ct)
{
    await _sender.Send(command with { Id = id }, ct);
    return NoContent();
}

// DELETE — 204 No Content
public async Task<IActionResult> Delete(int id, CancellationToken ct)
{
    await _sender.Send(new DeleteOrderCommand(id), ct);
    return NoContent();
}
```

### Minimal API (Carter)
```csharp
group.MapGet("/{id:int}", async (int id, ISender sender, CancellationToken ct) =>
{
    var result = await sender.Send(new GetOrderByIdQuery(id), ct);
    return Results.Ok(result);
})
.Produces<OrderResponse>()
.ProducesProblem(StatusCodes.Status404NotFound)
.WithName("GetOrderById");
```

## Exception Handling

Semua exception dihandle oleh `ExceptionHandlingMiddleware`. Handler **tidak boleh** punya try-catch kecuali untuk rollback transaksi eksplisit.

```csharp
// NotFoundException → 404
// ValidationException → 400 dengan detail error per field
// DomainException → 422 Unprocessable Entity
// Exception lainnya → 500 (log detail, return generic message)
```

Format response error (Problem Details — RFC 7807):
```json
{
  "type": "https://tools.ietf.org/html/rfc7807",
  "title": "Validation Error",
  "status": 400,
  "errors": {
    "CustomerId": ["Customer ID is required"],
    "Items": ["At least one item is required"]
  }
}
```

## Async/Await

- Semua method yang menyentuh I/O (DB, HTTP, file) harus `async Task<T>`
- Selalu pass `CancellationToken ct` ke semua async method
- Jangan `await` method yang langsung di-`return` — gunakan `return Task` secara langsung jika tidak ada logic setelah await:

```csharp
// Kurang baik
public async Task<int> Handle(...)
{
    return await context.SaveChangesAsync(ct);  // unnecessary async overhead
}

// Lebih baik jika tidak ada logic setelah ini
public Task<int> Handle(...)
    => context.SaveChangesAsync(ct);

// Tapi jika ada multiple await, tetap pakai async/await
public async Task<OrderResponse> Handle(...)
{
    var order = await context.Orders.FindAsync([request.Id], ct)
        ?? throw new NotFoundException(nameof(Order), request.Id);
    return new OrderResponse(order.Id, order.CustomerId, order.Status.ToString(), order.TotalAmount);
}
```
