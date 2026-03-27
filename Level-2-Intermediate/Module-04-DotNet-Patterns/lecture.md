# Модуль 04: .NET патерни з Copilot — EF Core, Async, CQRS
## Рівень: Intermediate | Тривалість: 4 години

---

## Цілі модуля

- Використовувати Copilot для генерації production-ready EF Core коду без помилок N+1
- Правильно генерувати async/await з CancellationToken
- Застосовувати Copilot для реалізації CQRS через MediatR — від Command до Handler
- Уникати типових пасток у згенерованому коді для роботи з даними

---

## 1. Entity Framework Core — ефективна робота з Copilot

### 1.1 Проблема N+1 у згенерованому коді

Copilot **часто генерує N+1 запити** якщо не вказати явно. Це критична пастка:

```csharp
// ❌ Що Copilot генерує за замовчуванням:
var orders = await _context.Orders.ToListAsync(ct);
foreach (var order in orders)
{
    // N+1 ! Окремий запит на кожен order
    var customer = await _context.Customers.FindAsync(order.CustomerId);
    order.CustomerName = customer.Name; // Припустимо це DTO маппінг
}

// ✅ Правильний промпт: "using Include for eager loading, no N+1 queries"
var orders = await _context.Orders
    .Include(o => o.Customer)
    .Include(o => o.OrderLines)
        .ThenInclude(l => l.Product)
    .AsNoTracking()  // Read-only query
    .ToListAsync(ct);
```

**Правило для промптів:** Завжди вказуйте — `"use eager loading with Include"` або `"use explicit projection to avoid N+1"`.

### 1.2 Проекція замість завантаження всієї сутності

```csharp
// Промпт:
// Query orders for customer id, project to OrderSummaryDto
// Include only: OrderId, OrderDate, TotalAmount, CustomerName, ItemCount
// Use Select projection directly in EF Core query (no AutoMapper)
// NO loading of full Order entities — direct projection for performance

public async Task<IReadOnlyList<OrderSummaryDto>> GetOrderSummariesAsync(
    int customerId, CancellationToken ct)
{
    // Copilot згенерує:
    return await _context.Orders
        .Where(o => o.CustomerId == customerId)
        .Select(o => new OrderSummaryDto
        {
            OrderId = o.Id,
            OrderDate = o.CreatedAt,
            TotalAmount = o.TotalAmount,
            CustomerName = o.Customer.FullName,
            ItemCount = o.OrderLines.Count
        })
        .AsNoTracking()
        .ToListAsync(ct);
}
```

### 1.3 Оптимістичний паралелізм з Copilot

```csharp
// Промпт для Entity з optimistic concurrency:
// Product entity with optimistic concurrency via RowVersion timestamp
// UpdateProductStock: find by id, check new stock >= 0,
// update stock, handle DbUpdateConcurrencyException with retry (max 3 attempts)
// Return updated stock level or throw StockUpdateConflictException after max retries

public async Task<int> UpdateProductStockAsync(
    int productId, int delta, CancellationToken ct)
{
    const int maxRetries = 3;
    for (int attempt = 0; attempt < maxRetries; attempt++)
    {
        try
        {
            var product = await _context.Products
                .FirstOrDefaultAsync(p => p.Id == productId, ct)
                ?? throw new ProductNotFoundException(productId);

            var newStock = product.StockQuantity + delta;
            if (newStock < 0) throw new InsufficientStockException(productId, Math.Abs(delta));

            product.StockQuantity = newStock;
            await _context.SaveChangesAsync(ct);
            return newStock;
        }
        catch (DbUpdateConcurrencyException) when (attempt < maxRetries - 1)
        {
            await Task.Delay(TimeSpan.FromMilliseconds(50 * Math.Pow(2, attempt)), ct);
            _context.ChangeTracker.Clear(); // Скинути state
        }
    }
    throw new StockUpdateConflictException(productId);
}
```

### 1.4 Bulk операції

```csharp
// Промпт:
// Bulk deactivate products where LastSoldDate is older than cutoffDate
// Use ExecuteUpdateAsync (EF Core 7+, no entity loading, single SQL UPDATE)
// Return count of updated records
// Log operation with structured logging including cutoffDate and count

public async Task<int> BulkDeactivateStaleProductsAsync(
    DateTimeOffset cutoffDate, CancellationToken ct)
{
    var count = await _context.Products
        .Where(p => p.IsActive && p.LastSoldDate < cutoffDate)
        .ExecuteUpdateAsync(s => s
            .SetProperty(p => p.IsActive, false)
            .SetProperty(p => p.UpdatedAt, DateTimeOffset.UtcNow), ct);

    _logger.LogInformation(
        "Bulk deactivated {Count} stale products with LastSoldDate before {CutoffDate}",
        count, cutoffDate);
    
    return count;
}
```

---

## 2. Async/Await — правильні патерни через Copilot

### 2.1 Типові помилки в згенерованому async коді

```csharp
// ❌ Copilot іноді пропонує це (deadlock в ASP.NET Core):
public User GetUser(int id)
{
    return _repository.GetByIdAsync(id).Result; // НІКОЛИ
}

// ❌ async void — проковтує виключення:
public async void ProcessOrder(int id)  // ТІЛЬКИ для event handlers
{
    await _service.ProcessAsync(id);
}

// ❌ Не передає CancellationToken:
public async Task<List<User>> GetAllAsync()
{
    return await _context.Users.ToListAsync(); // Немає CT!
}
```

**Промпт-захист:**
```
IMPORTANT: All async methods must:
- Accept CancellationToken as last parameter named 'ct'
- Propagate ct to ALL internal async calls
- Never use .Result, .Wait(), or GetAwaiter().GetResult()
- Use ConfigureAwait(false) ONLY in library/infrastructure code, never in ASP.NET Core handlers
```

### 2.2 Паралельне виконання через Copilot

```csharp
// Промпт:
// Load product details and reviews in parallel using Task.WhenAll
// Both can fail independently — capture both results
// If product not found (null), throw ProductNotFoundException immediately
// If reviews fail, log warning and return empty list (graceful degradation)

public async Task<ProductDetailsResponse> GetProductDetailsAsync(
    int productId, CancellationToken ct)
{
    var productTask = _productRepo.GetByIdAsync(productId, ct);
    var reviewsTask = _reviewService.GetReviewsAsync(productId, ct);

    await Task.WhenAll(productTask, reviewsTask);

    var product = await productTask
        ?? throw new ProductNotFoundException(productId);

    IReadOnlyList<ReviewDto> reviews;
    try
    {
        reviews = await reviewsTask;
    }
    catch (Exception ex)
    {
        _logger.LogWarning(ex, "Failed to load reviews for product {ProductId}", productId);
        reviews = Array.Empty<ReviewDto>();
    }

    return new ProductDetailsResponse(product, reviews);
}
```

### 2.3 Streaming з IAsyncEnumerable

```csharp
// Промпт:
// Stream large report data row by row using IAsyncEnumerable<T>
// Read from database using EF Core AsAsyncEnumerable()
// Apply optional filter, yield each row transformed to ReportRow DTO
// Caller controls iteration speed (backpressure)
// CancellationToken must be checked at each yield

public async IAsyncEnumerable<SalesReportRow> StreamSalesReportAsync(
    SalesReportFilter filter,
    [EnumeratorCancellation] CancellationToken ct)
{
    var query = _context.Sales
        .AsNoTracking()
        .Where(s => s.Date >= filter.From && s.Date <= filter.To);

    if (filter.RegionId.HasValue)
        query = query.Where(s => s.RegionId == filter.RegionId);

    await foreach (var sale in query.AsAsyncEnumerable().WithCancellation(ct))
    {
        yield return new SalesReportRow
        {
            Date = sale.Date,
            Amount = sale.Amount,
            Region = sale.Region.Name,
            Product = sale.Product.Name
        };
    }
}
```

---

## 3. CQRS з MediatR — повний цикл через Copilot

### 3.1 Структура проєкту

```
Application/
├── Orders/
│   ├── Commands/
│   │   ├── CreateOrder/
│   │   │   ├── CreateOrderCommand.cs
│   │   │   ├── CreateOrderCommandHandler.cs
│   │   │   └── CreateOrderCommandValidator.cs
│   │   └── CancelOrder/
│   │       ├── CancelOrderCommand.cs
│   │       └── CancelOrderCommandHandler.cs
│   └── Queries/
│       └── GetOrderById/
│           ├── GetOrderByIdQuery.cs
│           ├── GetOrderByIdQueryHandler.cs
│           └── OrderDetailsDto.cs
```

### 3.2 Промпт для Command + Handler + Validator за один раз

```csharp
// Create complete CQRS structure for CreateOrder:
// 
// CreateOrderCommand (record): CustomerId, IReadOnlyList<OrderLineItem> Lines
//   - OrderLineItem (record): ProductId, Quantity, UnitPrice (from catalog, not from request)
//
// CreateOrderCommandValidator (FluentValidation):
//   - CustomerId > 0
//   - Lines not empty, max 50 items
//   - Each line: ProductId > 0, Quantity between 1-1000
//
// CreateOrderCommandHandler (IRequestHandler<CreateOrderCommand, Result<Guid>>):
//   - Validate customer exists (ICustomerRepository)
//   - Load products and validate all exist (IProductRepository.GetByIdsAsync)
//   - Check stock availability for each line
//   - Create Order aggregate with OrderLines
//   - Save via IOrderRepository
//   - Publish OrderCreatedEvent via IPublisher
//   - Return Result<Guid> with new OrderId
//
// Use constructor injection, CancellationToken, structured logging

public record CreateOrderCommand(
    int CustomerId,
    IReadOnlyList<OrderLineItem> Lines) : IRequest<Result<Guid>>;
```

Copilot з таким промптом згенерує майже production-ready код.

### 3.3 Pipeline Behaviors для cross-cutting concerns

```csharp
// Промпт:
// MediatR IPipelineBehavior<TRequest, TResponse> implementations:
//
// 1. ValidationBehavior: runs all IValidator<TRequest> validators via DI,
//    if any failures - return Result failure with ValidationError list,
//    never throw, use Result pattern
//
// 2. LoggingBehavior: log request name and duration,
//    log Warning if duration > 500ms, log Error if unhandled exception
//    include request type name in structured log
//
// 3. TransactionBehavior: wrap command handlers (ICommand marker interface) 
//    in EF Core transaction, rollback on exception,
//    do NOT wrap queries (they are read-only)

public class ValidationBehavior<TRequest, TResponse> 
    : IPipelineBehavior<TRequest, TResponse>
    where TRequest : IRequest<TResponse>
    where TResponse : IResult  // Ваш Result marker interface
```

---

## 4. Repository Pattern — генерація через Copilot

### 4.1 Generic Repository з правильними обмеженнями

```csharp
// Generic repository interface:
// IRepository<TEntity, TId> where TEntity : Entity<TId>
// Methods:
//   - GetByIdAsync(TId id, CancellationToken ct) → TEntity?
//   - GetAllAsync(CancellationToken ct) → IReadOnlyList<TEntity>
//   - FindAsync(Expression<Func<TEntity, bool>> predicate, CancellationToken ct) → IReadOnlyList<TEntity>
//   - AddAsync(TEntity entity, CancellationToken ct) → void
//   - Update(TEntity entity) → void (sync, EF tracks changes)
//   - Remove(TEntity entity) → void (sync)
//   - ExistsAsync(TId id, CancellationToken ct) → bool
//
// EfRepository<TEntity, TId> implementation:
//   - Constructor accepts DbContext
//   - FindAsync uses AsNoTracking() + predicate
//   - ExistsAsync uses AnyAsync (no entity loading)
public interface IRepository<TEntity, TId> where TEntity : Entity<TId>
```

### 4.2 Specification Pattern

```csharp
// Specification pattern for complex queries:
// ISpecification<T> with: Criteria (Expression), Includes (List of Expression),
// IncludeStrings (for multi-level), OrderBy, OrderByDescending,
// Take, Skip, IsPagingEnabled, IsNoTracking
//
// EfSpecificationEvaluator<T> that builds IQueryable from ISpecification:
// Apply criteria, includes, ordering, pagination, no-tracking
//
// Example spec: ActiveProductsInCategorySpec(int categoryId)
//   - Criteria: p => p.IsActive && p.CategoryId == categoryId
//   - OrderBy: p => p.Name
//   - Include: p.Category, p.ProductTags

public abstract class Specification<T>
```

---

## Практичне завдання (60 хвилин)

### Lab 4.1 — Рефакторинг з Copilot

Дано "сирий" сервіс (частина надається тренером):

```csharp
public class BadOrderService
{
    private readonly string _connectionString;
    
    public List<Order> GetOrders(int customerId)
    {
        using var conn = new SqlConnection(_connectionString);
        var orders = conn.Query<Order>("SELECT * FROM Orders WHERE CustomerId = @id", 
            new { id = customerId }).ToList();
        
        foreach (var order in orders)
        {
            order.Lines = conn.Query<OrderLine>(
                "SELECT * FROM OrderLines WHERE OrderId = @id", 
                new { id = order.Id }).ToList();
        }
        return orders;
    }
}
```

**Завдання:** За допомогою Copilot Chat (`/fix` та ітеративні промпти) приведіть до:
1. Async/await з CancellationToken
2. EF Core з eager loading (без N+1)
3. Repository pattern
4. Result<T> тип, що повертається
5. Структуроване логування

### Lab 4.2 — CQRS Feature

Реалізуйте повний CQRS флоу для `ApproveOrder`:
- Command з OrderId та ApproverId
- Бізнес-правила: замовлення повинно бути в статусі Pending, ApproverId != CustomerId
- Доменна подія `OrderApprovedEvent`
- Оновлення статусу та аудит лог
- Unit тести на Handler

---

## Підсумки модуля

- EF Core: завжди специфікувати `Include`, `AsNoTracking`, проекцію через `Select` — Copilot не робить це за замовчуванням
- Async: явна заборона на `.Result/.Wait()` у промпті — обов’язкова
- CQRS: детальний промпт з бізнес-правилами дає 90% готовий Handler
- Pipeline Behaviors — ідеальний кандидат для генерації Copilot (шаблонний код)

**← Попередній модуль:** [Модуль 03](../../Level-1-Beginner/Module-03-Prompt-Basics/lecture.md)  
**Наступний модуль →** [Модуль 05: Тестування](../Module-05-Testing/lecture.md)
