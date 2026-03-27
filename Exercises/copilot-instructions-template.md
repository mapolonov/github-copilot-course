# Copilot Instructions Template
# Файл: .github/copilot-instructions.md
# 
# Скопіюйте цей файл в .github/copilot-instructions.md вашого проєкту
# та адаптуйте під ваш стек і стандарти команди.
# ─────────────────────────────────────────────────────────────────────

## Project Overview
<!-- TODO: Короткий опис проєкту та його призначення (2-4 речення) -->
This is a [describe service] microservice for [company/domain].
Domain: [brief domain description].
Team: [N] .NET developers.

## Technology Stack (Mandatory - do not suggest alternatives)
- Runtime: .NET 8, C# 12
- API: [ASP.NET Core Web API / Minimal API]
- ORM: [Entity Framework Core 8 / Dapper] with [PostgreSQL / SQL Server]
- CQRS: [MediatR 12 / None]
- Validation: FluentValidation 11
- Messaging: [MassTransit / Confluent.Kafka / None]
- Resilience: Polly v8
- Testing: xUnit + FluentAssertions + FakeItEasy + AutoFixture + Testcontainers
- Logging: Serilog (structured, NO Console.WriteLine ever)
- Observability: OpenTelemetry → [Jaeger/Zipkin] + Prometheus
- DI: Microsoft.Extensions.DependencyInjection (built-in)

## Architecture Pattern
<!-- TODO: виберіть та розкоментуйте -->
<!-- Clean Architecture: Domain → Application → Infrastructure → API -->
<!-- Vertical Slice: Features/{FeatureName}/ per use case -->
<!-- N-Tier: Controllers → Services → Repositories -->

Rules:
- [Layer X] has ZERO dependency on [Layer Y]
- All external dependencies behind interfaces
- [Repository/Port] interfaces in [Application/Domain], implementations in Infrastructure

## Naming Conventions
- Async methods: MUST end with `Async` suffix
- Commands: `[Verb][Noun]Command` (e.g., `CreateOrderCommand`)
- Queries: `Get[Noun]By[Key]Query` (e.g., `GetOrderByIdQuery`)
- Events: `[Noun][PastTense]Event` (e.g., `OrderPlacedEvent`)
- DTOs/Records: `[Action][Noun]Request` / `[Noun]Dto` / `[Noun]Response`
- Database tables: `snake_case` plural (e.g., `order_lines`)

## Async/Threading Rules
- ALL I/O-bound methods MUST be async
- Accept `CancellationToken ct` as LAST parameter on all async public methods
- Propagate `ct` to ALL internal async calls (EF, HTTP, etc.)
- NEVER use `.Result`, `.Wait()`, or `.GetAwaiter().GetResult()`
- `ConfigureAwait(false)` ONLY in library/infrastructure code, NOT in handlers/controllers

## Error Handling Pattern
- Business rule violations: return `Result<T>.Failure(Error)` — NEVER throw for business errors
- Infrastructure failures (DB unavailable, HTTP 5xx): throw exceptions
- Domain invariant violations: throw custom domain exceptions (DomainException subclasses)
- API layer: ALL errors → `ProblemDetails` format (RFC 7807)
- NEVER swallow exceptions silently — log before return/rethrow

## Logging Standards
- Use structured logging: `_logger.LogInformation("Order {OrderId} placed", orderId)`
- NEVER string interpolation in log messages: NO `$"Order {orderId} placed"`
- NEVER log: passwords, secrets, tokens, credit cards, national IDs
- Log levels:
  - Trace: internal loops, iterations
  - Debug: developer diagnostics
  - Information: business events (order created, payment processed)
  - Warning: recoverable issues (retry attempt, fallback used)
  - Error: failures requiring attention (with exception object)
  - Critical: system-level failures

## Code Quality Rules
- All `public` members MUST have XML documentation comments
- No magic numbers or strings — use named constants or enums
- Prefer `pattern matching` over long if-else chains
- Prefer `record` for DTOs and Value Objects (immutable)
- Use `var` ONLY when type is obvious from RHS expression
- Max method length: 30 lines (suggest extraction if longer)
- Max class length: 200 lines (suggest refactoring if longer)

## Security Rules (Non-Negotiable)
- NEVER construct SQL strings from user input — always parameterized
- NEVER hardcode credentials, connection strings, API keys
- ALL configuration via `IConfiguration` / `IOptions<T>`
- Validate ALL external inputs (API, message queue, file import)
- NEVER trust client-provided IDs for authorization — verify ownership server-side

## What NOT To Generate
- NO AutoMapper — use explicit `Select()` projections or manual mapping
- NO `[FromServices]` in endpoint method parameters
- NO `Thread.Sleep()` — use `Task.Delay()` with CancellationToken
- NO catching `Exception` without specific handling — catch specific exception types
- NO TODO comments without GitHub issue reference format: `// TODO: #123 -`
- NO blocking async: `.Result`, `.Wait()`, `.GetAwaiter().GetResult()`
- NO static mutable state (static fields/properties storing mutable data)
