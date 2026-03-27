# Бібліотека промптів для .NET розробників
## Готові до використання шаблони | Версія 1.0

> Скопіюй у Copilot Chat або Inline Chat. Замінюй `[...]` на конкретні значення.
> Зберігай команду цей файл в закладках — це найбільш запитуваний артефакт після тренінгу.

---

## Зміст

- [🏗️ Архітектура та структура](#️-архітектура-та-структура)
- [📦 Сутності та DTO](#-сутності-та-dto)
- [⚙️ Сервіси та CQRS](#️-сервіси-та-cqrs)
- [🗄️ Entity Framework Core](#️-entity-framework-core)
- [✅ Валідація](#-валідація)
- [🧪 Тестування](#-тестування)
- [🔄 Рефакторинг](#-рефакторинг)
- [📝 Документація](#-документація)
- [🔒 Безпека](#-безпека)
- [⚡ Продуктивність](#-продуктивність)
- [🔧 DevOps та Infrastructure](#-devops-та-infrastructure)

---

## 🏗️ Архітектура та структура

### Аналіз архітектури проєкту

```
@workspace Analyze the current project architecture and:
1. Identify any violations of Clean Architecture (dependencies pointing wrong way)
2. Find business logic that leaked into Controller/API layer
3. Find infrastructure concerns (EF, HTTP) in Domain layer
4. List missing abstractions (direct concrete class usage instead of interfaces)
5. Suggest top 3 refactoring priorities by risk (complexity × test coverage)
```

### Проєктування нового Bounded Context

```
Act as a senior DDD architect.
I need to design a [ContextName] bounded context that handles: [відповідальності].

This context interacts with:
- [Context A]: [що отримує/що надає]
- [Context B]: [що отримує/що надає]

Questions:
1. What are the Aggregates and their invariants?
2. What Domain Events does this context publish?
3. What integration events does it subscribe to?
4. What's the Context Map pattern with [Context A]? (ACL, Conformist, Partnership...)
5. Where should [SharedConcept] live?

Use DDD ubiquitous language. Be specific about Aggregate boundaries.
```

### Вертикальний зріз (Feature Slice)

```
Generate a complete vertical slice for feature "[FeatureName]":

Files to create:
1. Features/[FeatureName]/[FeatureName]Endpoint.cs — Minimal API endpoint
2. Features/[FeatureName]/[FeatureName]Query.cs or Command.cs — MediatR request
3. Features/[FeatureName]/[FeatureName]Handler.cs — handler implementation
4. Features/[FeatureName]/[FeatureName]Dto.cs — request/response DTOs
5. Features/[FeatureName]/[FeatureName]Tests.cs — xUnit tests

Business rules: [опис правил]
Endpoint: [HTTP method] [path]
Stack: .NET 8 Minimal API, MediatR, EF Core 8, FluentValidation, Result<T> pattern
Follow .github/copilot-instructions.md
```

---

## 📦 Сутності та DTO

### Доменна сутність з інваріантами

```
Implement [EntityName] domain entity following DDD:

PROPERTIES: [список полів з типами та обмеженнями]

INVARIANTS (enforce in methods, throw DomainException if violated):
[перелік інваріантів]

LIFECYCLE: [Status1] → [Status2] → [Status3] | [TerminalStatus]
State transitions via domain methods (not property setters).

DOMAIN EVENTS to raise:
- [EntityCreatedEvent]: when created
- [StatusChangedEvent]: when status changes (include old and new status)

RULES:
- Private setters for all properties
- Static factory method Create(...) instead of public constructor
- No external dependencies (no EF, HTTP, repositories)
- Use Result<T> pattern ONLY for operations that can fail by business rule
- Throw DomainException for invariant violations
```

### Record / DTO з валідацією

```
Create [RequestName] request record with inline FluentValidation validator:

Fields:
- [Field1]: [тип], [обмеження — required/max length/range/regex]
- [Field2]: [тип], [обмеження]
- [Field3]: [тип], optional, [обмеження якщо є]

Validator rules:
- Validate each field according to constraints above
- Include Ukrainian/English error messages
- Use Must() for cross-field validation: [якщо є залежні поля]

Single file: record + validator class in same file.
```

### Value Object

```
Implement [ValueObjectName] value object:
- Wraps: [wrapped type]
- Constraints: [список обмежень]
- Equality: structural (based on value, not reference)
- Operators: [+/-/*/comparison — вкажи що потрібно]
- ToString(): [формат виводу]
- Implicit/explicit conversions: [від/до чого]
- Static factory: [ValueObjectName].Of([params]) — throws [ExceptionName] on invalid input
- readonly record struct (immutable, value semantics, no heap allocation)
```

---

## ⚙️ Сервіси та CQRS

### MediatR Command + Handler + Validator

```
Generate complete CQRS structure for [ActionName]:

COMMAND ([ActionName]Command):
[перелік полів команди]
Note: [будь-які важливі застереження, наприклад "Price is loaded from DB, not from request"]

VALIDATOR ([ActionName]CommandValidator, FluentValidation):
[перелік правил для кожного поля]

HANDLER ([ActionName]CommandHandler → IRequestHandler<[ActionName]Command, Result<[ReturnType]>>):
Steps:
1. [Крок 1]
2. [Крок 2]
...
N. Return Result<[ReturnType]>.Success([value])

Error cases (return Result.Failure, do NOT throw):
- [Сценарій A]: [ErrorCode]
- [Сценарій B]: [ErrorCode]

Throw only for:
- Infrastructure failures (DB down, HTTP 5xx)

Dependencies: [IRepo1, IService2, ILogger]
All async calls receive CancellationToken ct.
```

### Application Service (без CQRS)

```
Implement [ServiceName] application service:

Interface I[ServiceName]:
[список методів з сигнатурами та коментарями]

Implementation:
- Constructor injection: [залежності]
- All methods async with CancellationToken ct as last parameter
- Use Result<T> for business rule violations, throw for infrastructure failures
- Structured logging at Information level for each business operation
- Log parameters: [які параметри включати в лог]
- Thread-safe (no shared mutable state)
```

### MediatR Pipeline Behavior

```
Implement MediatR IPipelineBehavior<TRequest, TResponse> for [CrossCuttingConcern]:

Behavior: [опис що робить]
Apply to: [All requests / Only commands (ICommand marker) / Only queries]
On success: [що робить]
On failure/exception: [що робить]
Order in pipeline: [Before/After validation / Before/After transaction]

Registration: services.AddTransient(typeof(IPipelineBehavior<,>), typeof([BehaviorName]<,>))
```

---

## 🗄️ Entity Framework Core

### EF Core конфігурація сутності

```
IEntityTypeConfiguration<[EntityName]> for EF Core 8:
- Table: [table_name] (snake_case)
- PK: [тип та стратегія — identity, hi-lo, GUID]
- [Поле]: [тип в БД, nullable, default value, index]
- [Enum field]: stored as string, max length [N]
- [Money/Value Object]: owned entity → columns: [column_name_amount] decimal([p],[s]), [column_name_currency] varchar(3)
- FK [NavigationProp]: [cascade / restrict / set null]
- [Collection]: owned collection → table [child_table_name]
- RowVersion for optimistic concurrency on [fields affected by concurrent updates]
- UTC for all DateTime/DateTimeOffset columns
- Ignored: [props to ignore — domain events, computed props]
- No lazy loading
```

### Складний EF Core запит

```
EF Core 8 query for [QueryDescription]:

Filters (apply only if parameter HasValue/not null/not empty):
- [Filter1]: [умова]
- [Filter2]: [умова]

Sorting: by [SortField] [Asc/Desc], secondary sort by [Field2] to ensure stable pagination

Projection to [DtoName]:
- [SourceProp] → [DtoProp]
- [NavigationProp.ChildProp] → [DtoProp] (no N+1 — use Select projection, not Include)

Requirements:
- AsNoTracking (read-only)
- Single round-trip to DB (no N+1)
- CancellationToken ct
- Return IReadOnlyList<[DtoName]>
```

### Bulk операції (EF Core 7+)

```
Bulk [update/delete] using EF Core 8 ExecuteUpdateAsync/ExecuteDeleteAsync:
- Entity: [EntityName]
- Filter: [умова]
- Update: [поля для оновлення з новими значеннями]
- Return: count of affected records
- Log: operation name, filter params, affected count at Information level
- CancellationToken: yes
Note: no entity loading, single SQL statement
```

---

## ✅ Валідація

### FluentValidation для складних сценаріїв

```
FluentValidation AbstractValidator<[TypeName]> with:

Simple rules:
- [Field]: [NotEmpty/MaximumLength(N)/GreaterThan(0)/EmailAddress/etc.]

Conditional rules:
- [Field]: required only when [OtherField] == [Value]

Collection rules:
- [CollectionField]: not empty, max [N] items
- Each item in [CollectionField]: [child validator class or inline rules]

Cross-field rules:
- [Field1] must be greater than [Field2]
- [DateTo] must be after [DateFrom]

Async rules (DB checks):
- [Field]: must exist in DB (inject [IRepository] via constructor, use MustAsync)

Error messages: Ukrainian language, specific and actionable
```

---

## 🧪 Тестування

### Повне покриття методу

```
Generate comprehensive xUnit tests for [MethodName] in #file:[FileName].cs

Test stack: xUnit, FluentAssertions, FakeItEasy, AutoFixture
Naming: [MethodName]_[StateUnderTest]_[ExpectedBehavior]
Pattern: AAA (// Arrange // Act // Assert with blank lines between sections)

Cover:
1. Happy path — valid input, expected result
2. Edge cases — [boundary values specific to this method]
3. Each business rule violation — one test per rule
4. Null/empty inputs (where applicable)
5. Exception propagation from dependencies

FakeItEasy:
- Use A.Fake<T>() for all dependencies
- Verify side effects: A.CallTo(...).MustHaveHappenedOnceExactly()
- Capture arguments for complex assertions: A.CallTo(...).Invokes(...)

AutoFixture: use _fixture.Build<T>().With(...).Create() for test data
```

### Інтеграційний тест ендпоінту

```
xUnit integration test for [HTTP method] [endpoint path] using:
- WebApplicationFactory<Program>
- Testcontainers PostgreSQL (configure in IAsyncLifetime)
- EF Core migrations run on test DB before tests
- Each test wrapped in transaction, rollback after (IAsyncLifetime per test)

Seed data: [опис тестових даних що потрібні]

Test cases:
1. Valid request → [expected status code] + [expected response body fields]
2. [Invalid scenario] → [expected error status + ProblemDetails structure]
3. [Authorization scenario if applicable]

Assert:
- HTTP status code
- Response body (use System.Text.Json or explicit DTO deserialization)
- Database state after operation (direct DbContext query)
```

### TDD — тести перед реалізацією

```
Generate FAILING xUnit tests that SPECIFY the behavior of [InterfaceName]:

Business rules to test:
[детальний список правил — кожне правило = мінімум 1 тест]

Requirements:
- Tests should compile but FAIL (no implementation exists yet)
- Cover at least [N] test cases
- Include boundary value tests for: [числові поля]
- Include negative path tests for each business rule
- Use FluentAssertions for all assertions
- DO NOT write the implementation
```

---

## 🔄 Рефакторинг

### Аналіз legacy коду

```
/explain
Analyze this legacy code:
1. What does it do? List all business rules it implements.
2. What are all the side effects?
3. What anti-patterns are present? (list each with line reference)
4. What security issues exist?
5. List all implicit dependencies (hidden coupling)
6. Suggest modern .NET 8 equivalent approach for each issue
```

### Рефакторинг конкретного антипатерну

```
Refactor #file:[FileName].cs to fix [specific problem]:

Current problem: [опис проблеми]

Requirements:
- [Конкретна вимога 1]
- [Конкретна вимога 2]
- Keep the same public interface (do not change method signatures)
- All existing tests must remain green after refactoring
- Add missing CancellationToken parameters
- Use .NET 8 language features where appropriate (pattern matching, records, etc.)

DO NOT:
- Change business logic
- Add new features
- Modify test files
```

### Copilot Edits — масовий рефакторинг

```
[Для Copilot Edits режиму — задай список файлів]

Task: Add CancellationToken to all async methods that are missing it.

Rules:
- Parameter name: ct
- Position: last parameter
- Propagate ct to ALL internal async calls
- Update method signature everywhere it's called
- Do NOT change synchronous methods
- Do NOT add to private methods that only call synchronous code
```

---

## 📝 Документація

### XML документація для класу

```
/doc
Add comprehensive XML documentation to all public members of this class:

For each method include:
- <summary>: what it does (1-2 sentences)
- <param name="...">: each parameter with constraints/expectations
- <returns>: what is returned and in what cases
- <exception cref="...">: all exceptions that can be thrown and when
- <remarks>: edge cases, performance notes, thread safety
- <example>: code snippet for complex methods (at least for public API surface)

For the class:
- <summary>: responsibility and role in the system
- <remarks>: thread safety statement, usage context
- <seealso cref="...">: related types

Use C# XML doc syntax. Be specific, not generic.
```

### Architecture Decision Record

```
Generate an ADR for the decision: [рішення]

Format: Michael Nygard style

Context: [опис контексту і проблеми]
Team constraints: [розмір команди, досвід, терміни]
Rejected alternatives: [A], [B] — explain why each was rejected
Chosen decision: [C]

Include:
- Status: [Proposed / Accepted / Deprecated / Superseded]
- Consequences: positive, negative, neutral
- Implementation notes (specific to our .NET stack)
- Review date: [через скільки переглянути рішення]
```

---

## 🔒 Безпека

### Security Review

```
Perform a security review of #file:[FileName].cs focusing on OWASP Top 10:

Check for:
1. A01 Broken Access Control — unauthorized resource access, missing ownership checks
2. A02 Cryptographic Failures — weak hashing, hardcoded secrets, sensitive data in logs
3. A03 Injection — SQL, LINQ, command injection via string concatenation
4. A05 Security Misconfiguration — verbose errors, debug info in production
5. A07 Auth/Auth failures — missing [Authorize], IDOR vulnerabilities
6. A09 Logging failures — insufficient audit trail, sensitive data logged

Return: numbered list with severity (Critical/High/Medium/Low), 
line number, description, and specific remediation code.
```

### Безпечна реалізація конкретного патерну

```
Implement [SecurityFeature] following security best practices:

Requirements:
- [Вимога 1]
- Use industry-standard algorithms/libraries (not custom crypto)
- No hardcoded values — all from IConfiguration
- Audit log: log all [actions] with user id and timestamp (do NOT log sensitive values)
- Rate limiting: [якщо застосовно]

Reference: [OWASP cheatsheet / RFC / standard якщо є]
```

---

## ⚡ Продуктивність

### BenchmarkDotNet

```
Generate BenchmarkDotNet benchmark comparing:
- [Approach A]: [опис]
- [Approach B]: [опис]
- [Approach C]: [опис — якщо є]

Setup:
- GlobalSetup: [що підготувати — DB connection, test data]
- Params: [розміри датасетів для тестування, наприклад N = 100, 1000, 10000]

For each benchmark:
- [OperationName]: [що вимірювати]

Add [MemoryDiagnoser] (allocations matter).
Use realistic data sizes.
Include [SimpleJob(RuntimeMoniker.Net80)] attribute.
```

### Аналіз продуктивності EF Core запиту

```
Optimize this EF Core query for performance:
[вставити код запиту]

Analyze:
1. Is there N+1? Where?
2. Is AsNoTracking missing for read-only?
3. Are unnecessary columns loaded (no projection)?
4. Can Count/Any replace ToList().Count?
5. Can the query be rewritten as a single SQL with projection?
6. Are there missing database indexes? (suggest which columns to index)

Rewrite with all optimizations applied. Keep the same return type.
```

---

## 🔧 DevOps та Infrastructure

### Dockerfile

```
Generate optimized multi-stage Dockerfile for .NET 8 project:
- Solution structure: [описати структуру csproj — Api, Application, Infrastructure]
- Base image: mcr.microsoft.com/dotnet/aspnet:8.0-alpine (minimal size)
- Run as non-root user
- Health check: GET /health/live
- Exposed port: 8080
- Copy only necessary projects (not test projects)
- Label with git commit hash (ARG GIT_HASH)
- .dockerignore already exists
```

### Docker Compose для локальної розробки

```
Generate docker-compose.yml for local development with:
- [ServiceName] .NET API (build from Dockerfile, hot-reload via volume)
- PostgreSQL [version]: persistent volume, health check
- [Redis / RabbitMQ / Kafka — вкажи що потрібно]: [конфіг]
- Seq (structured log viewer): http://localhost:5341
- [pgAdmin — якщо потрібен]: http://localhost:5050

Environment variables from .env file (include .env.example).
Networks: isolated bridge network.
Depends_on with condition: service_healthy.
```

### GitHub Actions CI

```
Generate GitHub Actions workflow for .NET 8 project:

Trigger: PR to main, push to main

Jobs:
1. build-and-test:
   - PostgreSQL service container (Testcontainers will use it or direct connection)
   - dotnet restore with NuGet cache
   - dotnet build --configuration Release
   - dotnet test with coverage (Coverlet, format: cobertura)
   - Fail if coverage < [threshold]%
   - Publish test results as GitHub Actions summary

2. security-scan:
   - dotnet list package --vulnerable --include-transitive
   - Fail on CRITICAL vulnerabilities

Environment variables: all from GitHub Secrets (document which secrets needed)
```
