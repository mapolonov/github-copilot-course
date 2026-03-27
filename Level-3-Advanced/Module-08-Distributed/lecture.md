# Модуль 08: Розподілені системи з Copilot — Kafka, Outbox, Resilience
## Рівень: Advanced | Тривалість: 4 години

---

## Цілі модуля

- Генерувати production-ready Kafka consumers/producers через Copilot
- Реалізувати Outbox Pattern для надійної доставки подій
- Використовувати Copilot для налаштування Polly (Retry, Circuit Breaker, Bulkhead)
- Проєктувати Saga/Process Manager за допомогою Copilot

---

## 1. Kafka з .NET через Copilot

### 1.1 Producer з ідемпотентністю

```csharp
// Промпт:
/*
Implement Kafka producer for domain events:

ProducerConfig requirements:
- Idempotent producer (EnableIdempotence = true)
- Acks = All (wait for all ISR replicas)
- MaxInFlight = 1 (strict ordering per partition)
- RetryBackoff = 100ms, MessageSendMaxRetries = 3
- CompressionType = Snappy

IEventPublisher interface:
- PublishAsync<T>(T domainEvent, CancellationToken ct) where T : IDomainEvent
- PublishBatchAsync<T>(IEnumerable<T> events, CancellationToken ct)

KafkaEventPublisher implementation:
- Topic naming: use ITopicRegistry that maps event type to topic name
- Message key: domainEvent.AggregateId.ToString() (ordering by aggregate)
- Message value: serialize with System.Text.Json
- Headers: X-Event-Type, X-Event-Id, X-Correlation-Id, X-Occurred-At
- On Kafka error: wrap in EventPublishingException with original KafkaException
- Producer is singleton (expensive to create), dispose on IHostedService stop
- Log each published event with correlation id and topic
*/
public class KafkaEventPublisher : IEventPublisher, IAsyncDisposable
```

### 1.2 Consumer з обробкою помилок

```csharp
// Промпт:
/*
Implement Kafka consumer IHostedService:

ConsumerConfig:
- GroupId from configuration
- AutoOffsetReset = Earliest (for new groups)
- EnableAutoCommit = false (manual commit after processing)
- MaxPollIntervalMs = 300000
- SessionTimeoutMs = 30000

Consuming pattern:
- Consume with 200ms timeout in tight loop
- Deserialize message value to specific event type based on X-Event-Type header
- Resolve IEventHandler<TEvent> from IServiceScope (not root container!)
- On successful processing: commit offset
- On handling exception:
  - Retry up to 3 times with 1s delay (in-process retry, not requeue)
  - After max retries: publish to Dead Letter Topic (same topic + ".dlq" suffix)
  - Commit offset even on DLQ (prevent poison pill blocking)
  - Log error with event headers for tracing
- Graceful shutdown: stop consuming when CancellationToken cancelled,
  wait for in-progress message to complete (max 30 seconds)
- Log: consumer group, topic, partition, offset on each consumed message
*/
public class KafkaConsumerService : BackgroundService
```

### 1.3 Обробка Dead Letter Queue

```csharp
// Промпт:
/*
Implement DLQ processing service:
- Reads from all DLQ topics (pattern: "*.dlq")
- Stores failed messages in database table "failed_messages":
  topic, partition, offset, event_type, payload (json), error_message, 
  failed_at, retry_count, next_retry_at
- Scheduled retry: every 5 minutes try to reprocess messages where 
  next_retry_at <= UtcNow and retry_count < MaxRetries (10)
- Exponential backoff: next_retry_at = UtcNow + 2^retry_count minutes
- After MaxRetries: mark as permanently_failed, alert via IAlertingService
- Manual retry endpoint: POST /admin/dlq/{messageId}/retry (with admin auth)
*/
```

---

## 2. Outbox Pattern — надійна доставка подій

### 2.1 Повна реалізація Outbox

```csharp
// Промпт:
/*
Implement Transactional Outbox Pattern for reliable event delivery:

OUTBOX TABLE (EF Core entity):
- Id (Guid, PK), EventType, EventPayload (json), AggregateId, AggregateType
- Status: Pending | Processing | Sent | Failed
- CreatedAt, ProcessedAt, RetryCount, Error
- Topic (Kafka topic to publish to)

WRITE SIDE (під час SaveChanges):
OutboxInterceptor : SaveChangesInterceptor
- BeforeSaveChanges: find all entities with domain events (implement IDomainEventSource)
- For each domain event: serialize + create OutboxMessage (same DbContext transaction)
- Clear domain events from entities after creating outbox messages
- This ensures: domain state + outbox message are committed atomically

RELAY PROCESSOR (BackgroundService):
- Every 5 seconds: SELECT TOP 50 messages WHERE Status = Pending ORDER BY CreatedAt
- Use SELECT FOR UPDATE SKIP LOCKED (EF Core raw SQL for PostgreSQL)
  → PostgreSQL: "SELECT ... FOR UPDATE SKIP LOCKED"
  → This allows multiple processor instances without duplicates
- Set Status = Processing (optimistic lock with RowVersion)
- Publish to Kafka via IEventPublisher
- On success: Status = Sent, ProcessedAt = UtcNow
- On failure: Status = Failed, RetryCount++, Error = exception message
  If RetryCount < 5: Status back to Pending, retry after 2^RetryCount minutes
- Transaction per batch item (not per batch)
*/

public class OutboxMessage
{
    public Guid Id { get; private set; }
    public string EventType { get; private set; } = null!;
    // ...
}
```

### 2.2 Ідемпотентні Consumers

```csharp
// Промпт:
/*
Implement idempotent event handler decorator:

IdempotentEventHandler<TEvent> wraps any IEventHandler<TEvent>:
1. Before handling: check "processed_events" table for MessageId (from Kafka header X-Event-Id)
2. If already processed: skip silently (log at Debug level)
3. If not processed:
   a. Begin transaction
   b. Call inner handler
   c. Insert into processed_events (MessageId, EventType, ProcessedAt, ConsumerGroup)
   d. Commit transaction
4. Handle unique constraint violation (race condition): treat as already-processed, skip

"processed_events" table partitioned by ProcessedAt (monthly), 
auto-delete records older than 30 days.

Register via DI: services.Decorate<IEventHandler<TEvent>, IdempotentEventHandler<TEvent>>()
(using Scrutor)
*/
```

---

## 3. Resilience з Polly v8

### 3.1 Resilience Pipeline через Copilot

```csharp
// Промпт:
/*
Build a Polly v8 resilience pipeline for calling external Payment API:

Pipeline (in this order):
1. Timeout: 10 seconds total (TimeoutResilientStrategy)

2. Retry: 
   - Max 3 retries
   - Exponential backoff with jitter: wait = 2^attempt seconds + random 0-500ms
   - Retry only on HttpRequestException and TaskCanceledException (not PaymentDeclinedException!)
   - Log each retry attempt with attempt number and delay

3. Circuit Breaker:
   - Open after 5 failures in 30-second sampling duration
   - Half-open after 60 seconds (1 probe request)
   - On Open: throw BrokenCircuitException immediately (do not call downstream)
   - Log state changes (Closed→Open, Open→HalfOpen, HalfOpen→Closed)

4. Bulkhead (Concurrency Limiter):
   - Max 10 concurrent calls to payment API
   - Queue up to 20 additional requests
   - Reject with BulkheadRejectedException if queue full

Register as named pipeline "payment-api" via IServiceCollection.
Inject via ResiliencePipelineProvider<string>.GetPipeline("payment-api")
*/
public static class PaymentResilienceExtensions
```

### 3.2 Health Checks з Circuit Breaker status

```csharp
// Промпт:
/*
ASP.NET Core Health Check that reports circuit breaker states:

CircuitBreakerHealthCheck : IHealthCheck
- Inject ResiliencePipelineProvider<string>
- For each registered circuit breaker (by convention, key ending in "-cb"):
  Check state: Closed = Healthy, HalfOpen = Degraded, Open = Unhealthy
- Return aggregate health with data dictionary: circuit_name → state
- Tag: "circuit_breakers" (for separate health endpoint)
  
Register: /health/ready — all checks
         /health/circuit-breakers — only circuit breaker checks
*/
```

---

## 4. Distributed Tracing та Observability

### 4.1 Налаштування OpenTelemetry через Copilot

```csharp
// Промпт:
/*
Set up OpenTelemetry for .NET 8 microservice:

Tracing:
- AspNetCore instrumentation (incoming HTTP)
- HttpClient instrumentation (outgoing HTTP)  
- EntityFrameworkCore instrumentation (DB queries)
- Custom ActivitySource "MyService.BusinessOperations"
- Export to OTLP (Jaeger or Zipkin based on config)
- Propagate W3C TraceContext + Baggage headers

Metrics:
- Runtime metrics (GC, thread pool, memory)
- ASP.NET Core metrics (request duration, active requests)
- Custom meters: orders_created_total, payment_processing_duration (histogram)
- Export to Prometheus (/metrics endpoint via prometheus-net.AspNetCore)

Logging:
- Serilog with OpenTelemetry sink
- Enrich logs with TraceId, SpanId (correlation with traces)
- Structured log to OTLP

Register in Program.cs with builder.Services.AddOpenTelemetry()
All endpoints configurable via environment variables (OTEL_EXPORTER_OTLP_ENDPOINT etc.)
*/
```

### 4.2 Custom Activity для бізнес-операцій

```csharp
// Промпт:
/*
Create distributed tracing for Order Processing Domain Activity:

OrderProcessingActivitySource:
- ActivitySource name: "ECommerce.OrderProcessing"
- StartOrderPlacement(Guid orderId, Guid customerId) → Activity
  Tags: order.id, customer.id, operation = "place_order"
- StartPaymentProcessing(Guid orderId, decimal amount, string currency) → Activity
  Tags: order.id, payment.amount, payment.currency, operation = "process_payment"
- RecordOrderError(Activity activity, Exception ex, string operation)
  Sets Status = Error, records exception event

Usage in handler:
using var activity = _activitySource.StartOrderPlacement(orderId, customerId);
try { ... activity?.SetTag("order.lines_count", linesCount); ... }
catch (Exception ex) { _activitySource.RecordOrderError(activity, ex, "place_order"); throw; }
*/
```

---

## 5. Saga Pattern

### 5.1 Process Manager для Order Fulfillment

```csharp
// Промпт:
/*
Implement a Process Manager (Orchestration Saga) for Order Fulfillment:

SAGA: OrderFulfillmentSaga
Triggered by: OrderPlacedEvent
Steps:
1. ReserveInventory → publishes ReserveInventoryCommand
   On success (InventoryReservedEvent): proceed to step 2
   On failure (InsufficientInventoryEvent): compensate → CancelOrderCommand → end

2. ProcessPayment → publishes ProcessPaymentCommand  
   On success (PaymentProcessedEvent): proceed to step 3
   On failure (PaymentFailedEvent): compensate → ReleaseInventoryCommand → CancelOrderCommand → end

3. CreateShipment → publishes CreateShipmentCommand
   On success (ShipmentCreatedEvent): complete saga
   On failure: compensate → RefundPaymentCommand → ReleaseInventoryCommand → CancelOrderCommand

IMPLEMENTATION:
- State stored in DB: SagaId (=OrderId), CurrentStep, Status, StartedAt, CompletedAt
- Each event handler: load saga state, validate current step matches, transition state
- Timeouts: if step not completed in 5 minutes, trigger timeout handler (compensation)
- Idempotent: re-processing same event must not re-execute step
- Use MediatR for command dispatch within saga

Use MassTransit Saga OR custom state machine (choose based on team convention).
*/
```

---

## Практичне завдання (75 хвилин)

### Lab 8.1 — Event-Driven Feature

Реалізуйте повний event-driven флоу для "Order Placed → Inventory Reserved":

1. Використовуючи Copilot, реалізуйте `OrderPlacedEventHandler` в Inventory context:
   - Отримує `OrderPlacedIntegrationEvent` з Kafka
   - Резервує stock для кожної позиції
   - Публікує `InventoryReservedEvent` або `InsufficientInventoryEvent`
   - Ідемпотентен (повторна обробка не порушує консистентність)

2. Додайте Polly retry + circuit breaker для виклику Inventory API з Order context

3. Напишіть інтеграційний тест з Testcontainers (Kafka + PostgreSQL)

### Lab 8.2 — Observability Dashboard

```
@workspace Generate a complete observability setup:
1. Docker Compose with Jaeger, Prometheus, Grafana
2. Grafana dashboard JSON for our service metrics
3. Alert rules: p99 latency > 1s, error rate > 1%, circuit breaker open
```

---

## Підсумки модуля

- Kafka producer/consumer: Copilot чудово генерує boilerplate, головне — детально описати error та retry стратегії
- Outbox Pattern: `SaveChangesInterceptor` + relay processor — 100% generated by Copilot при правильному промпті
- Polly v8: Pipeline — ідеальний кандидат для Copilot, багато конфігурації, мало бізнес-логіки
- Distributed Tracing: додавання OpenTelemetry через Copilot — економить 2-3 години налаштування

**← Попередній модуль:** [Модуль 07](../Module-07-Architecture/lecture.md)  
**Наступний модуль →** [Модуль 09: Enterprise Copilot](../Module-09-Enterprise/lecture.md)
