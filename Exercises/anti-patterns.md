# Антипатерни Copilot — Що НЕ треба робити
## Для .NET розробників | Розбір реальних помилок

---

> Ця сторінка — збірка типових пасток. Кожен антипатерн має пояснення чому так трапляється і як зробити правильно.

---

## Категорія 1: Прийняття без перевірки

### ❌ Антипатерн: "Tab-happy" — приймати все підряд

```csharp
// Copilot запропонував. Розробник натиснув Tab не читаючи.
public async Task<User?> GetUserAsync(int id)
{
    // ⚠️ Copilot вигадав метод SmartFetch який не існує в EF Core
    return await _context.Users.SmartFetch(id).FirstOrDefaultAsync();
    //                           ^^^^^^^^^^ НЕ ІСНУЄ
}
```

**Чому так виникає:** Пропозиція виглядає правдоподібно. LLM генерує плавний, "мовно правильний" код.

**Як правильно:**
```csharp
return await _context.Users.FirstOrDefaultAsync(u => u.Id == id, ct);
```

**Правило:** Незнайомий метод — завжди перевір через IntelliSense або `F12`. Якщо немає — галюцинація.

---

### ❌ Антипатерн: Приймати NuGet пакет за іменем без перевірки

```
// Copilot запропонував:
// Install-Package Microsoft.EntityFrameworkCore.BulkExtensions

// Насправді правильний пакет:
// Install-Package EFCore.BulkExtensions
// (від автора zzzprojects, не від Microsoft)
```

**Правило:** NuGet пакети — завжди перевіряй на nuget.org. Copilot плутає vendor-prefix та неофіційні пакети.

---

## Категорія 2: Проблеми безпеки

### ❌ Антипатерн: SQL через конкатенацію (SQL Injection)

```csharp
// Copilot іноді генерує це для legacy-стилю або Dapper:
var sql = "SELECT * FROM Users WHERE Username = '" + username + "'";
// ^ КРИТИЧНА ВРАЗЛИВІСТЬ якщо username = "' OR '1'='1"
```

**Як правильно — з Dapper:**
```csharp
var user = await connection.QuerySingleOrDefaultAsync<User>(
    "SELECT * FROM Users WHERE Username = @Username",
    new { Username = username }, cancellationToken: ct);
```

**Як правильно — з EF Core:**
```csharp
var user = await _context.Users
    .FirstOrDefaultAsync(u => u.Username == username, ct);
```

---

### ❌ Антипатерн: Hardcoded credentials в коді

```csharp
// Copilot "знає" паттерни з публічного коду (включаючи витоки):
private readonly string _apiKey = "sk-prod-AbCdEf123456";
private readonly string _connString = "Server=10.0.0.1;Password=Admin@123";
```

**Як правильно:**
```csharp
// appsettings.json + IConfiguration або IOptions<T>
private readonly string _apiKey;

public PaymentService(IConfiguration config)
{
    _apiKey = config["Payment:ApiKey"] 
        ?? throw new InvalidOperationException("Payment:ApiKey not configured");
}
```

---

### ❌ Антипатерн: Логування чутливих даних

```csharp
// Copilot може включити в лог password або token якщо вони є у параметрах:
_logger.LogInformation("Login attempt: user={User}, password={Password}", 
    username, password); // ❌ password в логах!
```

**Як правильно:**
```csharp
_logger.LogInformation("Login attempt for user {Username}", username);
// Password ніколи не логується
```

---

## Категорія 3: Async/Threading пастки

### ❌ Антипатерн: Блокування async коду (.Result / .Wait)

```csharp
// Copilot іноді пропонує це в синхронних методах або конструкторах:
public User GetCurrentUser()
{
    return _userService.GetCurrentUserAsync().Result; // DEADLOCK у ASP.NET Core!
}

public MyService(IUserService userService)
{
    _currentUser = userService.GetCurrentUserAsync().GetAwaiter().GetResult(); // ❌
}
```

**Як правильно:** Зроби метод async або використовуй lazy initialization:
```csharp
public async Task<User> GetCurrentUserAsync(CancellationToken ct) =>
    await _userService.GetCurrentUserAsync(ct);
```

---

### ❌ Антипатерн: Втрата CancellationToken у ланцюжку

```csharp
// Copilot генерував метод з ct, але "забув" передати його всередину:
public async Task ProcessOrderAsync(int orderId, CancellationToken ct)
{
    var order = await _context.Orders.FindAsync(orderId); // ❌ ct не передано!
    var customer = await _customerService.GetAsync(order.CustomerId); // ❌ ct не передано!
    await _emailService.SendAsync(customer.Email, "Order confirmed"); // ❌
}
```

**Як правильно:**
```csharp
public async Task ProcessOrderAsync(int orderId, CancellationToken ct)
{
    var order = await _context.Orders.FindAsync([orderId], ct);
    var customer = await _customerService.GetAsync(order!.CustomerId, ct);
    await _emailService.SendAsync(customer.Email, "Order confirmed", ct);
}
```

**Промпт-захист:** Завжди додавай у промпт: `"propagate ct to ALL async calls"`.

---

### ❌ Антипатерн: async void

```csharp
// Copilot може запропонувати async void для event-like методів:
public async void OnOrderPlaced(OrderPlacedEvent evt)
{
    await _notificationService.NotifyAsync(evt.OrderId); // Виняток тут — непомічений!
}
```

**Чому небезпечно:** Виняток в `async void` крашить весь процес без можливості catch.

**Як правильно:**
```csharp
// Тільки для справжніх event handlers (WinForms, кнопки UI):
// async void OnButtonClick(object sender, EventArgs e) — ОК
// Для бізнес-логіки:
public async Task HandleOrderPlacedAsync(OrderPlacedEvent evt, CancellationToken ct)
{
    await _notificationService.NotifyAsync(evt.OrderId, ct);
}
```

---

## Категорія 4: Entity Framework Core помилки

### ❌ Антипатерн: N+1 query за замовчуванням

```csharp
// Copilot часто генерує це якщо не вказати інакше:
var orders = await _context.Orders.ToListAsync(ct);
foreach (var order in orders)
{
    // N окремих запитів до БД!
    Console.WriteLine(order.Customer.Name); 
}
```

**Як правильно:**
```csharp
// Проєкція: завантажуємо тільки потрібні дані
var orders = await _context.Orders
    .Select(o => new { o.Id, CustomerName = o.Customer.Name })
    .AsNoTracking()
    .ToListAsync(ct);

// Або Include якщо потрібна вся сутність:
var orders = await _context.Orders
    .Include(o => o.Customer)
    .AsNoTracking()
    .ToListAsync(ct);
```

---

### ❌ Антипатерн: Трекінг для read-only запитів

```csharp
// Copilot часто забуває AsNoTracking для read-only сценаріїв:
var products = await _context.Products
    .Where(p => p.IsActive)
    .ToListAsync(ct); // EF Core трекує всі об'єкти — зайве навантаження
```

**Як правильно:**
```csharp
var products = await _context.Products
    .Where(p => p.IsActive)
    .AsNoTracking() // For read-only: ~30% швидше, менше пам'яті
    .ToListAsync(ct);
```

---

### ❌ Антипатерн: AutoMapper замість проєкції (витік даних)

```csharp
// Copilot може запропонувати:
var orders = await _context.Orders.ToListAsync(ct);        // Завантажує ВСЕ
return _mapper.Map<List<OrderDto>>(orders);                // Потім маппить у пам'яті
// ❌ Якщо Order має 50 полів, а OrderDto — 5, завантажили зайві 45 полів
```

**Як правильно:**
```csharp
return await _context.Orders
    .Select(o => new OrderDto                              // SQL проєкція
    {
        Id = o.Id,
        CustomerName = o.Customer.Name,
        Total = o.OrderLines.Sum(l => l.UnitPrice * l.Quantity)
    })
    .AsNoTracking()
    .ToListAsync(ct);
```

---

## Категорія 5: Архітектурні помилки

### ❌ Антипатерн: Бізнес-логіка в Controller / Endpoint

```csharp
// Copilot дуже часто кладе логіку прямо в контролер:
[HttpPost]
public async Task<IActionResult> CreateOrder(CreateOrderRequest req, CancellationToken ct)
{
    // ❌ Бізнес-логіка в контролері
    if (!await _context.Customers.AnyAsync(c => c.Id == req.CustomerId, ct))
        return BadRequest("Customer not found");
    
    var total = req.Lines.Sum(l => l.Price * l.Quantity);
    if (total > 100_000)
        return BadRequest("Order exceeds limit");
    
    var order = new Order { CustomerId = req.CustomerId, Total = total };
    _context.Orders.Add(order);
    await _context.SaveChangesAsync(ct);
    return Ok(order.Id);
}
```

**Як правильно:**
```csharp
[HttpPost]
public async Task<IActionResult> CreateOrder(CreateOrderRequest req, CancellationToken ct)
{
    // Контролер — тільки маршрутизація + маппінг
    var result = await _mediator.Send(new CreateOrderCommand(req.CustomerId, req.Lines), ct);
    return result.IsSuccess ? Ok(result.Value) : result.ToProblemDetails();
}
```

---

### ❌ Антипатерн: Infrastructure у Domain

```csharp
// Copilot може запропонувати EF Core у доменній сутності:
public class Order
{
    [Required]                          // ❌ Data Annotations = EF/ASP.NET залежність
    public string CustomerName { get; set; }
    
    public async Task Save(DbContext ctx) // ❌ Repository у агрегаті — порушення DDD
    {
        ctx.Orders.Add(this);
        await ctx.SaveChangesAsync();
    }
}
```

**Як правильно:**
```csharp
// Доменна сутність — тільки бізнес-логіка, без зовнішніх залежностей
public class Order : AggregateRoot<Guid>
{
    public string CustomerName { get; private set; } = null!;
    
    // Інваріанти через конструктор або фабричний метод
    public static Order Create(string customerName)
    {
        ArgumentException.ThrowIfNullOrWhiteSpace(customerName);
        var order = new Order { CustomerName = customerName };
        order.RaiseDomainEvent(new OrderCreatedEvent(order.Id));
        return order;
    }
}
```

---

## Категорія 6: Проблеми з промптами

### ❌ Антипатерн: Занадто розмитий промпт → generic-код

```csharp
// Промпт: "add error handling"
// Результат від Copilot:
try
{
    await ProcessOrderAsync(orderId, ct);
}
catch (Exception ex)
{
    _logger.LogError(ex, "Error");
    throw;
}
// ↑ Нічого не вирішує. Swallows nothing, але й не допомагає.
```

**Правильний підхід:** Вказати конкретні типи помилок і реакцію на кожну:
```
"Handle exceptions:
 - NotFoundException → return 404 ProblemDetails
 - ValidationException → return 422 with errors list  
 - DbUpdateConcurrencyException → retry once, then 409
 - All others → log as Error with OrderId, return 500"
```

---

### ❌ Антипатерн: Приймати перший результат без ітерацій

```
Розробник попросив Copilot згенерувати метод.
Перший результат — 70% правильний.
Розробник сказав "досить добре" і пішов далі.

Правильно: ітерувати!
"The retry logic uses Thread.Sleep — change to Task.Delay with CancellationToken"
"The catch block is too broad — handle only HttpRequestException and TimeoutException"
"Add structured logging with the orderId parameter"
```

**Правило:** Перший результат — чорновик. Уточни 2-3 рази — і отримаєш production-ready код.

---

## Швидкий чеклист антипатернів

```
БЕЗПЕКА:
❌ SQL конкатенація з user input
❌ Hardcoded credentials / API keys
❌ Логування паролів, токенів, PII

ASYNC:
❌ .Result / .Wait() / GetAwaiter().GetResult()
❌ async void (крім UI event handlers)
❌ Не передаєш CancellationToken далі

EF CORE:
❌ Навігаційні властивості без Include → N+1
❌ Без AsNoTracking для read-only запитів
❌ Завантаження повної сутності коли потрібні 3 поля

АРХІТЕКТУРА:
❌ Бізнес-логіка в Controller / Endpoint
❌ EF/HTTP залежності в Domain шарі
❌ [Required] атрибути в доменних сутностях

COPILOT USAGE:
❌ Приймати метод без перевірки що він існує
❌ Не ітерувати після першого результату
❌ Розмиті промпти без конкретних вимог
```
