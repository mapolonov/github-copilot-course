# GitHub Copilot — Шпаргалка для .NET розробника
## Роздрукуй та постав поруч з монітором

---

## Гарячі клавіші

### Visual Studio 2022

| Дія | Клавіші |
|-----|---------|
| Прийняти пропозицію | `Tab` |
| Відхилити | `Esc` |
| Наступна пропозиція | `Alt + ]` |
| Попередня пропозиція | `Alt + [` |
| Відкрити всі пропозиції | `Ctrl + Alt + \` |
| Inline Chat | `Ctrl + I` |
| Copilot Chat панель | `Ctrl + Shift + C` |
| Прийняти одне слово | `Ctrl + →` |

### VS Code

| Дія | Клавіші |
|-----|---------|
| Прийняти пропозицію | `Tab` |
| Відхилити | `Esc` |
| Наступна / Попередня | `Alt + ]` / `Alt + [` |
| Прийняти одне слово | `Ctrl + →` |
| Inline Chat | `Ctrl + I` |
| Copilot Chat панель | `Ctrl + Shift + I` |
| Inline Chat у терміналі | `Ctrl + I` (у вбудованому терміналі) |

---

## Chat — Учасники та змінні

| Синтаксис | Що додає до контексту |
|-----------|----------------------|
| `@workspace` | Весь індексований проєкт |
| `@vscode` | Налаштування та команди редактора |
| `@terminal` | Команди терміналу |
| `#file:FileName.cs` | Конкретний файл |
| `#editor` | Поточний відкритий файл |
| `#selection` | Виділений текст |
| `#codebase` | (Enterprise) Проіндексована codebase |

---

## Slash Commands

| Команда | Призначення | Коли використовувати |
|---------|------------|---------------------|
| `/explain` | Пояснити код | Legacy код, складні алгоритми |
| `/fix` | Виправити помилку | Compile error, баг, NullRef |
| `/tests` | Згенерувати тести | Після написання методу |
| `/doc` | Додати документацію | XML comments, README |
| `/simplify` | Спростити код | Висока когнітивна складність |
| `/optimize` | Оптимізувати | Повільний EF Core запит |

---

## Найкращі промпти для .NET — Готові шаблони

### Entity / Record
```
// [EntityName] entity: [field1 (type, constraints)], [field2], ...
// [Optional: навігаційні властивості]
public class [EntityName]
```

### Repository / Service
```
// [ServiceName]: [опис відповідальності]
// Dependencies: [список залежностей]
// Methods: [список публічних методів з коротким описом]
// USE: async/await, CancellationToken ct, Result<T> for errors
```

### MediatR Handler
```
// [CommandName]Handler:
// 1. Validate [щось]
// 2. Load [entity] via [repository]
// 3. [Основна бізнес-операція]
// 4. Save + publish [EventName]
// Return: Result<[ReturnType]>
// Throw ONLY for infrastructure failures, return Result.Failure for business rules
```

### Unit Test
```
// [MethodName]_[SценарійUndерTest]_[ОчікуванийРезультат]
// AAA pattern, xUnit, FluentAssertions, FakeItEasy
// Дані через AutoFixture
[Fact]
public async Task
```

### EF Core запит
```
// Query [EntityName] where [умова фільтрації]
// Include: [навігаційні властивості]
// Project to [DtoName] (use Select, NOT AutoMapper)
// AsNoTracking (read-only), CancellationToken ct
// Return: IReadOnlyList<[DtoName]>
```

### Валідація (FluentValidation)
```
// Validator for [RequestName]:
// - [Property1]: [правила] (required, max length N, regex...)
// - [Property2]: [правила]
// Each rule = separate RuleFor, include error message in Ukrainian/English
public class [RequestName]Validator : AbstractValidator<[RequestName]>
```

### Middleware
```
// ASP.NET Core IMiddleware:
// Purpose: [що робить]
// Input: [звідки читає — headers, body, context]
// Output: [що додає до response / context]
// Error handling: [як обробляє помилки]
// Performance: [про що треба піклуватися]
public class [Name]Middleware : IMiddleware
```

---

## Copilot CLI — Термінал

```bash
# Пояснити команду
gh copilot explain "docker run -p 8080:80 -e DB_URL=... myapp"

# Запропонувати команду за описом
gh copilot suggest "watch folder for changes and restart dotnet run"

# Псевдоніми для зручності (додати в .bashrc / PowerShell profile):
# ghce → gh copilot explain
# ghcs → gh copilot suggest
```

---

## Правила контексту — Що відкрити перед роботою

```
Завдання                    Відкрити у вкладках
──────────────────────────────────────────────────────────
Новий Service              Interface + Entity + аналогічний Service
Новий Handler              Command + Repository Interface + Aggregate
Написати тести             Клас який тестуєш + його Interface
Рефакторинг                Клас + тести (якщо є) + аналогічний "добрий" клас
EF Configuration           Entity + аналогічний Configuration
```

---

## Acceptance Rate — орієнтири

| Показник | Значення |
|----------|---------|
| < 15% | Погані промпти або нерелевантний контекст |
| 25–40% | Норма для досвідченого користувача |
| > 50% | Або дуже прості задачі, або варто критичніше переглядати |

**Низький acceptance rate ≠ погано.** Відхилення та коригування — теж цінна взаємодія.

---

## Чеклист перед прийняттям пропозиції

- [ ] API / методи дійсно існують? (перевір IntelliSense)
- [ ] Немає `.Result` / `.Wait()` в async контексті?
- [ ] `CancellationToken` передається далі?
- [ ] Немає hardcoded рядків (credentials, magic numbers)?
- [ ] Немає SQL конкатенації з user input?
- [ ] Логіка відповідає бізнес-правилам (Copilot не знає твого домену!)?
