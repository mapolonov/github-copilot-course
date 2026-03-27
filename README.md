# GitHub Copilot для .NET розробників
## Корпоративна програма навчання

---

## Огляд програми

| Рівень | Аудиторія | Тривалість | Формат |
|---------|-----------|--------------|--------|
| **Beginner** | Junior / Mid — ніколи не працювали з Copilot | 2 дні | Лекції + практика |
| **Intermediate** | Mid / Senior — базовий досвід з Copilot | 2.5 дні | Workshop + code review |
| **Advanced** | Senior / Lead — впевнений користувач Copilot | 2 дні | Architecture sessions + pair programming |

**Загальна тривалість повного курсу:** 7 днів (можна розділити на 3 окремі блоки)  
**Фінальний проєкт (Capstone):** додатково 1 день — наскрізне завдання для всіх рівнів

---

## Структура курсу

```
GitHubCopilotLearning/
├── README.md                              ← цей файл
│
├── Level-1-Beginner/
│   ├── Module-01-Introduction/            ← Що таке Copilot, як працює LLM
│   ├── Module-02-First-Steps/             ← Inline completions, Chat, встановлення
│   └── Module-03-Prompt-Basics/           ← Основи prompt engineering для .NET
│
├── Level-2-Intermediate/
│   ├── Module-04-DotNet-Patterns/         ← CRUD, EF Core, async/await з Copilot
│   ├── Module-05-Testing/                 ← xUnit, FluentAssertions, тест-сценарії
│   ├── Module-06-Refactoring/             ← Рефакторинг, документація, Code Review
│   └── Module-06b-CLI-and-Actions/        ← Copilot CLI, gh copilot, GitHub Actions  ← НОВИЙ
│
├── Level-3-Advanced/
│   ├── Module-07-Architecture/            ← CQRS, Clean Arch, DDD з Copilot
│   ├── Module-08-Distributed/             ← Kafka, Outbox, мікросервіси
│   ├── Module-08b-Performance/            ← BenchmarkDotNet, profiling, allocations  ← НОВИЙ
│   └── Module-09-Enterprise/              ← Custom instructions, Workspace, агенти
│
└── Exercises/
    ├── all-labs.md                        ← Лабораторні роботи всіх рівнів
    ├── capstone-project.md                ← Фінальний проєкт MiniShop
    ├── cheat-sheet.md                     ← Шпаргалка (роздрукуй!)
    ├── anti-patterns.md                   ← Галерея антипатернів
    ├── prompt-library.md                  ← Бібліотека готових промптів  ← НОВИЙ
    ├── copilot-instructions-template.md   ← Шаблон для вашого проєкту
    └── training-schedule.md               ← Варіанти розкладу
```

---

## Попередні вимоги

### Для всіх рівнів
- Visual Studio 2022 (17.8+) або VS Code з C# Dev Kit
- GitHub акаунт з активною підпискою Copilot (Individual / Business / Enterprise)
- .NET 8 SDK
- Git

### Для Intermediate+
- Docker Desktop / Podman Desktop
- SQL Server або PostgreSQL (локально або в Docker)

### Для Advanced
- Розуміння мікросервісної архітектури
- Досвід роботи з Kafka / RabbitMQ / Azure Service Bus

---

## Принципи навчання

1. **Hands-on First** — кожна теоретична концепція одразу закріплюється вправою
2. **Real-world Code** — жодного академічного HelloWorld, тільки production-grade патерни
3. **Fail & Learn** — демонструємо галюцинації та неправильні пропозиції Copilot, навчаємо критичному мисленню
4. **Context is King** — червона нитка курсу: якість output = якість input
5. **Measure Everything** — метрики продуктивності: час, покриття тестами, cyclomatic complexity до/після

---

## Швидкі посилання

### Модулі

| # | Модуль | Рівень | Тривалість |
|---|--------|--------|-----------|
| 01 | [Вступ до Copilot](Level-1-Beginner/Module-01-Introduction/lecture.md) | Beginner | 3 год |
| 02 | [Перші кроки](Level-1-Beginner/Module-02-First-Steps/lecture.md) | Beginner | 3 год |
| 03 | [Основи Prompt Engineering](Level-1-Beginner/Module-03-Prompt-Basics/lecture.md) | Beginner | 3 год |
| 04 | [.NET патерни з Copilot](Level-2-Intermediate/Module-04-DotNet-Patterns/lecture.md) | Intermediate | 4 год |
| 05 | [Тестування](Level-2-Intermediate/Module-05-Testing/lecture.md) | Intermediate | 4 год |
| 06 | [Рефакторинг та документація](Level-2-Intermediate/Module-06-Refactoring/lecture.md) | Intermediate | 3 год |
| 06b | [Copilot CLI та GitHub Actions](Level-2-Intermediate/Module-06b-CLI-and-Actions/lecture.md) | Intermediate | 2.5 год |
| 07 | [Архітектурні патерни](Level-3-Advanced/Module-07-Architecture/lecture.md) | Advanced | 4 год |
| 08 | [Розподілені системи](Level-3-Advanced/Module-08-Distributed/lecture.md) | Advanced | 4 год |
| 08b | [Продуктивність та Діагностика](Level-3-Advanced/Module-08b-Performance/lecture.md) | Advanced | 3 год |
| 09 | [Enterprise Copilot](Level-3-Advanced/Module-09-Enterprise/lecture.md) | Advanced | 4 год |

### Матеріали для практики

| Файл | Призначення |
|------|------------|
| [Шпаргалка](Exercises/cheat-sheet.md) | Гарячі клавіші, команди, шаблони промптів — роздрукуй! |
| [Бібліотека промптів](Exercises/prompt-library.md) | ~30 готових промптів по категоріях — скопіюй і використовуй |
| [Антипатерни](Exercises/anti-patterns.md) | Що НЕ треба робити — з поясненнями і виправленнями |
| [Лабораторні роботи](Exercises/all-labs.md) | 9 labs + 3 challenger tasks |
| [Фінальний проєкт](Exercises/capstone-project.md) | MiniShop API — наскрізне завдання |
| [Шаблон copilot-instructions](Exercises/copilot-instructions-template.md) | Скопіюй у свій проєкт |
| [Розклад навчання](Exercises/training-schedule.md) | 3 варіанти (інтенсив / блоки / щотижня) |
