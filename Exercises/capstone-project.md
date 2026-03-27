# Фінальний Проєкт: MiniShop API
## Наскрізне завдання — всі рівні | Тривалість: 4–6 годин

---

## Мета

Реалізувати мінімальний, але **production-grade** REST API для інтернет-магазину, використовуючи **виключно Copilot** для генерації коду (з вашим критичним контролем). Проєкт охоплює всі теми курсу від Beginner до Advanced.

---

## Вимоги до проєкту

### Технічний стек
- .NET 8, ASP.NET Core Minimal API
- Entity Framework Core 8 + PostgreSQL (Testcontainers для тестів)
- MediatR (CQRS)
- FluentValidation
- xUnit + FluentAssertions + FakeItEasy
- Serilog
- Docker Compose для локального запуску

### Доменна область — `MiniShop`

```
Entities:
├── Product (Id, Name, Price, StockQuantity, CategoryId, IsActive)
├── Category (Id, Name, Slug)
├── Order (Id, CustomerId, Status, CreatedAt, OrderLines)
├── OrderLine (Id, OrderId, ProductId, Quantity, UnitPriceAtOrder)
└── Customer (Id, Email, FullName, CreatedAt)

Statuses: Draft → Placed → Confirmed → Cancelled
```

---

## Структура завдань

Кожен блок відповідає рівню навчання. Команди/учасники виконують свій рівень + за бажанням беруть наступний.

---

## 🟢 Блок Beginner: Scaffold + Basic CRUD

### Завдання 1: Структура проєкту (Copilot допомагає)

```bash
# Попроси Copilot Chat:
# "@workspace Help me set up a Clean Architecture folder structure 
#  for a .NET 8 Minimal API project called MiniShop"
```

Очікувана структура (прийми пропозицію або скоригуй):
```
MiniShop/
├── src/
│   ├── MiniShop.Domain/
│   ├── MiniShop.Application/
│   ├── MiniShop.Infrastructure/
│   └── MiniShop.Api/
└── tests/
    ├── MiniShop.Unit.Tests/
    └── MiniShop.Integration.Tests/
```

### Завдання 2: Доменні сутності

У `MiniShop.Domain/Entities/` напиши лише коментар і дай Copilot заповнити кожен клас:

```csharp
// Product entity: Id (Guid), Name (required, max 200), 
// Price (decimal, positive), StockQuantity (int, >= 0),
// CategoryId (Guid FK), IsActive (default true),
// CreatedAt (DateTimeOffset UTC), UpdatedAt (DateTimeOffset UTC nullable)
// Navigation: Category, ICollection<OrderLine>
public class Product
```

### Завдання 3: Перший endpoint — GET Products

```csharp
// В Copilot Chat:
// "Generate a GET /api/products endpoint using Minimal API that:
//  - Returns paged list of active products
//  - Supports filtering by categoryId (optional query param)
//  - Supports search by name (optional, case-insensitive)
//  - Returns ProductDto (not the domain entity)
//  - Uses MediatR GetProductsQuery
//  Follow the .github/copilot-instructions.md conventions"
```

**Критерій успіху:** `dotnet run`, виклик `GET /api/products` повертає 200.

---

## 🟡 Блок Intermediate: Core Business Logic

### Завдання 4: Створення замовлення (CQRS)

```
Промпт для Copilot Chat:
"Generate complete CQRS flow for PlaceOrder:

PlaceOrderCommand:
- CustomerId (Guid)
- Lines: IReadOnlyList<PlaceOrderLineDto> (ProductId, Quantity)
- Note: UnitPrice is loaded from Product, NOT from request

PlaceOrderCommandValidator (FluentValidation):
- CustomerId not empty
- Lines: 1-20 items, no duplicate ProductIds
- Each quantity: 1-100

PlaceOrderCommandHandler:
1. Validate customer exists
2. Load products, check all exist and IsActive
3. Check stock for each line (throw InsufficientStockException if not enough)
4. Create Order with OrderLines (UnitPriceAtOrder = Product.Price at time of order)
5. Decrement stock for each product (ExecuteUpdateAsync, not entity tracking)
6. Return Result<Guid> with new OrderId

All steps use CancellationToken.
Infrastructure failures throw, business rule violations return Result.Failure."
```

### Завдання 5: Unit тести для PlaceOrderCommandHandler

```
"/tests Generate xUnit tests for PlaceOrderCommandHandler:
 - Happy path: valid order placed, stock decremented, orderId returned
 - Customer not found: Result.Failure returned
 - Product not available (IsActive=false): Result.Failure
 - Insufficient stock: InsufficientStockException
 - Partial failure: first product ok, second has no stock → no order created (rollback)
 
 Use FluentAssertions, FakeItEasy, AutoFixture.
 Verify stock decrement call happened with correct parameters."
```

### Завдання 6: EF Core конфігурація

```csharp
// Попроси для кожної сутності:
// "EF Core IEntityTypeConfiguration for Order:
//  - Table: orders (snake_case)
//  - Status stored as string enum, not int
//  - OrderLines: owned collection → table order_lines
//  - RowVersion for optimistic concurrency
//  - All DateTimeOffset columns store as UTC
//  - Index on CustomerId"
public class OrderConfiguration : IEntityTypeConfiguration<Order>
```

---

## 🔴 Блок Advanced: Architecture + Resilience

### Завдання 7: Domain Events

```
"Add domain events to the Order aggregate:
 - OrderPlacedEvent: fires when Order transitions to Placed status
 - OrderCancelledEvent: fires on cancellation with reason
 
 Implement:
 1. Entity base class with RaiseDomainEvent and GetDomainEvents
 2. SaveChangesInterceptor that collects and dispatches events via MediatR
    AFTER SaveChanges (persist first, then publish)
 3. OrderPlacedEventHandler that logs the event with structured logging
    and updates a separate CustomerOrderStats table (total orders, total spent)"
```

### Завдання 8: Polly Resilience для зовнішнього email сервісу

```csharp
// "Implement resilience pipeline for IEmailNotificationService (HTTP client):
//  - Retry: 3 times, exponential backoff (1s, 2s, 4s) + jitter 0-500ms
//  - Circuit Breaker: open after 5 failures in 30s, half-open after 60s
//  - Timeout: 5 seconds per request
//  - Bulkhead: max 5 concurrent email sends
//  Register as named HttpClient 'email-service' with Polly pipeline.
//  Log each retry and circuit state change."
```

### Завдання 9: Integration Test для PlaceOrder endpoint

```csharp
// "Generate integration test using WebApplicationFactory + Testcontainers:
//  - Real PostgreSQL container (Testcontainers)
//  - Seed: 1 customer, 2 products with stock
//  - Call POST /api/orders with valid request
//  - Assert: 201 Created, Location header contains orderId
//  - Assert: stock decremented in DB
//  - Assert: order exists in DB with correct status
//  - Second call with same data: stock decremented again (idempotency NOT required here)
//  - Call with insufficient stock: 422 Unprocessable Entity"
```

---

## Критерії фінального оцінювання

### Beginner ✅
- [ ] Сервіс запускається через `docker-compose up`
- [ ] `GET /api/products` повертає коректну paged відповідь
- [ ] `GET /api/products/{id}` повертає 404 для неіснуючого
- [ ] Код компілюється без warnings

### Intermediate ✅
- [ ] `POST /api/orders` створює замовлення і декрементує stock
- [ ] `POST /api/orders` з недостатнім stock → 422
- [ ] Усі unit тести зелені (мінімум 15 тестів)
- [ ] EF Core міграції застосовуються при старті

### Advanced ✅
- [ ] Domain events публікуються після збереження
- [ ] CustomerOrderStats оновлюється при кожному замовленні
- [ ] Integration тести з Testcontainers зелені
- [ ] Polly pipeline налаштований для email сервісу
- [ ] `.github/copilot-instructions.md` відображає архітектуру проєкту

---

## Додаткові виклики (для найшвидших)

### Challenge A: Outbox Pattern
```
Замінити прямий dispatch domain events на Outbox:
- Зберігати events в таблицю outbox_messages в тій самій транзакції що і Order
- BackgroundService читає і публікує через MediatR кожні 5 секунд
- Ідемпотентна обробка (не публікувати двічі)
```

### Challenge B: Copilot CLI Pipeline
```
Налаштувати GitHub Actions workflow:
- Build + Test з PostgreSQL
- Coverage report у PR
- dotnet list package --vulnerable як окремий крок
- Docker image build та scan через trivy
```

### Challenge C: Read Model Optimization
```
Додати окремий QueryService для звітності:
- GET /api/reports/top-selling-products?days=30
  (топ 10 за кількістю продажів через Dapper, без EF Core)
- GET /api/reports/customer/{customerId}/history
  (історія замовлень з деталями через один JOIN запит)
Порівняти швидкість: EF Core з Include vs Dapper з JOIN
```

---

## Демо-презентація (15 хв на команду)

Кожна команда представляє:

1. **Демо** (5 хв): Live запит через Swagger / curl
2. **Код-хайлайт** (5 хв): Один цікавий prompt → результат → що скоригували
3. **Метрика** (5 хв):
   - Скільки рядків коду написано вручну vs прийнято від Copilot?
   - Яка задача зайняла найбільше ітерацій промптів?
   - Що б зробили інакше наступного разу?

---

## Репозиторій-стартер

```bash
# Попроси Copilot Chat:
# "@workspace Create a .sln file and all .csproj project files 
#  for the MiniShop Clean Architecture structure.
#  Include all necessary NuGet package references."

# Або використай готовий шаблон:
dotnet new sln -n MiniShop
dotnet new classlib -n MiniShop.Domain -o src/MiniShop.Domain
dotnet new classlib -n MiniShop.Application -o src/MiniShop.Application
dotnet new classlib -n MiniShop.Infrastructure -o src/MiniShop.Infrastructure
dotnet new webapi -n MiniShop.Api -o src/MiniShop.Api --no-openapi
dotnet new xunit -n MiniShop.Unit.Tests -o tests/MiniShop.Unit.Tests
dotnet new xunit -n MiniShop.Integration.Tests -o tests/MiniShop.Integration.Tests

# Додати проєкти до .sln
dotnet sln add src/**/*.csproj tests/**/*.csproj

# Налаштувати залежності між проєктами:
# Application → Domain
# Infrastructure → Application + Domain
# Api → Application + Infrastructure
# Tests → відповідні проєкти
```
