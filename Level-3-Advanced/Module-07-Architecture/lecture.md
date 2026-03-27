# Модуль 07: Архітектурні патерни з Copilot — Clean Arch, DDD, CQRS
## Рівень: Advanced | Тривалість: 4 години

---

## Цілі модуля

- Використовувати Copilot як архітектурного помічника, а не лише генератора коду
- Реалізовувати DDD Aggregates, Value Objects, Domain Events через Copilot
- Будувати Clean Architecture проєкти з Copilot від структури до коду
- Генерувати Infrastructure-level код (persistence, messaging) без domain leakage

---

## 1. Архітектурні обговорення з Copilot

### 1.1 Copilot як архітектурний радник

Copilot Chat — потужний інструмент для архітектурних дискусій:

```
You are a senior software architect specializing in DDD and Clean Architecture.

I'm designing an e-commerce platform. We have these bounded contexts:
- Order Management (Orders, OrderLines, Fulfillment)
- Catalog (Products, Categories, Pricing)
- Customer (Profiles, Addresses, Loyalty)
- Payment (Transactions, Refunds)

Questions:
1. Where should the Cart concept live? It spans Catalog and Order contexts.
2. How should Order context know about current prices without coupling to Catalog?
3. What's the integration pattern between Order and Payment contexts?
4. Should Loyalty points calculation be in Customer or Order context?

Use DDD ubiquitous language. Suggest context map patterns (Shared Kernel, ACL, etc).
```

### 1.2 Оцінка архітектурних рішень

```
Analyze this dependency: OrderService directly calls IPaymentGateway (Stripe SDK).

Evaluate:
1. What DDD / Clean Architecture principles does this violate?
2. What are the operational risks? (availability, testability, deployment coupling)
3. Suggest an anti-corruption layer implementation
4. Suggest an alternative using the Ports & Adapters pattern
5. What would be the migration path from the current approach?

#file:OrderService.cs #file:IPaymentGateway.cs
```

---

## 2. Domain-Driven Design з Copilot

### 2.1 Aggregate Root з інваріантами

```csharp
// Промпт для складного Aggregate:
/*
Implement Order Aggregate Root following DDD:

INVARIANTS (must ALWAYS be maintained):
1. Order can only be placed if it has at least one OrderLine
2. Total amount cannot exceed customer's credit limit (checked at creation)  
3. OrderLines cannot be modified after Order is Confirmed
4. Order cannot be Cancelled if it's in Shipped or Delivered status
5. TotalAmount = Sum of (UnitPrice × Quantity × (1 - DiscountRate)) for all lines

LIFECYCLE: Draft → Placed → Confirmed → Shipped → Delivered | Cancelled

DOMAIN EVENTS (raise, not publish directly):
- OrderPlaced: when transitioning Draft → Placed
- OrderCancelled: when cancelled (include reason)
- OrderLineAdded: when line added to Draft order

RULES:
- Use private setters for all properties
- State transitions via domain methods (not property setters)
- Throw DomainException for invariant violations (never return null for errors)
- Raise domain events via protected RaiseDomainEvent(IDomainEvent) on Entity base class
- No external dependencies (NO EF, NO repositories, NO services in this class)
*/
public class Order : AggregateRoot<Guid>
```

### 2.2 Value Objects

```csharp
// Промпт для Value Object:
/*
Implement Money value object:
- Properties: Amount (decimal), Currency (Currency enum: USD, EUR, GBP, UAH)
- Immutable (record or readonly struct)
- Equality based on both Amount and Currency
- Operators: +, -, * (by decimal multiplier), / (by decimal)
- Conversion: ToDisplayString() "USD 1,234.56"
- Validation: Amount cannot be negative, Currency must be valid enum
- Implicit conversion from (decimal amount, Currency currency) tuple
- Static factory: Money.Of(decimal amount, Currency currency)
- Money.Zero(Currency) static factory

Throw MoneyDomainException for:
- Different currencies in arithmetic operations
- Negative amount result from subtraction
- Division by zero
*/
public readonly record struct Money
```

### 2.3 Доменні події та обробники

```csharp
// Промпт для Domain Event infrastructure:
/*
Implement domain event dispatching infrastructure:
1. IDomainEvent marker interface (with OccurredAt DateTimeOffset)
2. Entity base class: stores List<IDomainEvent>, RaiseDomainEvent(event) method
   GetDomainEvents() IReadOnlyList, ClearDomainEvents()
3. IDomainEventDispatcher with DispatchAndClearEvents(IEnumerable<Entity>) 
4. MediatRDomainEventDispatcher implementation:
   - Iterates entities, collects domain events
   - Publishes each as MediatR INotification
   - Clears events after publishing
5. EF Core SaveChanges override:
   - Before SaveChanges: collect all tracker entities that have events
   - After SaveChanges: dispatch collected events
   - Order: persist first, then event (no saga/outbox here — simple dispatch)
*/
public abstract class Entity<TId>
```

---

## 3. Clean Architecture — генерація всіх шарів

### 3.1 Повний шар Application за один промпт

```csharp
// Промпт для Application Use Case:
/*
Generate complete Application layer for "CreateOrder" use case:

COMMAND: CreateOrderCommand
- CustomerId (Guid)
- Lines: IReadOnlyList<CreateOrderLineDto> (ProductId: Guid, Quantity: int)
- Note: UnitPrice will be fetched from Catalog, NOT from request (price manipulation protection)

VALIDATOR (FluentValidation):
- CustomerId: not empty
- Lines: not empty, max 50 items, no duplicate ProductIds
- Each line Quantity: 1 to 1000

HANDLER (IRequestHandler<CreateOrderCommand, Result<Guid>>):
Steps:
1. Validate customer exists via ICustomerRepository → Result.Failure if not found
2. Fetch current prices via ICatalogService.GetCurrentPricesAsync(productIds) 
   → Result.Failure if any product not found
3. Check customer credit limit via ICustomerRepository.GetCreditLimitAsync
   → Result.Failure if total exceeds limit
4. Create Order aggregate via Order.Create(customerId, lines, prices) (domain factory)
   → Result.Failure if domain exception
5. Persist via IOrderRepository.AddAsync
6. Dispatch domain events via IDomainEventDispatcher
7. Return Result<Guid>.Success(order.Id)

Dependencies: IOrderRepository, ICustomerRepository, ICatalogService, 
              IDomainEventDispatcher, ILogger

All external calls use CancellationToken.
*/
```

### 3.2 Infrastructure шар — persistence mapping

```csharp
// Промпт для EF Core конфігурації з DDD:
/*
EF Core IEntityTypeConfiguration for Order aggregate:
- Table: "orders" (snake_case)
- OrderId: Guid, clustered PK + separate non-clustered UNIQUE on (CustomerId, OrderNumber)
- Status: stored as string (not int), max length 20
- Money TotalAmount: owned entity → columns: total_amount_amount (decimal 18,4), 
  total_amount_currency (varchar 3)
- OrderLines: owned collection → table "order_lines", FK to OrderId
  - OrderLine: ProductId, Quantity, UnitPrice (Money, same owned config), DiscountRate
- DomainEvents: IGNORED (not persisted)
- All navigation properties private (DDD encapsulation)
- No lazy loading (explicit Include required)
- RowVersion for optimistic concurrency on Order
*/
public class OrderConfiguration : IEntityTypeConfiguration<Order>
```

---

## 4. Advanced CQRS Patterns

### 4.1 Read side з окремими read models

```csharp
// Промпт:
/*
Implement a highly optimized Read Side for Order queries:

Problem: Using EF Core with full Aggregate loading for reads is too slow.
Solution: Separate read models with raw SQL / Dapper for queries.

1. IOrderReadRepository interface:
   - GetOrderDetailsAsync(Guid orderId) → OrderDetailsReadModel?
   - GetOrdersPagedAsync(OrderListFilter filter) → PagedResult<OrderSummaryReadModel>
   - GetCustomerOrderHistoryAsync(Guid customerId, int months) → IReadOnlyList<OrderHistoryItem>

2. DapperOrderReadRepository implementation:
   - Constructor: accepts IDbConnectionFactory (not DbContext)
   - GetOrderDetailsAsync: JOIN orders + order_lines + products + customers in one query
   - GetOrdersPagedAsync: dynamic filter building (Dapper with params, no string concat!)
   - Use async Dapper methods, pass CancellationToken via WithCancellation()

3. OrderDetailsReadModel record: all fields flattened (no nested objects in read model)
*/
```

### 4.2 Eventual Consistency та Projections

```csharp
// Промпт:
/*
Implement an event-driven projection for CustomerOrderStats:

CustomerOrderStats read model:
- CustomerId, TotalOrders, TotalSpent (Money), LastOrderDate, AverageOrderValue (Money)
- Stored in separate "customer_order_stats" table

Event handlers (INotificationHandler<T>):
1. OrderPlacedEventHandler → increment TotalOrders, add to TotalSpent, update LastOrderDate
2. OrderCancelledEventHandler → decrement TotalOrders, subtract from TotalSpent
3. Handler is idempotent: use ProcessedEvents table to skip duplicates
   - Check: if EventId already in ProcessedEvents → skip
   - After processing: insert EventId into ProcessedEvents in same transaction

Use EF Core, wrap in DbTransaction.
Upsert stats using ExecuteUpdateAsync or load-update-save (handle concurrency with retry).
*/
```

---

## 5. Vertical Slice Architecture з Copilot

### 5.1 Feature Slice замість Layer Structure

```
@workspace What's your opinion on Vertical Slice Architecture vs Clean Architecture 
for this project? Consider:
- Team size: 8 developers
- Domain complexity: https://... (опишіть домен)
- Current pain points: cross-layer changes for single features take hours
- Current benefits: clear separation of persistence and business logic

Compare: folder organization, testability, onboarding time, coupling.
Give concrete recommendation with reasoning.
```

### 5.2 Feature через Copilot Edits

```
Create a complete "GetProductReviews" vertical slice:
Files to generate:
1. Features/Products/GetReviews/GetProductReviewsEndpoint.cs (Minimal API)
2. Features/Products/GetReviews/GetProductReviewsQuery.cs (MediatR IRequest)
3. Features/Products/GetReviews/GetProductReviewsHandler.cs 
4. Features/Products/GetReviews/ProductReviewDto.cs
5. Features/Products/GetReviews/GetProductReviewsTests.cs (xUnit integration test)

All in Feature-based folder structure.
Endpoint: GET /api/products/{productId}/reviews?page=1&pageSize=20&sort=latest
Include paging, validation, sorted by date desc.
```

---

## Практичне завдання (75 хвилин)

### Lab 7.1 — Домен з нуля

**Завдання:** Спроєктувати та реалізувати `Inventory` Bounded Context

1. Обговоріть з Copilot межі контексту:
```
Help me design an Inventory bounded context for an e-commerce platform.
Responsibilities: track stock levels, reserve stock for orders, handle returns.
What are the aggregates? What are the invariants? 
What events does this context publish? What events does it subscribe to?
```

2. Реалізуйте `StockItem` агрегат з інваріантами:
- Stock не можна резервувати більше ніж Available
- Reserved + Available = Total завжди
- Reservation автоматично звільняється якщо Order скасовано (через подію)

3. Напишіть тести для всіх інваріантів

### Lab 7.2 — Архітектурний рев'ю

Використовуйте Copilot для аналізу існуючого проєкту команди:
```
@workspace Analyze the current architecture and identify:
1. Any domain logic leaking into Infrastructure or API layers
2. Missing abstraction layers
3. Circular dependencies between projects
4. Suggest a migration path to Clean Architecture without big bang rewrite
```

---

## Підсумки модуля

- Copilot як архітектурний радник — недооцінений сценарій, реально допомагає
- DDD агрегати: детальний промпт з інваріантами та життєвим циклом = 85% production-ready коду
- Separate Read Models через Dapper — Copilot добре генерує, набагато швидше ніж вручну
- Vertical Slice + Copilot Edits = feature за 30 хвилин замість 2 годин

**← Попередній модуль:** [Модуль 06](../../Level-2-Intermediate/Module-06-Refactoring/lecture.md)  
**Наступний модуль →** [Модуль 08: Розподілені системи](../Module-08-Distributed/lecture.md)
