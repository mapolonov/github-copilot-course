# Модуль 02: Перші кроки — Inline Completions та Copilot Chat
## Рівень: Beginner | Тривалість: 3 години

---

## Цілі модуля

- Впевнено працювати з inline completions у реальних .NET сценаріях
- Розрізняти ситуації, коли використовувати inline vs Chat
- Використовувати **slash commands** та **учасники чату** (`@workspace`, `@vscode`, `#file`)
- Написати перший згенерований Copilot юніт-тест

---

## 1. Inline Completions — глибоке занурення

### 1.1 Як Copilot приймає рішення про пропозицію

Copilot аналізує **bidirectional context** — те, що ДО і ПІСЛЯ курсора одночасно:

```csharp
// ============ ЩО COPILOT БАЧИТЬ ============

// PREFIX (до курсора):
using System.Threading.Tasks;
using Microsoft.EntityFrameworkCore;

public class ProductRepository : IProductRepository
{
    private readonly AppDbContext _context;
    
    public async Task<Product?> GetByIdAsync(int id, CancellationToken ct)
    {
        // [CURSOR HERE]
        
// SUFFIX (після курсора):  
    }
    
    public async Task<IReadOnlyList<Product>> GetAllAsync(CancellationToken ct)
    {
        return await _context.Products.AsNoTracking().ToListAsync(ct);
    }
}
```

Бачачи патерн `GetAllAsync` нижче, Copilot розуміє стиль коду і запропонує:

```csharp
return await _context.Products
    .AsNoTracking()
    .FirstOrDefaultAsync(p => p.Id == id, ct);
```

### 1.2 Стратегії роботи з inline completions

#### Стратегія 1: Descriptive Method Names

Ім'я методу — це міні-промпт. Порівняйте:

```csharp
// Погане ім'я → погана пропозиція
public async Task Process(int id) { }

// Хороше ім'я → Copilot розуміє намір
public async Task DeactivateUserAndRevokeTokensAsync(int userId, CancellationToken ct) { }
```

#### Стратегія 2: Коментар перед методом

```csharp
// Validates that the order total does not exceed customer's credit limit,
// throws InsufficientCreditException with remaining limit details if exceeded
public void ValidateOrderCredit(Order order, Customer customer)
{
    // Copilot заповнить тіло методу
}
```

#### Стратегія 3: Step-by-step коментарі всередині методу

```csharp
public async Task<OrderResult> ProcessOrderAsync(CreateOrderCommand command, CancellationToken ct)
{
    // 1. Validate command
    
    // 2. Check product availability in warehouse
    
    // 3. Reserve stock with optimistic concurrency
    
    // 4. Create order and order lines
    
    // 5. Publish OrderCreated domain event
    
    // 6. Return result with order id and estimated delivery date
}
```

Пишете кроки — Copilot заповнює реалізацію за кожним коментарем.

#### Стратегія 4: Відкриті інтерфейси як контекст

```
Правило: Перед написанням реалізації — відкрийте інтерфейс у сусідній вкладці.
Copilot прочитає контракт і згенерує реалізацію у строгій відповідності.
```

---

## 2. Патерни використання для .NET розробки

### 2.1 Record / DTO генерація

```csharp
// Один коментар → повний DTO з валідацією
// CreateProductRequest: product name (required, max 200), price (positive decimal),
// category id (required), optional description, list of tag ids
public record CreateProductRequest
{
    // Copilot згенерує властивості + DataAnnotations
}
```

### 2.2 Extension Methods

```csharp
public static class QueryableExtensions
{
    // Adds pagination to any IQueryable, returns PagedResult<T> with items, total count, page info
    public static async Task<PagedResult<T>> ToPagedResultAsync<T>(
        this IQueryable<T> query, int page, int pageSize, CancellationToken ct)
    {
        // Copilot заповнить
    }
}
```

### 2.3 Конфігурація EF Core

```csharp
public class OrderConfiguration : IEntityTypeConfiguration<Order>
{
    public void Configure(EntityTypeBuilder<Order> builder)
    {
        // Order: table "orders", id is identity, 
        // CustomerId FK with cascade delete,
        // OrderDate has default value of UtcNow,
        // Status is stored as string enum, max length 50
        // TotalAmount is decimal(18,2)
        // Has many OrderLines, owned by Order
    }
}
```

### 2.4 FluentValidation

```csharp
public class CreateProductRequestValidator : AbstractValidator<CreateProductRequest>
{
    public CreateProductRequestValidator()
    {
        // Validate: Name required and between 3-200 chars,
        // Price must be > 0 and <= 1_000_000,
        // CategoryId must be positive,
        // TagIds must not exceed 10 items,
        // if Description provided, must be between 10-2000 chars
    }
}
```

---

## 3. Copilot Chat — професійне використання

### 3.1 Учасники чату (Chat Participants)

```
@workspace  — контекст всього проєкту (індексує всі файли)
@vscode     — питання про налаштування редактора, extensions
@terminal   — питання про команди терміналу
```

```
#file       — прикріпити конкретний файл до запиту
#selection  — прикріпити виділений текст
#editor     — прикріпити поточний файл
```

**Приклад комбінованого запиту:**
```
@workspace Analyze the #file:OrderService.cs and suggest improvements 
considering the IOrderRepository interface in #file:IOrderRepository.cs
```

### 3.2 Slash Commands

| Команда | Що робить | Приклад |
|---------|-----------|--------|
| `/explain` | Пояснити код | `/explain this LINQ query` |
| `/fix` | Виправити помилку | `/fix NullReferenceException in this method` |
| `/tests` | Генерувати тести | `/tests for the selected method` |
| `/doc` | Додати документацію | `/doc add XML comments` |
| `/simplify` | Спростити код | `/simplify reduce cognitive complexity` |
| `/optimize` | Оптимізувати | `/optimize this EF Core query` |

### 3.3 Ефективні запити в Chat — шаблони

#### Шаблон "Роль + Задача + Контекст + Обмеження"

```
Act as a senior .NET architect.
Refactor the #file:UserService.cs to use the Repository pattern.
The project uses .NET 8, EF Core 8, and follows Clean Architecture.
Do not change the public interface of IUserService.
```

#### Шаблон для Code Review

```
Review #file:PaymentController.cs for:
1. Security vulnerabilities (injection, unauthorized access)
2. Missing input validation
3. Incorrect async/await usage
4. Missing cancellation token propagation
Return findings as a structured list with line numbers.
```

#### Шаблон для пояснення legacy коду

```
/explain
Explain what this method does step by step.
Identify any anti-patterns and suggest modern .NET 8 alternatives.
Context: this is legacy code from .NET Framework 4.8 being migrated.
```

#### Шаблон для генерації тестів

```
/tests
Generate xUnit tests for #file:DiscountService.cs using:
- FluentAssertions for assertions
- FakeItEasy for mocking
- AutoFixture for test data
Cover: happy path, boundary conditions, exception scenarios.
Test class should follow AAA pattern (Arrange-Act-Assert).
```

---

## 4. Inline Chat (`Ctrl+I`) — коли це краще Chat панелі

Inline Chat працює прямо у файлі і ідеальний для:

### 4.1 Швидкий рефакторинг виділеного коду

```
Виділяєте код → Ctrl+I → вводите команду:

"Convert to async/await, add CancellationToken parameter, 
 propagate to all EF Core calls"
```

### 4.2 Додавання XML-документації

```
Виділяєте клас → Ctrl+I:

"Add complete XML documentation comments to all public members.
 Include <example> tags with usage examples for complex methods."
```

### 4.3 Додавання атрибутів валідації

```
Виділяєте DTO → Ctrl+I:

"Add Data Annotations validation attributes based on property names and types.
 Email properties must have [EmailAddress], phone - [Phone], etc."
```

---

## 5. Copilot в терміналі

### 5.1 `@terminal` в Chat

```
@terminal How do I create a new migration for the Orders table 
in EF Core 8 and apply it only to the test database?
```

```
@terminal Write a PowerShell script that watches the bin/Debug folder 
and restarts dotnet run on file change
```

### 5.2 Shell suggestions (VS Code)

В терміналі VS Code можна натиснути **`Ctrl+I`** прямо в терміналі:

```bash
# Введіть опис → Copilot запропонує команду:
> find all .cs files modified in the last 24 hours
# Copilot: find . -name "*.cs" -mtime -1
```

---

## 6. Практичні обмеження — що варто робити руками

| Задача | Copilot корисний? | Чому |
|--------|-----------------|--------|
| Генерація CRUD коду | ✅ Відмінно | Патерн повторюється, немає бізнес-специфіки |
| Написання migration SQL | ✅ Добре | Синтаксис стандартний |
| Алгоритми сортування/пошуку | ✅ Добре | Добре представлений у навчальних даних |
| Бізнес-правила предметної області | ⚠️ Обережно | Не знає вашого домену |
| Security-критичний код (crypto, auth) | ⚠️ Обов'язкове ревю | Часто пропонує застарілі патерни |
| Регулярні вирази | ✅ Відмінно | Попросіть пояснити і перевірте |
| LINQ-запити | ✅ Відмінно | Багатий контекст у навчанні |
| Infrastructure as Code | ✅ Добре | Docker, k8s YAML, bicep |

---

## Практичне завдання (45 хвилин)

### Lab 2.1 — CRUD API за 10 хвилин

Створіть чистий проєкт і використовуючи лише Copilot (без самостійного написання коду) створіть:

**Ціль:** `ProductsController` з повним CRUD

1. Створіть `Product.cs` записавши лише:
```csharp
// Product entity: Id, Name (required), Price (decimal), CategoryId, 
// CreatedAt (UTC), IsActive flag, ICollection of ProductTags
public class Product
```

2. Створіть `CreateProductRequest.cs`:
```csharp
// Request DTO for creating product, include FluentValidation validator in same file
public record CreateProductRequest
```

3. Відкрийте обидва файли і створіть `ProductsController.cs`:
```csharp
// RESTful controller for products: GET list with pagination, 
// GET by id, POST create, PUT update, DELETE soft-delete
[ApiController]
[Route("api/[controller]")]
public class ProductsController : ControllerBase
```

**Засічіть час** від початку до working компіляції.

### Lab 2.2 — Дослідження Chat

Використовуючи Copilot Chat дайте відповідь на питання про щойно створений код:

```
1. @workspace What's the cyclomatic complexity of ProductsController?
2. /explain the pagination logic
3. Review this controller for missing error handling
4. /tests generate tests for the GET by id endpoint
```

---

## Підсумки модуля

- Inline completion — потужний інструмент лише за правильного іменування та коментування
- `@workspace`, `#file`, `@terminal` — ключові учасники для професійної роботи
- Slash commands `/explain`, `/fix`, `/tests` прискорюють рутинні операції
- Inline Chat (`Ctrl+I`) ідеальний для точкових змін без переключення контексту
- Деякі задачі (security, domain logic) вимагають більше людського контролю

**Наступний модуль →** [Модуль 03: Основи Prompt Engineering](../Module-03-Prompt-Basics/lecture.md)
