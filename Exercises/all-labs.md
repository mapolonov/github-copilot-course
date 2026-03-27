# Практичні лабораторні роботи — Всі рівні
## Quickstart для тренера

---

## Beginner Labs — Рівень 1

### Lab B-1: Знайомство з Copilot (45 хв)
**Файл:** `Exercises/Beginner-Labs/Lab-B1-GettingStarted.md`

**Мета:** Перший робочий досвід з Copilot, налаштування середовища

**Кроки:**
```
1. Створити новий Web API проект:
   dotnet new webapi -n CopilotFirstSteps --no-openapi

2. Відкрити Program.cs — спостерігати як Copilot доповнює реєстрацію сервісів

3. Створити файл Models/Product.cs
   Написати коментар: // Product with Id, Name, Price, Category, IsAvailable
   Дати Copilot згенерувати клас

4. Створити Controllers/ProductsController.cs
   Написати тільки [ApiController] та [Route] атрибути — Copilot запропонує решту
   
5. Відкрити Copilot Chat:
   "Explain what this controller does"
   "/tests generate basic tests for ProductsController"

Критерій успіху: Проект компілюється, студент прийняв >5 пропозицій Copilot
```

**Поширені проблеми:**
- Copilot не показує пропозиції → перевірити статус в status bar (іконка Copilot)
- Пропозиції нерелевантні → переконатися що `.cs` файл активний
- VS Studio: перевірити Tools → Options → GitHub → Copilot → Enable

---

### Lab B-2: Prompt Engineering Challenge (45 хв)
**Мета:** Порівняти якість різних промптів

**Завдання (пари):**

Кожна пара реалізує одну і ту ж функцію тричі:

**Раунд 1 — Minimal Prompt:**
```csharp
// send email
public async Task SendEmail()
```

**Раунд 2 — Descriptive Name:**
```csharp
public async Task SendOrderConfirmationEmailAsync(
    string recipientEmail, Order order, CancellationToken ct)
```

**Раунд 3 — Full Context Prompt:**
```csharp
/// <summary>
/// Sends HTML order confirmation email.
/// Uses injected IEmailSender (SMTP abstraction).
/// Subject: "Order #{OrderNumber} Confirmed"
/// Body: customer name, order date, list of items, total amount.
/// Throws EmailDeliveryException if SMTP fails (do NOT swallow).
/// Logs sent email at Information level with recipient and orderId.
/// </summary>
public async Task SendOrderConfirmationEmailAsync(
    string recipientEmail, Order order, CancellationToken ct)
```

**Обговорення:** Що змінилося? В якому раунді код найбільш production-ready?

---

### Lab B-3: Copilot Chat Investigation (30 хв)
**Мета:** Опанувати Chat для дослідження та документування

**Надати студентам файл `LegacySnippet.cs`:**
```csharp
public class Proc 
{
    public object[] Run(object[][] matrix, int[] cfg, bool[] flags)
    {
        var res = new List<object>();
        for(int i = 0; i < matrix.Length; i++) 
        {
            if(flags[i % flags.Length])
            {
                double v = 0;
                for(int j = 0; j < matrix[i].Length; j++)
                    v += Convert.ToDouble(matrix[i][j]) * cfg[j % cfg.Length];
                if(v > 100) res.Add(Math.Round(v, 2));
            }
        }
        return res.ToArray();
    }
}
```

**Завдання через Copilot Chat:**
1. `/explain` — що робить цей код?
2. Попросити перейменувати всі змінні
3. Попросити додати unit тест
4. Попросити визначити баги

---

## Intermediate Labs — Рівень 2

### Lab I-1: EF Core Performance Kata (60 хв)
**Мета:** Усунути N+1 та оптимізувати EF Core запити з Copilot

**Надати проект з навмисними проблемами:**

```csharp
// BadOrderQueryService.cs — навмисно поганий код
public async Task<List<OrderReportDto>> GetOrderReportAsync(int year)
{
    var orders = await _context.Orders
        .Where(o => o.CreatedAt.Year == year)
        .ToListAsync();  // проблема 1: немає AsNoTracking
    
    var result = new List<OrderReportDto>();
    foreach (var order in orders)
    {
        var customer = await _context.Customers  // проблема 2: N+1
            .FindAsync(order.CustomerId);
        
        var lines = await _context.OrderLines   // проблема 3: N+1
            .Where(l => l.OrderId == order.Id)
            .ToListAsync();
        
        result.Add(new OrderReportDto
        {
            OrderId = order.Id,
            CustomerName = customer!.FullName,   // проблема 4: null!
            LineCount = lines.Count,
            Total = lines.Sum(l => l.UnitPrice * l.Quantity)  // проблема 5: завантажує весь об'єкт
        });
    }
    return result;
}
```

**Завдання:** Використовуючи Copilot Chat, виправити всі 5 проблем. Промпт:
```
Review this EF Core code for performance issues. 
Identify all N+1 queries, missing AsNoTracking, and unnecessary data loading.
Rewrite using a single optimized query with proper projection.
```

**Очікуваний результат:**
```csharp
return await _context.Orders
    .AsNoTracking()
    .Where(o => o.CreatedAt.Year == year)
    .Select(o => new OrderReportDto
    {
        OrderId = o.Id,
        CustomerName = o.Customer.FullName,
        LineCount = o.OrderLines.Count,
        Total = o.OrderLines.Sum(l => l.UnitPrice * l.Quantity)
    })
    .ToListAsync(ct);
```

---

### Lab I-2: TDD з Copilot — LoyaltyPoints (75 хв)
**Мета:** TDD flow — тести через Copilot → реалізація через Copilot

**Кроки:**

1. Написати промпт для тестів (без реалізації):
```
Generate comprehensive failing xUnit tests for ILoyaltyPointsCalculator:

Rules:
- 1 point per $10 spent (floor division)
- VIP customers (CustomerTier.Vip): 2x multiplier
- Birthday month: 3x multiplier (additive to VIP: VIP + Birthday = 6x)
- Orders below $5 earn 0 points
- Negative order amounts throw ArgumentOutOfRangeException

Use FluentAssertions. Name tests: Calculate_[Scenario]_Should[ExpectedResult]
Provide at least 15 test cases covering all rules and boundaries.
```

2. Переглянути тести — переконатися що вони перевіряють потрібне
3. Запустити — всі червоні ✓
4. Написати промпт для реалізації:
```
Implement LoyaltyPointsCalculator to satisfy all tests.
Make all tests green without modifying the tests.
```
5. Запустити — всі зелені ✓

---

### Lab I-3: Рефакторинг Adventure (60 хв)
**Мета:** Пройти повний цикл рефакторингу з Copilot

**Надати `LegacyInvoiceProcessor.cs` (~200 рядків)**:
- Без іменувань, магічні числа, змішана відповідальність
- SQL через конкатенацію рядків (SQL Injection)
- Sync IO з .Result
- Бойовий hardcoded connection string

**Чеклист рефакторингу:**
```markdown
- [ ] /explain — зрозуміти що робить код
- [ ] /tests — написати тести на поточну поведінку  
- [ ] Всі тести зелені ПЕРЕД тим як чіпати код
- [ ] Прибрати SQL Injection (EF Core / параметризований Dapper)
- [ ] Async/await + CancellationToken
- [ ] Прибрати hardcoded values
- [ ] Розділити на методи (SRP)
- [ ] Перейменувати (ROLE техніка Copilot)
- [ ] /doc — додати документацію
- [ ] Всі тести зелені ПІСЛЯ рефакторингу
```

---

## Advanced Labs — Рівень 3

### Lab A-1: DDD Aggregate Implementation (75 хв)
**Мета:** Реалізувати складний Aggregate з інваріантами

**Завдання:**

Реалізуйте `Invoice` Aggregate для системи білінгу:

```
Інваріанти:
1. Invoice.Total = сума InvoiceLines × (1 + TaxRate)
2. InvoiceLine не можна додати до оплаченого Invoice
3. Не можна виставити Invoice без хоча б одного рядка
4. DueDate повинна бути в майбутньому при створенні
5. Знижка не може перевищувати 30% від суми до податків

Статуси: Draft → Issued → PartiallyPaid → Paid | Overdue | Cancelled

Доменні події: InvoiceIssued, PaymentReceived, InvoiceOverdue, InvoiceCancelled

Value Objects: Money, TaxRate (percentage 0-100), InvoiceNumber (формат INV-YYYY-#####)
```

Використовувати Copilot для:
1. Проектування через Chat (запитати про інваріанти, події)
2. Генерації Value Objects
3. Генерації Aggregate з factory method
4. Генерації тестів для всіх інваріантів

---

### Lab A-2: Event-Driven Integration (90 хв)
**Мета:** Реалізувати повний event-driven flow з Testcontainers

**Сценарій:** `OrderPlaced` → `EmailNotificationService` відправляє листа

**Компоненти:**
1. Kafka consumer для `order.placed` topic
2. `OrderPlacedIntegrationEvent` десеріалізація
3. Виклик зовнішнього `EmailService` (HTTP, з retry)
4. Idempotent обробка (не відправляти листа двічі)
5. Інтеграційний тест: 
   - Testcontainers: Kafka + PostgreSQL + WireMock (для EmailService mock)
   - Publish event → wait → verify email sent once

---

### Lab A-3: Copilot Instructions Tuning (60 хв)
**Мета:** Налаштувати інструкції та виміряти покращення

**Завдання:**

1. **Baseline:** Попросити Copilot створити `ReportService`:
   ```
   Implement a ReportService for generating monthly sales reports
   ```
   Оцінити код за чеклистом (async, CT, Result<T>, logging, patterns)

2. **Написати інструкції:** Створити `.github/copilot-instructions.md` — мінімум 20 правил

3. **After:** Повторити той самий запит

4. **Порівняти:** До vs Після
   - Скількох правил дотримується код без інструкцій?
   - Скількох — з інструкціями?

5. **Ітерувати:** Покращити інструкції за результатами

---

## Challenger Tasks — для найшвидших

### Challenge 1: MCP Server за 2 години
```
Створіть MCP Server, який дає Copilot доступ до:
- GitHub Issues вашого репозиторію (через GitHub API)
- Останніх 10 комітів (git log)
- TODO/FIXME коментарів у коді (grep по проекту)

Бонус: додайте tool "create_issue" для створення Issues прямо з Copilot Chat
```

### Challenge 2: Observability Stack
```
Використовуючи тільки Copilot (один великий промпт для кожного файлу):
1. Docker Compose: веб-сервіс + PostgreSQL + Jaeger + Prometheus + Grafana
2. OpenTelemetry налаштування в .NET сервісі
3. Grafana dashboard JSON для ключових метрик
4. Alert rules в Prometheus
```

### Challenge 3: Performance Benchmark
```
Завдання: знайти оптимальний алгоритм для масової вставки 100K записів в PostgreSQL

Порівняти через Copilot-guided BenchmarkDotNet:
1. EF Core Add + SaveChanges (naive)
2. EF Core BulkInsert (Z.EntityFramework.Extensions)
3. Npgsql COPY protocol  
4. Dapper + TVP

Написати бенчмарк, запустити, порівняти, задокументувати за допомогою Copilot.
```

---

## Оціночні критерії для тренера

### Beginner — залік якщо:
- [ ] Студент прийняв та скоригував мінімум 20 пропозицій Copilot
- [ ] Може пояснити різницю між хорошим та поганим промптом
- [ ] Знає гарячі клавіші (Tab, Alt+], Escape, Ctrl+I)
- [ ] Розуміє концепцію галюцинацій та вміє перевіряти пропозиції

### Intermediate — залік якщо:
- [ ] Написав сервіс з EF Core без N+1 queries (з Copilot)
- [ ] Написав >10 unit тестів для існуючого коду
- [ ] Провів рефакторинг legacy з тестами як страховкою
- [ ] Налаштував `.github/copilot-instructions.md` та бачить різницю

### Advanced — залік якщо:
- [ ] Реалізував DDD Aggregate з інваріантами (тести зелені)
- [ ] Налаштував Outbox або Idempotent Consumer
- [ ] Провів архітектурну дискусію з Copilot Chat
- [ ] Може навчити колег (провести mini session на одну техніку)
