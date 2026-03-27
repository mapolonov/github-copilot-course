# Модуль 06b: Copilot CLI та GitHub Actions — Copilot за межами редактора
## Рівень: Intermediate → Advanced | Тривалість: 2.5 години

---

## Цілі модуля

- Використовувати `gh copilot` у терміналі для пояснення та генерації команд
- Інтегрувати Copilot у CI/CD pipeline через GitHub Actions
- Автоматизувати Code Review через Copilot у Pull Requests
- Використовувати `copilot-cli` для DevOps задач

---

## 1. Copilot CLI — Copilot у терміналі

### 1.1 Встановлення та налаштування

```bash
# Передумова: gh CLI встановлено та авторизовано
gh auth login

# Встановити Copilot розширення для gh CLI
gh extension install github/gh-copilot

# Перевірка
gh copilot --version

# Зручні псевдоніми (додати в PowerShell $PROFILE або .bashrc):
# PowerShell:
Set-Alias ghce 'gh copilot explain'
Set-Alias ghcs 'gh copilot suggest'

# Bash/Zsh:
alias ghce='gh copilot explain'
alias ghcs='gh copilot suggest'
```

### 1.2 `gh copilot explain` — пояснення команд

```bash
# Що робить ця незрозуміла команда?
gh copilot explain "git rebase -i HEAD~5"

# Пояснення Docker команди
gh copilot explain "docker run --rm -v $(pwd):/app -w /app mcr.microsoft.com/dotnet/sdk:8.0 dotnet test"

# Пояснення SQL
gh copilot explain "SELECT * FROM pg_stat_activity WHERE state = 'active' AND wait_event_type = 'Lock'"

# Пояснення EF Core migration команди
gh copilot explain "dotnet ef migrations add AddOrderIndex --project src/Infrastructure --startup-project src/Api"
```

**Реальний use case:** Новий розробник отримав скрипт від DevOps. Замість питати колег — `gh copilot explain "..."`.

### 1.3 `gh copilot suggest` — генерація команд

```bash
# Описуєш що хочеш — отримуєш команду
gh copilot suggest "watch all .cs files and run dotnet test on change"

gh copilot suggest "find all TODO comments in .cs files and write to todo.txt"

gh copilot suggest "create a self-signed SSL certificate for localhost development"

gh copilot suggest "bulk rename all migration files to add timestamp prefix"

gh copilot suggest "show me tables with most disk usage in PostgreSQL"
```

**Інтерактивний режим:** після пропозиції Copilot запитає:
- `Run this command` — виконати
- `Revise the command` — уточнити
- `Copy the command` — скопіювати
- `Explain the command` — пояснити спочатку

### 1.4 Практичні .NET/DevOps сценарії

```bash
# Docker + .NET

gh copilot suggest "build .NET 8 docker image and tag with current git commit hash"
# → docker build -t myapp:$(git rev-parse --short HEAD) .

gh copilot suggest "run all xUnit tests in docker container and output junit xml report"

gh copilot suggest "scan docker image for vulnerabilities using trivy"


# Database

gh copilot suggest "create EF Core migration for adding index on Orders.CustomerId"
# → dotnet ef migrations add AddOrderCustomerIdIndex -p src/Infrastructure -s src/Api

gh copilot suggest "rollback last EF Core migration in production without dropping data"

gh copilot suggest "export PostgreSQL schema to SQL file excluding data"


# Git

gh copilot suggest "squash all commits on current branch into one with interactive rebase"

gh copilot suggest "find which commit introduced a specific string in codebase"
# → git log -S "string_to_find" --all

gh copilot suggest "create git hook that runs dotnet format before every commit"
```

---

## 2. GitHub Actions — Copilot як учасник CI/CD

### 2.1 Автоматичний Code Review у Pull Requests

```yaml
# .github/workflows/copilot-review.yml
# Copilot автоматично коментує PR з code review

name: Copilot Code Review

on:
  pull_request:
    types: [opened, synchronize]
    paths:
      - 'src/**/*.cs'
      - '!src/**/*.Tests/**'

jobs:
  copilot-review:
    runs-on: ubuntu-latest
    permissions:
      pull-requests: write
      contents: read

    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 0

      - name: Get changed .cs files
        id: changed-files
        uses: tj-actions/changed-files@v44
        with:
          files: |
            src/**/*.cs

      - name: Copilot Review via gh CLI
        if: steps.changed-files.outputs.any_changed == 'true'
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
        run: |
          gh pr review ${{ github.event.pull_request.number }} \
            --comment \
            --body "$(gh copilot review --files '${{ steps.changed-files.outputs.all_changed_files }}')"
```

### 2.2 Workflow для .NET з Copilot-suggested покращеннями

```yaml
# .github/workflows/dotnet-ci.yml
name: .NET CI

on:
  push:
    branches: [main, develop]
  pull_request:
    branches: [main]

jobs:
  build-and-test:
    runs-on: ubuntu-latest

    services:
      postgres:
        image: postgres:16-alpine
        env:
          POSTGRES_PASSWORD: testpass
          POSTGRES_DB: testdb
        options: >-
          --health-cmd pg_isready
          --health-interval 10s
          --health-timeout 5s
          --health-retries 5
        ports:
          - 5432:5432

    steps:
      - uses: actions/checkout@v4

      - name: Setup .NET
        uses: actions/setup-dotnet@v4
        with:
          dotnet-version: '8.0.x'

      - name: Cache NuGet packages
        uses: actions/cache@v4
        with:
          path: ~/.nuget/packages
          key: ${{ runner.os }}-nuget-${{ hashFiles('**/*.csproj') }}
          restore-keys: |
            ${{ runner.os }}-nuget-

      - name: Restore
        run: dotnet restore

      - name: Build
        run: dotnet build --no-restore --configuration Release

      - name: Run Tests with Coverage
        run: |
          dotnet test --no-build \
            --configuration Release \
            --collect:"XPlat Code Coverage" \
            --results-directory ./coverage \
            --logger trx \
            --logger "console;verbosity=normal"
        env:
          ConnectionStrings__DefaultConnection: "Host=localhost;Database=testdb;Username=postgres;Password=testpass"

      - name: Coverage Report
        uses: danielpalme/ReportGenerator-GitHub-Action@5
        with:
          reports: 'coverage/**/coverage.cobertura.xml'
          targetdir: 'coverage/report'
          reporttypes: 'HtmlInline;Cobertura;MarkdownSummaryGithub'

      - name: Coverage Summary в PR
        uses: marocchino/sticky-pull-request-comment@v2
        if: github.event_name == 'pull_request'
        with:
          recreate: true
          path: coverage/report/SummaryGithub.md

      - name: Fail if coverage < 80%
        run: |
          COVERAGE=$(dotnet-coverage-threshold --threshold 80 --reports ./coverage)
          echo "Coverage: $COVERAGE%"
```

### 2.3 Security Scanning + Copilot Summary

```yaml
# .github/workflows/security.yml
name: Security Scan

on:
  schedule:
    - cron: '0 8 * * 1'  # Щопонеділка о 8:00
  push:
    branches: [main]

jobs:
  dependency-scan:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: .NET Vulnerability Scan
        run: |
          dotnet restore
          dotnet list package --vulnerable --include-transitive 2>&1 | tee vuln-report.txt
          
          if grep -q "has the following vulnerable packages" vuln-report.txt; then
            echo "::warning::Vulnerable packages found!"
            cat vuln-report.txt
          fi

      - name: Docker Image Scan (Trivy)
        uses: aquasecurity/trivy-action@master
        with:
          image-ref: ${{ env.IMAGE_NAME }}:latest
          format: 'sarif'
          output: 'trivy-results.sarif'
          severity: 'CRITICAL,HIGH'

      - name: Upload to GitHub Security tab
        uses: github/codeql-action/upload-sarif@v3
        with:
          sarif_file: 'trivy-results.sarif'

  codeql-analysis:
    runs-on: ubuntu-latest
    permissions:
      security-events: write
    steps:
      - uses: actions/checkout@v4
      - uses: github/codeql-action/init@v3
        with:
          languages: csharp
          queries: security-extended
      - uses: github/codeql-action/autobuild@v3
      - uses: github/codeql-action/analyze@v3
```

---

## 3. Copilot у PR Review процесі

### 3.1 GitHub Copilot Code Review (нативний)

У GitHub (Enterprise/Business) Copilot може автоматично ревьюювати PR:

```
1. Відкрити Pull Request на GitHub
2. В правому меню "Reviewers" → "Copilot" → "Request review"
3. Copilot проаналізує diff та залишить inline коментарі

Що Copilot перевіряє:
- Потенційні баги та edge cases
- Security вразливості (injection, auth)
- Performance проблеми
- Відсутня обробка помилок
- Порушення стилю (якщо налаштовано .github/copilot-instructions.md)
```

### 3.2 Кастомний PR Summary через Actions

```yaml
# .github/workflows/pr-summary.yml
name: PR Summary

on:
  pull_request:
    types: [opened]

jobs:
  generate-summary:
    runs-on: ubuntu-latest
    permissions:
      pull-requests: write
    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 0

      - name: Generate PR Description via Copilot
        env:
          GH_TOKEN: ${{ secrets.GITHUB_TOKEN }}
        run: |
          # Отримати diff
          git diff origin/${{ github.base_ref }}...HEAD -- '*.cs' > changes.diff
          
          # Попросити Copilot згенерувати summary (через gh API)
          SUMMARY=$(gh api \
            -X POST /repos/${{ github.repository }}/issues/${{ github.event.pull_request.number }}/comments \
            -f body="@github-copilot Summarize these changes in Ukrainian. Focus on: what business logic changed, what tests were added, any migration changes." \
          )
```

---

## 4. Автоматизація рутинних задач DevOps

### 4.1 Git Hooks з Copilot-перевірками

```bash
# .git/hooks/pre-commit (або через Husky для .NET)
# Copilot перевіряє що немає hardcoded secrets

#!/bin/bash

echo "🔍 Scanning for potential secrets..."

# Перевірка на типові паттерни secrets
SECRETS_FOUND=$(git diff --cached --diff-filter=ACM | grep -E \
  "(password|secret|api_key|token|private_key)\s*=\s*['\"][^'\"]{8,}" \
  --ignore-case)

if [ -n "$SECRETS_FOUND" ]; then
  echo "❌ Potential secrets detected in staged changes:"
  echo "$SECRETS_FOUND"
  echo ""
  echo "Use: gh copilot suggest 'how to store secrets safely in .NET'"
  exit 1
fi

echo "✅ No obvious secrets found"
```

### 4.2 PowerShell скрипти через Copilot Suggest

```powershell
# Використовуй gh copilot suggest для DevOps автоматизації
# Приклади що можна попросити:

# "Monitor memory usage of dotnet process every 5 seconds and alert if > 2GB"
# "Find all .cs files not covered by any test project"  
# "Rotate app secrets in Azure Key Vault and update AKS deployment"
# "Generate changelog from git log since last tag in markdown format"
# "Check all microservices health endpoints and report status"
```

---

## 5. Copilot для Infrastructure as Code

### 5.1 Docker + Docker Compose

```bash
# Prompts via Copilot Chat або Inline:

gh copilot suggest "multi-stage Dockerfile for .NET 8 API optimized for size"
# → Copilot згенерує оптимальний multi-stage build

gh copilot suggest "docker-compose for: .NET API + PostgreSQL + Redis + Seq (logging) + Grafana"
```

```dockerfile
# Результат типового multi-stage Dockerfile від Copilot:
FROM mcr.microsoft.com/dotnet/sdk:8.0 AS build
WORKDIR /src
COPY ["src/Api/Api.csproj", "src/Api/"]
COPY ["src/Application/Application.csproj", "src/Application/"]
# ... (Copilot продовжить)
RUN dotnet restore "src/Api/Api.csproj"
COPY . .
RUN dotnet publish "src/Api/Api.csproj" -c Release -o /app/publish --no-restore

FROM mcr.microsoft.com/dotnet/aspnet:8.0 AS runtime
WORKDIR /app
# Security: non-root user
RUN adduser --disabled-password --gecos "" appuser && chown -R appuser /app
USER appuser
COPY --from=build /app/publish .
EXPOSE 8080
ENTRYPOINT ["dotnet", "Api.dll"]
```

### 5.2 Kubernetes manifests

```bash
gh copilot suggest "K8s deployment for .NET 8 API with:
 - 3 replicas, rolling update strategy
 - resource limits: 512Mi memory, 500m CPU
 - liveness probe on /health/live, readiness on /health/ready  
 - secrets from K8s Secret (connection string, API keys)
 - configMap for non-sensitive settings"
```

---

## Практичне завдання (45 хв)

### Lab CLI-1: Copilot у терміналі

1. Встановити `gh copilot` розширення
2. Пояснити 5 незнайомих команд з цього списку:
   - `netstat -tlnp | grep :5432`
   - `git bisect start HEAD HEAD~20`
   - `dotnet trace collect --process-id $(pgrep -f "MyApp")`
   - `pg_dump -Fc --no-acl --no-owner -h localhost mydb > backup.dump`
   - `kubectl rollout undo deployment/my-api --to-revision=3`

3. Згенерувати 3 команди за описом для вашого поточного проєкту

### Lab CLI-2: GitHub Actions Pipeline

На основі шаблонів із цього модуля, налаштуйте workflow для демо-проєкту:
- Build + Test з PostgreSQL через Services
- Coverage звіт у PR
- Vulnerability scan на `dotnet list package --vulnerable`

---

## Підсумки модуля

| Інструмент | Найкращий для |
|-----------|--------------|
| `gh copilot explain` | DevOps команди, незнайомий bash/SQL |
| `gh copilot suggest` | Автоматизація, скрипти, DevOps задачі |
| GitHub Actions + Copilot | Автоматичний PR review, security scanning |
| Copilot CLI у pre-commit | Захист від secrets в коді |

**← Попередній модуль:** [Модуль 06: Рефакторинг](../Module-06-Refactoring/lecture.md)  
**↑ До програми:** [README](../../README.md)
