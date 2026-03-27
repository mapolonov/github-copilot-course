# Модуль 05: Тестування з Copilot — xUnit, FluentAssertions, FakeItEasy
## Рівень: Intermediate | Тривалість: 4 години

---

## Цілі модуля

- Ефективно генерувати unit тести з повним покриттям через Copilot
- Використовувати Copilot для написання інтеграційних тестів з Testcontainers
- Генерувати edge cases та negative path тести, які розробники часто пропускають
- Налаштувати `TestBase` та загальний test scaffolding через Copilot

---

## 1. Чому тестування — найкращий ROI від Copilot

### 1.1 Статистика

Тестування — область де Copilot дає **максимальний ROI**:
- Unit тести — високо повторюваний патерн (AAA)
- Хороше покриття в навчальних даних
- Детермінований output (немає бізнес-специфіки)
- Розробники ненавидять писати тести — Copilot це змінює

**Типове прискорення:** написання тестів прискорюється в 3-5x при правильному промпті.

### 1.2 Mindset: тести ЯК СПЕЦИФІКАЦІЯ

```
Традиційний підхід: пишемо код → пишемо тести (як перевірка)
Правильний підхід з Copilot: пишемо тести (як специфікацію) → Copilot пише код

1. /tests generate for IOrderService interface (ще немає реалізації)
2. Тести описують очікувану поведінку
3. Реалізація пишеться щоб тести проходили
→ TDD з Copilot на швидкості
```

---

## 2. Генерація unit тестів — техніки

### 2.1 Базовий промпт для повного покриття

```csharp
// Промпт в Copilot Chat:
/*
Generate comprehensive xUnit tests for #file:DiscountService.cs

Stack: xUnit, FluentAssertions, FakeItEasy, AutoFixture

For EACH public method generate:
1. Happy path test (valid input, expected result)
2. Boundary value tests (min/max values, empty collections)
3. Null/empty input tests (where applicable)
4. Business rule violation tests (each rule = separate test)
5. Exception path tests (repository failures, etc.)

Naming convention: MethodName_StateUnderTest_ExpectedBehavior
Pattern: AAA (// Arrange // Act // Assert sections)
Use FluentAssertions for all assertions (never Assert.Equal)
Use FakeItEasy for all dependencies (never Moq)
Use AutoFixture for test data generation where appropriate
*/
```

### 2.2 Структура тестового класу через Copilot

```csharp
// Промпт:
// xUnit test class for DiscountService with:
// - Constructor-based fixture setup (no [SetUp])
// - AutoFixture IFixture for test data
// - FakeItEasy fakes for IProductRepository, ICustomerRepository, ILogger
// - Helper method CreateSut() that creates DiscountService with all fakes
// - No static state

public class DiscountServiceTests
{
    private readonly IFixture _fixture;
    private readonly IProductRepository _productRepo;
    private readonly ICustomerRepository _customerRepo;
    private readonly ILogger<DiscountService> _logger;

    public DiscountServiceTests()
    {
        _fixture = new Fixture().Customize(new AutoFakeItEasyCustomization());
        _productRepo = A.Fake<IProductRepository>();
        _customerRepo = A.Fake<ICustomerRepository>();
        _logger = A.Fake<ILogger<DiscountService>>();
    }

    private DiscountService CreateSut() => 
        new(_productRepo, _customerRepo, _logger);

    // Copilot заповнить тестові методи
}
```

### 2.3 Генерація тестів для конкретного сценарію

```csharp
// Після написання цього заголовка → Copilot заповнює:

[Fact]
public async Task CalculateDiscountAsync_WhenCustomerIsVipAndOrderExceeds1000_ShouldApply20PercentDiscount()
{
    // Arrange
    var customer = _fixture.Build<Customer>()
        .With(c => c.Tier, CustomerTier.Vip)
        .Create();
    var order = _fixture.Build<Order>()
        .With(o => o.TotalAmount, 1500m)
        .With(o => o.CustomerId, customer.Id)
        .Create();
    
    A.CallTo(() => _customerRepo.GetByIdAsync(customer.Id, A<CancellationToken>._))
        .Returns(customer);

    var sut = CreateSut();

    // Act
    var result = await sut.CalculateDiscountAsync(order, CancellationToken.None);

    // Assert
    result.Should().BeSuccess();
    result.Value.DiscountPercentage.Should().Be(20);
    result.Value.DiscountAmount.Should().Be(300m);
}
```

### 2.4 Theory tests для граничных значений

```csharp
// Промпт:
// [Theory] tests for price tier discount calculation:
// Tier 1 (0-100): 0% discount
// Tier 2 (100-500): 5% discount  
// Tier 3 (500-1000): 10% discount
// Tier 4 (1000+): 15% discount
// Test boundary values: 99.99, 100, 100.01, 499.99, 500, 500.01, 999.99, 1000, 1000.01

[Theory]
[InlineData(99.99, 0)]
[InlineData(100, 5)]
[InlineData(100.01, 5)]
[InlineData(499.99, 5)]
[InlineData(500, 10)]
// Copilot продовжить...
public async Task CalculateDiscount_BoundaryValues_ShouldReturnCorrectTier(
    decimal orderAmount, int expectedDiscountPercent)
```

---

## 3. FluentAssertions — професійні assertions

### 3.1 Типові патерни, які генерує Copilot

```csharp
// Для колекцій:
result.Value.Should().NotBeEmpty()
    .And.HaveCount(3)
    .And.AllSatisfy(item => item.IsActive.Should().BeTrue())
    .And.BeInAscendingOrder(item => item.Name);

// Для виключень:
var act = () => sut.ProcessOrder(invalidOrder, CancellationToken.None);
await act.Should().ThrowAsync<ValidationException>()
    .WithMessage("*OrderLines*")
    .Where(ex => ex.Errors.Count == 2);

// Для Result<T>:
result.Should().BeSuccess()  // Кастомний extension
    .WithValue(v => v.OrderId.Should().BePositive());

// Для часових значень:
order.CreatedAt.Should().BeCloseTo(DateTimeOffset.UtcNow, TimeSpan.FromSeconds(5));

// Для об’єктів (структурне порівняння):
actual.Should().BeEquivalentTo(expected, options => 
    options.Excluding(x => x.CreatedAt)
           .Excluding(x => x.Id));
```

### 3.2 Кастомні extensions для доменних типів

```csharp
// Промпт:
//Custom FluentAssertions extension methods for Result<T> type:
// BeSuccess() - asserts IsSuccess == true
// BeFailure() - asserts IsFailure == true  
// WithError(Error expected) - chain after BeFailure, checks error code and message
// WithValue(Action<T> assertion) - chain after BeSuccess, provides value for further assertions

public static class ResultAssertionExtensions
{
    public static AndConstraint<ObjectAssertions> BeSuccess<T>(
        this ObjectAssertions assertions, string because = "", params object[] becauseArgs)
    {
        // Copilot заповнить
    }
}
```

---

## 4. FakeItEasy — просунуті патерни

### 4.1 Просунуті налаштування через Copilot

```csharp
// Налаштування послідовних викликів:
// First call returns Pending order, second call returns Approved order
A.CallTo(() => _repo.GetByIdAsync(orderId, A<CancellationToken>._))
    .ReturnsNextFromSequence(pendingOrder, approvedOrder);

// Налаштування з захопленням аргументів:
Order? capturedOrder = null;
A.CallTo(() => _repo.UpdateAsync(A<Order>._, A<CancellationToken>._))
    .Invokes((Order order, CancellationToken _) => capturedOrder = order)
    .Returns(Task.CompletedTask);

// Після act:
capturedOrder.Should().NotBeNull();
capturedOrder!.Status.Should().Be(OrderStatus.Approved);

// Перевірка що метод НЕ був викликаний:
A.CallTo(() => _emailService.SendAsync(A<EmailMessage>._, A<CancellationToken>._))
    .MustNotHaveHappened();

// Перевірка точної кількості викликів:
A.CallTo(() => _logger.Log(LogLevel.Warning, A<EventId>._, A<object>._, null, A<Func<object, Exception, string>>._))
    .MustHaveHappenedOnceExactly();
```

### 4.2 Fake для ILogger

```csharp
// Промпт:
// Helper method to verify ILogger calls with FakeItEasy:
// VerifyLoggedWarning(logger, message) - checks LogWarning was called with specific message
// VerifyLoggedError(logger, exception, message) - checks LogError with exception
// Works with generic ILogger<T>

public static class LoggerFakeExtensions
{
    public static void VerifyLoggedWarning<T>(
        this ILogger<T> logger, string containsMessage)
    {
        A.CallTo(logger)
            .Where(call => call.Method.Name == "Log" 
                && call.GetArgument<LogLevel>(0) == LogLevel.Warning
                && call.GetArgument<object>(2)!.ToString()!.Contains(containsMessage))
            .MustHaveHappenedOnceOrMore();
    }
}
```

---

## 5. Інтеграційні тести з Testcontainers

### 5.1 Базове налаштування з Copilot

```csharp
// Промпт:
// xUnit integration test base class using Testcontainers:
// - PostgreSQL container (pgvector image if needed) 
// - Start container once per test class (IClassFixture)
// - Create real DbContext with connection string from container
// - Run EF Core migrations before first test
// - Wrap each test in transaction, rollback after (IAsyncLifetime)
// - Expose DbContext and HttpClient (if WebApplicationFactory used)
// - Implement IAsyncLifetime for proper cleanup

public class IntegrationTestBase : IAsyncLifetime
{
    private readonly PostgreSqlContainer _postgres;
    protected AppDbContext DbContext { get; private set; } = null!;
    private IDbContextTransaction? _transaction;

    public IntegrationTestBase()
    {
        _postgres = new PostgreSqlBuilder()
            .WithImage("postgres:16-alpine")
            .WithDatabase("testdb")
            .WithUsername("test")
            .WithPassword("test")
            .Build();
    }

    public async Task InitializeAsync()
    {
        await _postgres.StartAsync();
        
        var options = new DbContextOptionsBuilder<AppDbContext>()
            .UseNpgsql(_postgres.GetConnectionString())
            .Options;
        
        DbContext = new AppDbContext(options);
        await DbContext.Database.MigrateAsync();
        
        _transaction = await DbContext.Database.BeginTransactionAsync();
    }

    public async Task DisposeAsync()
    {
        await _transaction?.RollbackAsync()!;
        await DbContext.DisposeAsync();
        await _postgres.DisposeAsync();
    }
}
```

### 5.2 API Integration Tests з WebApplicationFactory

```csharp
// Промпт:
// WebApplicationFactory<Program> for integration tests:
// Replace real DbContext with Testcontainers PostgreSQL
// Replace external HTTP dependencies with WireMock stubs
// Add test authentication (fake JWT middleware bypassing real auth)
// Provide typed HttpClient with base address
// Seed test data via method SeedAsync(IEnumerable<T> entities)

public class ApiTestFactory : WebApplicationFactory<Program>, IAsyncLifetime
{
    private readonly PostgreSqlContainer _postgres = new PostgreSqlBuilder()
        .WithImage("postgres:16-alpine")
        .Build();

    protected override void ConfigureWebHost(IWebHostBuilder builder)
    {
        builder.ConfigureTestServices(services =>
        {
            // Remove real DbContext
            var descriptor = services.SingleOrDefault(d => 
                d.ServiceType == typeof(DbContextOptions<AppDbContext>));
            if (descriptor != null) services.Remove(descriptor);

            // Add test DbContext with container connection
            services.AddDbContext<AppDbContext>(options =>
                options.UseNpgsql(_postgres.GetConnectionString()));

            // Replace external payment service
            services.AddScoped<IPaymentGateway, FakePaymentGateway>();
        });
    }

    public async Task InitializeAsync() => await _postgres.StartAsync();
    
    public new async Task DisposeAsync()
    {
        await base.DisposeAsync();
        await _postgres.DisposeAsync();
    }
}
```

---

## 6. Copilot для тест-сценариев которые разработчики пропускают

### 6.1 Промпт для edge case discovery

```
Review #file:OrderService.cs and suggest 10 edge cases that should be tested 
but are commonly missed by developers. Focus on:
- Concurrency scenarios
- Empty/null collections
- Decimal precision edge cases
- Timezone edge cases
- MaxValue/MinValue boundaries
- Idempotency violations
- State machine invalid transitions
```

### 6.2 Property-Based Testing з FsCheck

```csharp
// Промпт:
// FsCheck property-based tests for MoneyAmount value object:
// Property 1: Adding two positive amounts always results in larger amount
// Property 2: Subtraction is never negative if minuend >= subtrahend
// Property 3: Currency conversion is reversible (round-trip within 0.01 precision)
// Property 4: MoneyAmount.Zero + any amount = that amount (identity)
// Use Xunit.FsCheck integration

[Property]
public Property AddingTwoPositiveAmounts_AlwaysLargerThanEither(
    PositiveDecimal a, PositiveDecimal b)
{
    var amountA = MoneyAmount.Of(a.Get, Currency.USD);
    var amountB = MoneyAmount.Of(b.Get, Currency.USD);
    var sum = amountA + amountB;
    
    return (sum >= amountA && sum >= amountB).ToProperty();
}
```

---

## Практичне завдання (60 хвилин)

### Lab 5.1 — TDD з Copilot

**Завдання:** Реалізувати `LoyaltyPointsService`

1. Відкрийте Copilot Chat:
```
Generate failing xUnit tests for LoyaltyPointsService with these rules:
- 1 point per $10 spent, rounded down
- VIP customers earn 2x points
- Birthday month: 3x multiplier (stackable with VIP)
- Minimum order $5 to earn points
- Points expire after 12 months
Tests first — no implementation yet.
```

2. Запустіть тести — переконайтесь що всі **червоні**

3. Попросіть Copilot реалізувати `LoyaltyPointsService` щоб тести стали **зеленими**:
```
Now implement LoyaltyPointsService to make all tests pass.
Follow the constraints in the test names.
```

### Lab 5.2 — Інтеграційний тест

Додайте інтеграційний тест для ендпоінту `POST /api/orders`:
- Використовувати `ApiTestFactory`
- Засіяти тестові дані (продукти, клієнт)
- Перевірити HTTP статус, тіло відповіді, стан БД

---

## Підсумки модуля

- Copilot + тести = максимальний ROI, не нехтуйте цим
- `/tests` команда + детальний контекст = 80% покриття з першого промпту
- Edge cases промпт допомагає знайти те, що розробник забуває
- Testcontainers інтеграція — шаблонний код, ідеально для Copilot
- TDD з Copilot — потужна комбінація: тести як специфікація, код як реалізація

**← Попередній модуль:** [Модуль 04](../Module-04-DotNet-Patterns/lecture.md)  
**Наступний модуль →** [Модуль 06: Рефакторинг та документація](../Module-06-Refactoring/lecture.md)
