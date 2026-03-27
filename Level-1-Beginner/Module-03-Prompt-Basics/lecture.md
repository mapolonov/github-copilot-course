# Модуль 03: Основи Prompt Engineering для .NET розробників
## Рівень: Beginner | Тривалість: 3 години

---

## Цілі модуля

- Розуміти, чому формулювання промпта безпосередньо впливає на якість коду
- Володіти 6 базовими техніками промптингу для .NET задач
- Налаштувати `.github/copilot-instructions.md` для командної уніфікації
- Вміти ітеративно покращувати промпт за поганого першого результату

---

## 1. Чому промпт важливіший за швидкість друку

### 1.1 Експеримент: одна задача — різні промпти

**Задача:** Створити метод кешування результатів з БД

#### Промпт рівня "новачок":
```
// get user from cache or db
```
**Результат:** Простий Dictionary<int, User> без TTL, без thread-safety, без async

#### Промпт рівня "спеціаліст":
```csharp
/// <summary>
/// Retrieves user by id with IMemoryCache. 
/// Cache key: "user:{id}", TTL: 5 minutes sliding expiration.
/// On cache miss: loads from repository, caches and returns.
/// Thread-safe. CancellationToken propagated to repository.
/// Returns null if user not found (no caching for misses).
/// </summary>
```
**Результат:** Повна production-ready реалізація з `IMemoryCache`, async/await, правильним TTL

**Висновок:** 30 секунд на написання хорошого промпта = 20 хвилин заощадженої роботи.

---

## 2. Шість технік промптингу для .NET

### Техніка 1: ROLE — задати роль/контекст

```
// Погано (без контексту):
// validate user input

// Добре (з роллю та стеком):
// You are implementing validation for a financial trading platform.
// Validate CreateTradeRequest: symbol must be non-empty uppercase 3-5 chars,
// quantity must be positive integer, price must be > 0 with max 2 decimal places,
// orderType must be one of [Market, Limit, StopLoss]
// Throw ValidationException with list of all violations (not fail-fast)
```

### Техніка 2: EXAMPLE — показати приклад патерну

```csharp
// Copilot реплікує патерни з того самого файлу.
// Напишіть один метод "правильно" — решта будуть у тому ж стилі.

// Приклад-якір:
public async Task<Result<Order>> GetOrderAsync(int id, CancellationToken ct)
{
    try
    {
        var order = await _repository.GetByIdAsync(id, ct);
        return order is null 
            ? Result<Order>.Failure(Error.NotFound("Order", id))
            : Result<Order>.Success(order);
    }
    catch (Exception ex)
    {
        _logger.LogError(ex, "Failed to get order {OrderId}", id);
        return Result<Order>.Failure(Error.Internal(ex.Message));
    }
}

// Тепер напишіть заголовок наступного методу — Copilot продовжить у тому ж стилі:
public async Task<Result<IReadOnlyList<Order>>> GetOrdersByCustomerAsync(
    int customerId, CancellationToken ct)
{
    // Copilot згенерує з Result<T>, try-catch, _logger — все як у прикладі вище
}
```

### Техніка 3: CONSTRAINTS — явні обмеження

```csharp
// Constraints роблять різницю між "робочим кодом" та "нашим кодом"

/// <summary>
/// Processes batch of payment transactions.
/// CONSTRAINTS:
/// - Max batch size: 100 items (throw ArgumentException if exceeded)
/// - Must use ConfigureAwait(false) on all awaits (library code)
/// - Do NOT use LINQ in hot path — use for loops for performance  
/// - Idempotent: same transactionId must not be processed twice (check in DB)
/// - Dead letter failed transactions to DLQ, do not throw on individual failures
/// - Log each failure with structured logging (transactionId, errorCode, message)
/// </summary>
```

### Техніка 4: FORMAT — задати формат виводу

```csharp
// Задати структуру → Copilot заповнює по структурі

public async Task<PagedResponse<ProductDto>> SearchProductsAsync(
    ProductSearchQuery query, CancellationToken ct)
{
    // Step 1: Build base IQueryable from Products DbSet with AsNoTracking
    
    // Step 2: Apply filters (if query.CategoryId has value), 
    //          (if query.MinPrice has value), (if query.SearchTerm not empty - use EF.Functions.Like)
    
    // Step 3: Apply sorting by query.SortBy (Name/Price/CreatedAt), 
    //          direction from query.SortDescending
    
    // Step 4: Get total count (separate query before pagination)
    
    // Step 5: Apply pagination (Skip/Take based on query.Page and query.PageSize)
    
    // Step 6: Project to ProductDto using Select (NOT AutoMapper here — explicit projection)
    
    // Step 7: Return PagedResponse with items, totalCount, page, pageSize, totalPages
}
```

### Техніка 5: NEGATIVE PROMPTING — що НЕ робити

```csharp
// Явно забороняйте антипатерни, які Copilot любить використовувати

/// <summary>
/// Sends email notification asynchronously.
/// DO NOT use .Result or .Wait() — must be properly async
/// DO NOT catch general Exception — only catch SmtpException and TimeoutException
/// DO NOT hardcode SMTP settings — use injected IEmailConfiguration
/// DO NOT use Thread.Sleep — use Task.Delay for retry delays
/// DO NOT block on async code — propagate CancellationToken
/// </summary>
public async Task SendOrderConfirmationAsync(Order order, CancellationToken ct)
```

### Техніка 6: ITERATIVE REFINEMENT — ітеративне уточнення

```
Перший промпт (грубий):
"Add error handling to PaymentService"

Copilot додасть базовий try-catch.

Другий промпт (уточнення):
"The error handling is too generic. 
 - HttpRequestException should return PaymentError.NetworkFailure
 - TimeoutException should trigger retry (max 3 times with exponential backoff)  
 - PaymentDeclinedException should return PaymentError.Declined with decline code
 - All other exceptions — log as critical and rethrow wrapped in PaymentProcessingException"

Третій промпт (фіналізація):
"Add circuit breaker state tracking — after 5 failures in 30 seconds, 
 open circuit for 60 seconds and return PaymentError.ServiceUnavailable without calling external API"
```

---

## 3. Корпоративні інструкції — `.github/copilot-instructions.md`

### 3.1 Що це і чому це важливо для команди

Файл `.github/copilot-instructions.md` у корені репозиторію **автоматично** додається в контекст всіх Copilot запитів. Це ваш спосіб налаштувати Copilot під стандарти команди.

**Без інструкцій:** Copilot пише код у універсальному стилі  
**З інструкціями:** Copilot пише код під ваші корпоративні патерни

### 3.2 Приклад корпоративного файлу інструкцій

```markdown
## Tech Stack
- .NET 8, C# 12, ASP.NET Core
- Entity Framework Core 8 (Code First, PostgreSQL)
- MediatR 12 (CQRS pattern — all business logic via Commands/Queries)
- FluentValidation for all input validation
- xUnit + FluentAssertions + FakeItEasy + AutoFixture for testing
- Serilog for structured logging (never use Console.WriteLine)

## Architecture
- Clean Architecture: Domain → Application → Infrastructure → API
- Domain layer has NO external dependencies (no EF, no HTTP clients)  
- All repositories are interfaces in Application, implemented in Infrastructure
- Domain events are published via MediatR INotification

## Coding Standards
- All public methods must have XML documentation
- Async methods MUST end with Async suffix and accept CancellationToken as last parameter
- Use Result<T> pattern for operation results (never throw for business rule violations)
- Use records for DTOs and Value Objects (immutable)
- Never use var for non-obvious types
- Prefer pattern matching over if-else chains
- All magic numbers/strings must be in constants

## Logging Standards
- Always use structured logging: _logger.LogInformation("Processing order {OrderId}", orderId)
- Log levels: Trace=internals, Debug=dev info, Information=business events, Warning=recoverable, Error=failures
- Never log sensitive data (passwords, tokens, card numbers, personal data)

## Error Handling
- Domain exceptions in Domain layer (inherit from DomainException)
- Use ProblemDetails for API error responses (RFC 7807)
- Global exception handler middleware handles unhandled exceptions

## Database
- DB-First migrations: always run through EF Core migrations, never manual SQL  
- Every migration must be reversible (Down() method required)
- Index all foreign keys
- Use UTC for all DateTime values (DateTimeOffset preferred)
```

### 3.3 Перевірка роботи інструкцій

Після створення файлу попросіть Copilot Chat:
```
Show me how you would implement a new Command handler for CreateOrder 
following the project conventions.
```

Copilot повинен запропонувати код з MediatR, FluentValidation, Result<T> — відповідно до інструкцій.

---

## 4. Контекст-менеджмент — управління вікном контексту

### 4.1 Що відкривати в редакторі перед роботою

**Правило "Відкриті вкладки = контекст"**

При написанні `OrderService`:
```
Відкрити перед роботою:
✅ IOrderRepository.cs     ← Copilot буде поважати контракт
✅ Order.cs                ← знає поля та методи домену
✅ OrderCreatedEvent.cs    ← буде публікувати правильний івент
✅ OrderResult.cs          ← поверне правильний тип
✅ Аналогічний сервіс (CustomerService.cs) ← Copilot скопіює патерн
```

### 4.2 Стратегія іменування файлів

Copilot використовує імена файлів як додатковий контекст:
- `OrderValidator.cs` → Copilot зрозуміє що це FluentValidation
- `OrderRepository.cs` → Copilot зрозуміє що це Repository pattern
- `CreateOrderCommandHandler.cs` → Copilot зрозуміє що це MediatR Command Handler

### 4.3 Управління релевантністю

```csharp
// ДОБРЕ: Включити в той же файл те, що потрібно Copilot знати

// Константи поруч з їх використанням:
internal static class OrderValidationRules
{
    public const int MaxOrderLineCount = 100;
    public const decimal MaxOrderTotal = 1_000_000m;
    public const int MinQuantityPerLine = 1;
}

// Тепер Copilot використовує ці константи в validation автоматично
```

---

## 5. Промпти для специфічних .NET задач

### 5.1 Middleware

```csharp
// ASP.NET Core middleware that:
// 1. Extracts X-Correlation-Id header (or generates new GUID if missing)
// 2. Adds correlation id to ILogger scope for entire request
// 3. Adds X-Correlation-Id to response headers
// 4. Measures request duration and logs: method, path, status code, duration in ms
// Follows IMiddleware pattern (not convention-based)
public class CorrelationMiddleware : IMiddleware
```

### 5.2 Background Service

```csharp
// BackgroundService that processes orders from a channel:
// - Reads from Channel<ProcessOrderMessage> injected via DI
// - Processes max 5 orders concurrently using SemaphoreSlim
// - Retries failed orders up to 3 times with 2-second delay (exponential: 2s, 4s, 8s)
// - Moves to dead-letter queue after max retries
// - Gracefully stops on cancellation (drains in-progress work, max 30 seconds)
// - Reports health via IHealthCheck interface
public class OrderProcessingService : BackgroundService
```

### 5.3 Generic Result Pattern

```csharp
// Implement a Result<T> type:
// - Implicit conversions from T (success) and Error (failure)
// - IsSuccess/IsFailure properties
// - Match<TResult>(Func<T, TResult> onSuccess, Func<Error, TResult> onFailure)
// - GetValueOrDefault(T defaultValue)
// - Map<TNew>(Func<T, TNew> mapper) — maps success value
// - Bind<TNew>(Func<T, Result<TNew>> binder) — railway oriented programming
// - NO exceptions for normal flow
public readonly struct Result<T>
```

---

## Практичне завдання (45 хвилин)

### Lab 3.1 — Створення корпоративних інструкцій

1. Створіть файл `.github/copilot-instructions.md` для вашого демо-проєкту
2. Додайте стандарти: async naming, logging з Serilog, Result<T>, FluentValidation
3. **До інструкцій:** попросіть Copilot створити метод збереження користувача
4. **Після інструкцій:** попросіть те саме — порівняйте результати

### Lab 3.2 — Промпт-чемпіонат

Розбийтеся на пари. Кожна пара отримує одну задачу:
- Пара 1: реалізувати метод пагінації через EF Core
- Пара 2: реалізувати retry policy для HTTP клієнта
- Пара 3: реалізувати generic repository

**Раунд 1:** Використайте мінімальний промпт (один рядок)  
**Раунд 2:** Використайте розгорнутий промпт з усіма техніками  
**Порівняйте:** якість, повноту, відповідність корпоративним стандартам

---

## Підсумки модуля

| Техніка | Застосовувати коли |
|---------|----------------|
| ROLE | Бізнес-специфічна логіка |
| EXAMPLE | Потрібна відповідність існуючому стилю |
| CONSTRAINTS | Відомі нефункціональні вимоги |
| FORMAT | Складний метод з кількома кроками |
| NEGATIVE | Уникнути відомих антипатернів |
| ITERATIVE | Перший результат близький, але не точний |

**`.github/copilot-instructions.md`** — обов'язковий елемент корпоративного впровадження Copilot.

**← Попередній модуль:** [Модуль 02](../Module-02-First-Steps/lecture.md)  
**Наступний рівень →** [Модуль 04: .NET патерни з Copilot](../../Level-2-Intermediate/Module-04-DotNet-Patterns/lecture.md)
