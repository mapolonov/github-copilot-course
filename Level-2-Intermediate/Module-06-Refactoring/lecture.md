# Модуль 06: Рефакторинг legacy коду та документація з Copilot
## Рівень: Intermediate | Тривалість: 3 години

---

## Цілі модуля

- Систематично рефакторити legacy .NET код за допомогою Copilot
- Використовувати Copilot для зворотного інжинірингу: зрозуміти що робить код без документації
- Генерувати повну XML документацію та архітектурні ADR
- Працювати з Copilot Edits для багатофайлових рефакторингів

---

## 1. Стратегія рефакторингу legacy коду

### 1.1 Фази рефакторингу з Copilot

```
Фаза 1: ЗРОЗУМІТИ код (Copilot як документатор)
    /explain + @workspace аналіз
    
Фаза 2: ЗАХИСТИТИ код (Copilot пише тести ДО рефакторингу)
    /tests для legacy методу "as-is"
    
Фаза 3: РЕФАКТОРИТИ (Copilot пропонує зміни)
    Inline Chat з ітеративними промптами
    
Фаза 4: ВЕРИФІКУВАТИ (тести повинні залишитися зеленими)
    Запустити тести з Фази 2

Правило: НІКОЛИ не рефакторьте без тестів.
         Спочатку попросіть Copilot написати тести на поточну "ведмежу" поведінку.
```

### 1.2 Аналіз legacy коду

**Дано код без документації:**

```csharp
// Крок 1 — попросіть Copilot пояснити:
/*
/explain
What does this method do? 
List all business rules it implements.
Identify all side effects.
Flag any security issues, anti-patterns, or bugs.
*/

public decimal Calc(List<object[]> d, int t, bool f)
{
    decimal r = 0;
    for(int i = 0; i < d.Count; i++)
    {
        if((string)d[i][3] != "D")
        {
            decimal p = (decimal)d[i][1] * (int)d[i][2];
            if(f && (int)d[i][2] > 10) p = p * 0.9m;
            if(t == 1) p = p * 1.2m; // tax
            r += p;
        }
    }
    return Math.Round(r, 2);
}
```

**Copilot відповість приблизно:**
- Обчислює підсумкову суму замовлення
- Пропускає позиції зі статусом "D" (Deleted/Discounted?)
- Застосовує 10% знижку якщо `f=true` і кількість > 10 (bulk discount)
- Застосовує 20% податок якщо `t=1`
- Податок тип (ПДВ?) застосовується до всієї суми, включаючи знижкові позиції — можливий баг

### 1.3 Рефакторинг через Inline Chat

```csharp
// Виділіть метод → Ctrl+I:
/*
Refactor this method:
1. Replace cryptic parameter names: d→orderLines, t→taxType, f→applyBulkDiscount
2. Create OrderLine record from object arrays (properties: Status, UnitPrice, Quantity)  
3. Replace magic strings ("D") with OrderLineStatus.Deleted enum
4. Replace magic numbers (0.9m, 1.2m, 10) with named constants
5. Replace for loop with LINQ
6. Add XML documentation explaining business rules
7. Keep exact same behavior (do NOT fix the tax calculation bug yet — requires separate ticket)
*/
```

**Після рефакторингу:**

```csharp
/// <summary>
/// Calculates the total order amount applying bulk discounts and tax.
/// </summary>
/// <param name="orderLines">Collection of order lines with status, unit price, and quantity.</param>
/// <param name="taxType">Tax type: 0 = no tax, 1 = VAT (20%).</param>
/// <param name="applyBulkDiscount">When true, applies 10% discount on lines with quantity > 10.</param>
/// <returns>Rounded total amount (2 decimal places).</returns>
/// <remarks>
/// NOTE: Tax is applied after discount calculation. 
/// Deleted lines (Status = "D") are excluded from calculation.
/// </remarks>
public decimal CalculateOrderTotal(
    IReadOnlyList<OrderLine> orderLines, 
    int taxType, 
    bool applyBulkDiscount)
{
    private const decimal BulkDiscountRate = 0.9m;
    private const decimal VatRate = 1.2m;
    private const int BulkDiscountThreshold = 10;

    var subtotal = orderLines
        .Where(line => line.Status != OrderLineStatus.Deleted)
        .Sum(line =>
        {
            var lineTotal = line.UnitPrice * line.Quantity;
            if (applyBulkDiscount && line.Quantity > BulkDiscountThreshold)
                lineTotal *= BulkDiscountRate;
            return lineTotal;
        });

    if (taxType == 1) subtotal *= VatRate;

    return Math.Round(subtotal, 2);
}
```

---

## 2. Migration: .NET Framework → .NET 8

### 2.1 Промпты для миграции

```
Migrate #file:LegacyHttpHelper.cs from .NET Framework 4.8 to .NET 8:
- Replace HttpWebRequest/HttpWebResponse with HttpClient
- Replace synchronous IO with async/await
- Replace ConfigurationManager with IConfiguration injection
- Replace custom XML serialization with System.Text.Json
- Keep same public interface signatures where possible
- Note any breaking changes as comments
```

```
Migrate #file:ThreadingHelper.cs from .NET Framework threading to .NET 8:
- Replace Thread with Task/Task.Run
- Replace ManualResetEvent with SemaphoreSlim or TaskCompletionSource
- Replace ThreadPool.QueueUserWorkItem with Task.Run
- Replace monitor-based locking with SemaphoreSlim where async is needed
- Identify any cases where Thread.Abort was used (must be replaced with CancellationToken)
```

### 2.2 Міграція WebForms/WCF патернів

```
This is a WCF service contract. Generate equivalent ASP.NET Core minimal API or 
gRPC service definition that preserves all operations:
[ServiceContract]
public interface IOrderService
{
    [OperationContract]
    OrderResponse CreateOrder(CreateOrderRequest request);
}

Generate:
1. REST API version with OpenAPI attributes
2. gRPC .proto definition
3. Note all differences in behavior/semantics
```

---

## 3. Документація через Copilot

### 3.1 XML документація для всього класу

```csharp
// Виділіть весь клас → Ctrl+I:
/*
/doc Add comprehensive XML documentation:
- Class summary explaining purpose and usage context
- Each public method: summary, parameters, returns, exceptions, remarks
- Include <example> with code snippet for complex methods
- Add <seealso> references to related types
- Note thread safety in class-level remarks
*/
```

### 3.2 README для проєкту/модуля

```
@workspace Generate a README.md for this project that includes:
1. Architecture overview (infer from project structure)
2. Prerequisites and setup steps
3. How to run locally with Docker
4. How to run tests
5. Key configuration settings (infer from appsettings.json)
6. API endpoints overview with examples
7. Contributing guidelines
```

### 3.3 Architecture Decision Records (ADR)

```
Generate an ADR (Architecture Decision Record) for the decision to use:
- MediatR for CQRS instead of direct service calls
Context: We're building an e-commerce platform expecting high scale
- Include: Status, Context, Decision, Consequences (positive and negative)
- Include rejected alternatives considered
Format: Markdown, Nygard style
```

---

## 4. Copilot Edits — багатофайловий рефакторинг

### 4.1 Що таке Copilot Edits

Copilot Edits (у VS Code — `Ctrl+Shift+I` в режимі Edit) дозволяє застосовувати зміни **одночасно до кількох файлів**, бачити diff, приймати/відхиляти кожну зміну.

### 4.2 Сценарії для Copilot Edits

#### Сценарій 1: Додати CancellationToken всюди

```
Files: [all *Service.cs and *Repository.cs in Application/ and Infrastructure/]

Task: Add CancellationToken parameter to ALL async methods that don't have it:
- Parameter name: 'ct'
- Position: last parameter
- Propagate to all internal async calls
- DO NOT add to private methods that only call synchronous code
- DO NOT change method signatures in IXxx interfaces if they're not in scope
```

#### Сценарій 2: Перехід з AutoMapper на ручну проекцію

```
Files: [*MappingProfile.cs, *Controller.cs, *Service.cs]

Task: Remove AutoMapper dependency:
1. Delete MappingProfile classes
2. Replace .Map<DestType>(source) with explicit new DestType { ... } constructors
3. Move mapping logic into static ToDto() extension methods on source types
4. Remove IMapper injection from constructors
5. Remove AutoMapper NuGet references from csproj
```

#### Сценарій 3: Додати structured logging

```
Files: [all *Service.cs files]

Task: Add structured logging to all public service methods:
- LogInformation at method entry with key parameters
- LogWarning when business rule violations occur (Result.Failure)
- LogError (with exception) in catch blocks
- Never log passwords, tokens, credit card numbers
- Use consistent log message templates: "Processing {EntityType} {EntityId}"
```

---

## 5. Code Review з Copilot

### 5.1 Автоматизований Code Review

```csharp
// Перед PR — попросіть Copilot провести рев’ю:
/*
@workspace Perform a code review of the changes in #file:OrderService.cs:

Check for:
1. SOLID violations (SRP, DIP especially)
2. Missing error handling or swallowed exceptions
3. Performance issues (N+1, missing async, blocking calls)
4. Security concerns (injection, auth bypass, sensitive data exposure)
5. Thread safety issues
6. Missing CancellationToken propagation
7. Logic bugs or off-by-one errors
8. Inconsistency with patterns in the rest of the codebase

Return: numbered list of findings, severity (Critical/Major/Minor), and suggested fix for each.
*/
```

### 5.2 Аналіз Cyclomatic Complexity

```
Analyze #file:LegacyController.cs for:
1. Methods with cyclomatic complexity > 10 (list each with complexity score)
2. Suggest how to extract into smaller methods
3. Identify which complex methods have test coverage (check #file:LegacyControllerTests.cs)
4. Prioritize refactoring candidates by risk (complexity × lack of tests)
```

---

## 6. Автоматизація через GitHub Actions

```
Generate GitHub Actions workflow for .NET 8 project that:
- Triggers on PR to main branch
- Builds solution with dotnet build
- Runs all tests with dotnet test, publishes test results
- Runs code coverage (Coverlet) and fails if < 80%
- Runs security scan (dotnet-security-scan or similar)
- Posts Copilot code review comment on PR (GitHub CLI)
- Caches NuGet packages for speed
```

---

## Практичне завдання (45 хвилин)

### Lab 6.1 — Legacy Rescue

Отримайте від тренера файл `LegacyOrderProcessor.cs` (~150 рядків спагеті-коду).

**Послідовність:**
1. `/explain` — зрозуміти що робить код. Скласти список бізнес-правил
2. `/tests generate tests for current behavior` — написати "golden master" тести
3. Запустити тести — переконатися що зелені
4. Ітеративно рефакторити з Inline Chat, зберігаючи зелені тести
5. `/doc` — додати документацію до результату

**Критерій успіху:** Читабельний код, всі тести зелені, XML документація на всіх public методах.

---

## Підсумки модуля

- Порядок рефакторингу: ЗРОЗУМІТИ → ЗАХИСТИТИ тестами → РЕФАКТОРИТИ → ВЕРИФІКУВАТИ
- Copilot Edits — потужний інструмент для cross-cutting рефакторингів (додати CT, прибрати AutoMapper)
- Code Review промпти допомагають знайти проблеми до PR
- Документація — найнедооціненіший сценарій, Copilot робить це відмінно

**← Попередній модуль:** [Модуль 05](../Module-05-Testing/lecture.md)  
**Наступний рівень →** [Модуль 07: Архітектурні патерни](../../Level-3-Advanced/Module-07-Architecture/lecture.md)
