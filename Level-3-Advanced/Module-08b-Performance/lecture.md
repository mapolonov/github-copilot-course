# Модуль 08b: Продуктивність та Діагностика з Copilot
## Рівень: Advanced | Тривалість: 3 години

---

## Цілі модуля

- Використовувати Copilot для написання BenchmarkDotNet бенчмарків
- Аналізувати memory allocations та GC tress через Copilot-generated profiling код
- Генерувати `dotnet-trace` / `dotnet-dump` сесії та інтерпретувати результати з Copilot
- Знаходити performance bottlenecks у EF Core та async коді через Copilot

---

## Чому цей модуль важливий

**Copilot для performance коду дає великий ROI саме тому, що:**
- BenchmarkDotNet boilerplate — шаблонний, нудний, але критичний
- Порівняльні бенчмарки (3-5 підходів) — 30 хвилин замість 3 годин
- Profiling скрипти — рідко пишуть "руками", але Copilot знає синтаксис
- Аналіз flamegraph виводу — Copilot пояснює результати

---

## 1. BenchmarkDotNet — мікробенчмарки з Copilot

### 1.1 Базова структура через промпт

```csharp
// Промпт:
/*
Generate BenchmarkDotNet benchmark comparing three approaches for bulk inserting 
10,000 orders into PostgreSQL:
  A) EF Core Add + SaveChanges (baseline)
  B) EF Core ExecuteUpdateAsync (batch SQL)
  C) Npgsql COPY protocol via NpgsqlConnection.BeginBinaryImportAsync

Setup:
- GlobalSetup: create PostgreSQL connection, ensure "orders" table exists
- GlobalCleanup: truncate table
- Params: [100, 1_000, 10_000] items

Add [MemoryDiagnoser], [SimpleJob(RuntimeMoniker.Net80)]
Use realistic Order data (10 fields including nested lines)
*/

[MemoryDiagnoser]
[SimpleJob(RuntimeMoniker.Net80)]
public class BulkInsertBenchmarks
{
    // Copilot заповнить
}
```

### 1.2 Порівняння LINQ vs for loops

```csharp
// Промпт:
/*
BenchmarkDotNet benchmark: compare 6 approaches for summing order totals
from List<OrderLine> (100K items):
  1. LINQ .Sum()
  2. foreach loop
  3. for loop with index
  4. Span<T> iteration (ReadOnlySpan<OrderLine>)
  5. PLINQ .AsParallel().Sum()
  6. Vector<T> SIMD (if applicable to decimal)

Params: [1_000, 10_000, 100_000] items
MemoryDiagnoser + ThreadingDiagnoser
Note: decimal vs double performance difference — include both
*/
```

### 1.3 Аналіз результатів з Copilot

```
Paste benchmark results into Copilot Chat:

| Method               | N      | Mean       | Allocated |
|---------------------- |------- |----------- |---------- |
| EfCoreAdd            | 10000  | 4,523.1 ms | 45.2 MB   |
| ExecuteUpdateAsync   | 10000  |   234.7 ms |  2.1 MB   |
| NpgsqlCopy           | 10000  |    89.3 ms |  0.8 MB   |

Ask:
"Analyze these benchmark results:
 1. What's the relative performance difference?
 2. Why does NpgsqlCopy have much lower allocation?
 3. What are the tradeoffs of NpgsqlCopy (type safety, error handling)?
 4. When would you choose each approach in production?
 5. Are these results surprising given the implementation complexity?"
```

---

## 2. Memory Profiling — allocation hunting

### 2.1 ObjectAllocationAnalyzer через Roslyn

```csharp
// Промпт:
/*
I'm seeing unexpectedly high memory allocations in hot path OrderCalculation.
Generate code to:
1. Use ObjectAllocationAnalyzer NuGet analyzer to find allocations
2. Add [SkipLocalsInit] where appropriate  
3. Convert string concatenation to interpolated strings or StringBuilder
4. Replace LINQ in hot path with for loops (explain which, why)
5. Use ArrayPool<T>.Shared.Rent/Return instead of new T[] in this method

Hot path is: [вставити метод]
Target: reduce allocations by >80% (per BenchmarkDotNet MemoryDiagnoser)
*/
```

### 2.2 Structs vs Classes — Copilot аналіз

```csharp
// Промпт:
/*
Analyze this type and recommend whether it should be class, struct, or record struct:
[вставити тип]

Consider:
1. Size (number and type of fields)
2. How it's used (passed by value? stored in collections? nullable?)
3. GC pressure (how often created?)
4. Mutability requirements

Rewrite as the recommended type with explanation.
Add XML comment explaining the choice.
*/
```

### 2.3 Генерація ObjectPool pattern

```csharp
// Промпт:
/*
Implement ObjectPool<T> pattern for [ExpensiveObject]:
- Use Microsoft.Extensions.ObjectPool.ObjectPool<T>
- Implement IPooledObjectPolicy<T>: Create() and Return(obj)
- Return() resets the object state (call Reset() or clear collections)
- Register as Singleton in DI
- Show usage pattern in service (GetObject/Return in try/finally)
- Add BenchmarkDotNet test: with pool vs without pool (1000 iterations)
*/
```

---

## 3. dotnet-trace та діагностика

### 3.1 Генерація профілювальних скриптів

```bash
# Попроси Copilot CLI або Chat:
gh copilot suggest "collect CPU trace for dotnet process named MyApi for 30 seconds, 
                    focusing on thread pool and GC events, save as myapi-trace.nettrace"

# Результат:
dotnet trace collect \
  --process-id $(pgrep -f "MyApi") \
  --duration 00:00:30 \
  --providers "Microsoft-DotNETCore-SampleProfiler,Microsoft-Windows-DotNETRuntime" \
  --output myapi-trace.nettrace

# Аналіз:
dotnet trace convert myapi-trace.nettrace --format SpeedScope
# Відкрити в speedscope.app
```

### 3.2 Скрипт моніторингу в реальному часі

```csharp
// Промпт:
/*
Generate a .NET 8 diagnostic tool using Microsoft.Diagnostics.NETCore.Client that:
1. Attaches to a running process by name (arg: --process-name MyApi)
2. Every 5 seconds prints:
   - Thread pool: pending work items, active threads
   - GC: gen0/1/2 collection counts, heap size MB
   - Exception rate: exceptions/sec in last 5s
   - Top 5 methods by CPU time (from sampling)
3. Output as structured JSON (for Splunk/Datadog ingestion)
4. Graceful stop on Ctrl+C
5. Alert to console if heap > [threshold]MB or exception rate > [N/sec]
*/
```

### 3.3 Аналіз dump файлів з Copilot

```
I took a memory dump of our production service. 
Here's the output of "dumpheap -stat" from dotnet-dump:

[вставити вивід]

Questions:
1. Which types are consuming the most memory?
2. Are there obvious memory leaks (types that should not accumulate)?
3. What questions would you ask to investigate further?
4. Generate dotnet-dump analyze commands to investigate the top suspects
```

---

## 4. EF Core Performance Analysis

### 4.1 Query Plan Analysis

```csharp
// Промпт:
/*
This EF Core query is slow (measured: ~2 seconds for 50K rows).
Help me:
1. Enable EF Core query logging with sensitive data logging to see the SQL
2. Identify potential issues from the LINQ code:
   [вставити LINQ запит]
3. Suggest indexes that would help
4. Rewrite query if fundamentally wrong approach
5. Add AsNoTracking where missing
6. Show how to use EF.Functions.Like vs Contains (performance difference)
*/
```

### 4.2 Дослідження connection pool

```csharp
// Промпт:
/*
Generate health check and metrics for Npgsql connection pool:
1. IHealthCheck that reports:
   - Active connections
   - Idle connections
   - Pending acquisitions
   - Connection timeouts in last minute
2. Prometheus metrics via IMeterFactory:
   - db.connections.active (gauge)
   - db.connections.idle (gauge)
   - db.connection.acquisition.duration (histogram)
3. Alert via ILogger.LogWarning when pending > 10 or active > 80% of max pool size
4. Tune NpgsqlConnectionStringBuilder for high-throughput: 
   MinPoolSize, MaxPoolSize, ConnectionIdleLifetime
*/
```

---

## 5. Async Performance

### 5.1 `ValueTask` vs `Task` через Copilot

```csharp
// Промпт:
/*
Analyze this method and determine if ValueTask is appropriate:
[вставити метод]

Criteria:
- Is the synchronous (hot) path common?
- Is this called in a tight loop?
- Does it complete synchronously often (e.g., from cache)?

If ValueTask is appropriate: rewrite with proper ValueTask usage.
If not: explain why Task is correct here.
Add BenchmarkDotNet test showing the difference in allocations.
*/
```

### 5.2 System.Threading.Channels для backpressure

```csharp
// Промпт:
/*
Implement producer-consumer pipeline using System.Threading.Channels for 
processing [EventType] events with backpressure:

Channel config:
- BoundedChannel capacity: [N] (backpressure trigger)
- BoundedChannelFullMode.Wait (producer blocks when full, no data loss)
- SingleReader: false (multiple consumer tasks)
- SingleWriter: false (multiple producers)

Producer:
- IAsyncEnumerable<[EventType]> source
- Writes to channel, respects cancellation
- Logs when channel is full (backpressure applied)

Consumer (BackgroundService):
- [parallelism] concurrent processing tasks
- Each task reads from channel via await foreach
- On processing failure: dead-letter to [DLQ]
- Graceful shutdown: drain in-progress items (max 30s), close channel

Add metrics: channel.occupancy, processing.duration, dlq.count
*/
```

---

## Практичне завдання (60 хв)

### Lab P-1: Benchmark і оптимізація

**Задача:** знайти і виправити performance bottleneck в наданому сервісі

Тренер надає `SlowReportService.cs` з реальними проблемами:
- N+1 queries в EF Core
- LINQ в hot path замість for loop
- Boxing через object[] array
- `string concatenation` замість StringBuilder
- Missing `AsNoTracking`

**Кроки:**
1. Написати BenchmarkDotNet benchmark для `GenerateReport()` методу (до оптимізації)
2. Запустити, зафіксувати базову лінію (Mean, Allocated)
3. Попросити Copilot проаналізувати та виправити проблеми:
   ```
   Analyze this method for performance issues. 
   Focus on: N+1 queries, unnecessary allocations, blocking operations.
   Fix all issues, keep the same return type and behavior.
   ```
4. Запустити benchmark після оптимізації
5. Порівняти результати — задокументувати покращення

**Очікуване покращення:** 5-15x по Mean, 80%+ по Allocated.

### Lab P-2: Profile живого сервісу

1. Запустити надане демо-API
2. Запустити навантаження через `bombardier -c 10 -n 1000 http://localhost:5000/api/orders`
3. Підключити `dotnet-trace` та зібрати 15-секундний trace
4. Відкрити в Speedscope, визначити top 3 hotspot методи
5. Спитати Copilot Chat:
   ```
   Here are the top methods by CPU time from dotnet-trace:
   [вставити список]
   Which ones are surprising? Which indicate bugs vs expected behavior?
   Suggest investigation steps for the top hotspot.
   ```

---

## Підсумки модуля

| Задача | Copilot ROI |
|--------|------------|
| Написання BenchmarkDotNet | ★★★★★ — шаблонний код, ідеально для Copilot |
| Аналіз результатів бенчмарку | ★★★★☆ — пояснює, але потрібне ваше розуміння |
| dotnet-trace команди | ★★★★★ — синтаксис складний, Copilot знає |
| Виправлення EF Core N+1 | ★★★★★ — найбільший практичний ефект |
| Memory allocation hunting | ★★★☆☆ — потрібен досвід читати результати |

**← Попередній модуль:** [Модуль 08: Розподілені системи](../Module-08-Distributed/lecture.md)  
**← Модуль 09:** [Enterprise Copilot](../Module-09-Enterprise/lecture.md)  
**↑ До програми:** [README](../../README.md)
