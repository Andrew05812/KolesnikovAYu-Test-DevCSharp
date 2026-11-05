# Vulnerability Tracker API

REST API на C# (ASP.NET Core, .NET 8) для учёта уязвимостей и запуска фоновых задач сканирования репозиториев.

Репозиторий: https://github.com/Andrew05812/KolesnikovAYu-Test-DevCSharp

## Что нужно для запуска

- .NET SDK 8.0 или новее (`dotnet --version`)

## Запуск

```
dotnet run --project src/VulnerabilityTracker.Api
```

Сервис слушает `http://localhost:5080`. 

Примеры запросов лежат в `src/VulnerabilityTracker.Api/requests.http` 

## Тесты

```
dotnet test
```

В тестах проверяются хранилище уязвимостей, валидация, менеджер сканирований (очередь, лимит параллельности, отмена, ошибки) и сами эндпоинты через `WebApplicationFactory`.

## Эндпоинты

| Метод | Путь | Что делает |
|-------|------|------------|
| POST | `/api/vulnerabilities` | Добавляет уязвимость, возвращает её id |
| GET | `/api/vulnerabilities/{id}` | Полная информация об уязвимости |
| POST | `/api/scans` | Ставит сканирование репозитория в очередь, возвращает id задачи |
| GET | `/api/scans/{id}` | Статус задачи, время последнего изменения, результат |
| POST | `/api/scans/{id}/cancel` | Отменяет сканирование (в очереди или уже идущее) |

### Добавление уязвимости

```
POST /api/vulnerabilities
{ "name": "SQL injection in login form", "severity": "High" }
```

`severity`: `Low`, `Medium`, `High`, `Critical`. Ответ `201 Created`:

```
{ "id": "6b0e0a52-0f8e-4c1c-9d0e-3c1f6f3b9a10" }
```

Дата добавления ставится сервером и возвращается при получении уязвимости (`createdAt`).

### Запуск сканирования

```
POST /api/scans
{ "repositoryUrl": "https://gitlab.com/some-group/some-project" }
```

Ответ `202 Accepted` с `{ "id": "..." }`. Сканирование идёт в фоне.

### Статус сканирования

`GET /api/scans/{id}` возвращает:

```
{
  "id": "...",
  "repositoryUrl": "https://gitlab.com/some-group/some-project",
  "status": "Completed",
  "updatedAt": "2025-01-01T10:00:10+00:00",
  "result": { "repositoryUrl": "...", "filesScanned": 87, "findings": [ ... ] },
  "error": null
}
```

Статусы: `Queued` (в очереди), `Running` (выполняется), `Completed` (завершена), `Failed` (ошибка, текст в `error`), `Canceled` (отменена). `result` заполнен только у завершённых задач.

### Отмена

`POST /api/scans/{id}/cancel` возвращает `id`, время отмены (`time`), признак успеха (`success`) и итоговый статус. Отменить можно задачу в очереди и уже выполняющуюся (её токен отмены срабатывает, слот освобождается). Для уже завершённых, упавших или отменённых задач `success` будет `false`.

## Настройки

Файл `src/VulnerabilityTracker.Api/appsettings.json`, секция `Scanning`:

| Параметр | По умолчанию | Смысл |
|----------|--------------|-------|
| `MaxParallelScans` | 2 | Сколько сканирований выполняется одновременно, остальные ждут в очереди со статусом `Queued` |
| `SimulatedDurationSeconds` | 10 | Сколько длится имитация сканирования |

Любой параметр можно переопределить переменной окружения, например:

```
set Scanning__MaxParallelScans=1
dotnet run --project src/VulnerabilityTracker.Api
```

## Где хранятся данные и файлы

Постоянного хранилища нет: уязвимости и задачи сканирования лежат только в памяти процесса и пропадают при остановке сервиса. Файлов сервис не создаёт. Логи пишутся в консоль (stdout).

## Логи и ошибки

Каждый запрос логируется одной строкой:

```
2026-01-01 10:00:00 info: VulnerabilityTracker.Api.Middleware.RequestLoggingMiddleware[0] POST /api/scans -> 202 (4 мс)
```

Также логируются постановка, запуск, завершение, ошибка и отмена сканирований.

Ошибки возвращаются в формате problem details (`application/problem+json`):

- `400` для некорректного JSON или невалидных полей (с описанием по каждому полю)
- `404` для неизвестного id
- `500` для непредвиденных ошибок, подробности пишутся в лог, наружу не отдаются

## Как устроено сканирование

Настоящего сканера в задании нет, поэтому `SimulatedScanner` ждёт `SimulatedDurationSeconds` секунд и возвращает набор находок, зависящий от ссылки. Ссылка, оканчивающаяся на `/broken` (например `https://gitlab.com/group/broken`), даёт статус `Failed`, так можно посмотреть обработку ошибки. Реальный сканер можно подключить, реализовав интерфейс `IRepositoryScanner`.

Лимит параллельности сделан через `SemaphoreSlim`: задача сразу создаётся со статусом `Queued`, ждёт свободный слот, затем переходит в `Running`. Переходы статусов защищены блокировкой, поэтому отмена и завершение не могут перезаписать друг друга.

## Структура

```
src/VulnerabilityTracker.Api
  Models/        модели и DTO
  Services/      хранилище, менеджер сканирований, сканер, валидация
  Middleware/    логирование запросов и обработка ошибок
  Program.cs     конфигурация и эндпоинты
tests/VulnerabilityTracker.Tests
```
