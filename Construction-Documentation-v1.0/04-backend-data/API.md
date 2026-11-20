# API-001 — API Specification

**Проект:** «Братья» / եղբայրներ  
**Документ:** API-001  
**Версия:** 0.3 — Consolidated Specification  
**Статус:** Draft — Consolidated API Specification  
**Дата:** 20.11.2026  
**Runtime:** Cloudflare Workers  
**Database:** Neon PostgreSQL  
**Validation:** Zod  
**API Version:** v1  
**Base Path:** `/api/v1/`

> **Consolidation note (v0.3):** This document supersedes the earlier combined API/Database draft and API-001 v0.2 Technical Consistency Update. API contracts are the single source for Public API, Telegram Webhooks, Internal API, application-service boundaries, security, idempotency, Outbox/Queues, notifications and acceptance criteria. Database entities, relationships, indexes, constraints and SQL/Drizzle details belong exclusively to **DATA-001**. Persistence behavior is referenced here without duplicating the complete schema or implying an existing Drizzle implementation. Earlier future deals/payments/commissions concepts are roadmap items to evaluate in DATA-001/business planning—not approved or implemented API endpoints.

---

## 1. Назначение документа

Документ определяет API-контракты для сайта строительной компании «Братья», Telegram-ботов и внутренних операций системы.

API обеспечивает:

- Приём заявок с публичного сайта.
- Валидацию и защиту входящих запросов.
- Сохранение заявок и атрибуции.
- Управление заявками через Telegram Lead Management Bot.
- Приём событий от Incident & Operations Bot и систем мониторинга.
- Управление напоминаниями и заметками.
- Получение операционной статистики.
- Обработку уведомлений через Transactional Outbox и Cloudflare Queues.
- Проверку здоровья API и инфраструктуры.

## 2. Архитектурные границы API

API разделяется на три логические зоны.

### 2.1. Public API

Предназначен для публичного сайта и клиентской формы.

Base path:

`/api/v1/`

Endpoints:

- `POST /api/v1/leads`
- `GET /api/v1/health`

Особенности:

- Доступен без пользовательской авторизации.
- Валидация входящих данных через Zod.
- Turnstile для защиты формы.
- Rate limiting.
- Idempotency.
- Серверная обработка данных.
- PostgreSQL — источник истины.

### 2.2. Telegram Webhooks

Предназначен для получения обновлений от двух Telegram-ботов.

Endpoints:

- `POST /api/v1/webhooks/telegram/leads`
- `POST /api/v1/webhooks/telegram/incident`

Особенности:

- Отдельный Telegram Bot Token для каждого бота.
- Проверка webhook secret.
- Проверка Telegram User ID и разрешений для команд, выполняющих операции от имени пользователя.
- Проверка Telegram Update ID.
- Идемпотентная обработка обновлений.
- Запрет прямого доступа к PostgreSQL из ботов.

Webhook authentication и user authorization — разные уровни защиты.

### 2.3. Internal API

Предназначен для внутренних операций приложения и Telegram-ботов.

Base path:

`/api/v1/internal/`

Lead Management:

- `GET /api/v1/internal/leads`
- `GET /api/v1/internal/leads/:leadId`
- `PATCH /api/v1/internal/leads/:leadId/status`
- `POST /api/v1/internal/leads/:leadId/notes`
- `POST /api/v1/internal/leads/:leadId/reminders`
- `GET /api/v1/internal/stats`

Incident & Operations:

- `GET /api/v1/internal/incidents`
- `GET /api/v1/internal/incidents/:incidentId`
- `POST /api/v1/internal/incidents`
- `PATCH /api/v1/internal/incidents/:incidentId/acknowledge`
- `PATCH /api/v1/internal/incidents/:incidentId/resolve`
- `GET /api/v1/internal/operations/status`

Internal API не предназначен для браузерного публичного доступа.

## 3. Общие правила API

### 3.1. Формат данных

- Request/Response: JSON.
- Encoding: UTF-8.
- Content-Type: `application/json`.
- API versioning: `/api/v1/`.
- Даты: ISO 8601 UTC.
- Идентификаторы передаются как строки.
- Все пользовательские входные данные проходят серверную валидацию.

### 3.2. Общий формат успешного ответа

```json
{
  "success": true,
  "data": {},
  "message": "Operation completed"
}
```

### 3.3. Общий формат ошибки

```json
{
  "success": false,
  "error": {
    "code": "VALIDATION_ERROR",
    "message": "Invalid request",
    "details": []
  }
}
```

Правила:

- `code` — стабильный машинный код.
- `message` — безопасное описание ошибки.
- `details` — необязательная информация о валидации.
- Внешнему клиенту запрещено возвращать SQL, stack traces, секреты и внутренние диагностические данные.

### 3.4. HTTP Status Codes

- `200 OK` — успешное выполнение.
- `201 Created` — создан ресурс.
- `202 Accepted` — запрос принят для асинхронной обработки.
- `400 Bad Request` — некорректный запрос.
- `401 Unauthorized` — отсутствует или недействительна аутентификация.
- `403 Forbidden` — недостаточно прав.
- `404 Not Found` — ресурс не найден.
- `409 Conflict` — конфликт состояния или idempotency.
- `422 Unprocessable Entity` — ошибка бизнес-валидации.
- `429 Too Many Requests` — превышен лимит запросов.
- `500 Internal Server Error` — внутренняя ошибка.
- `503 Service Unavailable` — временная недоступность сервиса.

## 4. Public API

### 4.1. Create Lead

`POST /api/v1/leads`

Создаёт новую заявку от потенциального клиента.

Headers:

```http
Content-Type: application/json
Idempotency-Key: <unique-key>
```

Request:

```json
{
  "name": "Aram",
  "phone": "+374XXXXXXXX",
  "email": "client@example.com",
  "service": "renovation",
  "preferredContact": "whatsapp",
  "message": "I need a full apartment renovation.",
  "locale": "en",
  "privacyConsent": true,
  "turnstileToken": "TURNSTILE_TOKEN",
  "attribution": {
    "landingPage": "/en/services/renovation",
    "referrer": "https://example.com",
    "utmSource": "google",
    "utmMedium": "cpc",
    "utmCampaign": "renovation-2026",
    "utmContent": "banner-a",
    "utmTerm": "apartment-renovation",
    "locale": "en"
  }
}
```

Обязательные поля:

- `name`
- `phone`
- `service`
- `preferredContact`
- `locale`
- `privacyConsent`
- `turnstileToken`

Необязательные:

- `email`
- `message`
- `attribution`

Допустимые значения:

`locale`:

- `hy`
- `ru`
- `en`

`preferredContact`:

- `whatsapp`
- `telegram`
- `phone`
- `email`

`service` — значение из утверждённого каталога услуг. Начальный перечень:

- `renovation`
- `house-construction`
- `interior-finishing`
- `interior-design`
- `facade`
- `roofing`
- `commercial`

Окончательный каталог должен соответствовать SRS-001 и фактической модели услуг сайта.

Response `201 Created`:

```json
{
  "success": true,
  "data": {
    "leadId": "RESOURCE_ID",
    "status": "NEW"
  },
  "message": "Your request has been submitted successfully."
}
```

Ошибки:

- `400 VALIDATION_ERROR`
- `403 TURNSTILE_VERIFICATION_FAILED`
- `409 IDEMPOTENCY_CONFLICT`
- `429 RATE_LIMIT_EXCEEDED`
- `500 INTERNAL_ERROR`
- `503 SERVICE_UNAVAILABLE`

### 4.2. Обработка Create Lead

Последовательность:

1. Проверить HTTP method и Content-Type.
2. Проверить размер тела запроса.
3. Выполнить Zod validation.
4. Проверить `privacyConsent`.
5. Проверить Turnstile token на сервере.
6. Проверить rate limit.
7. Проверить Idempotency-Key.
8. Выполнить DB-транзакцию:
   - Создать `leads`.
   - Создать `lead_attribution`, если передана атрибуция.
   - Создать `lead_events` с `LEAD_CREATED`.
   - Создать `notification_outbox`.
   - Сохранить результат idempotency.
9. Завершить DB-транзакцию.
10. Вернуть клиенту подтверждение создания заявки.
11. Передать Outbox-событие в асинхронную обработку.

Внешние уведомления не отправляются внутри DB-транзакции.

### 4.3. Health Check

`GET /api/v1/health`

Назначение — базовая проверка доступности API.

Response `200 OK`:

```json
{
  "success": true,
  "data": {
    "status": "healthy",
    "service": "construction-api",
    "timestamp": "2026-11-19T12:00:00Z"
  }
}
```

Health endpoint не должен раскрывать:

- DB connection strings.
- Environment secrets.
- Внутренние IP и инфраструктурные детали.
- Stack traces.
- Технические сведения, пригодные для атаки.

Подробная проверка зависимостей относится к защищённому Internal Operations API.

## 5. Telegram Webhooks

### 5.1. Общая архитектура

Используются два независимых бота:

**Lead Management Bot**

- Управление заявками.
- Изменение статусов.
- Заметки.
- Напоминания.
- Статистика.

**Incident & Operations Bot**

- Получение уведомлений об инцидентах.
- Подтверждение инцидентов.
- Разрешение инцидентов.
- Получение состояния инфраструктуры.

У каждого бота:

- Отдельный Telegram Bot Token.
- Отдельный webhook secret.
- Отдельная конфигурация.
- Отдельный scope разрешений.
- Отдельный обработчик команд.

### 5.2. Lead Management Webhook

`POST /api/v1/webhooks/telegram/leads`

Принимает Telegram Update для Lead Management Bot.

Headers:

```http
Content-Type: application/json
X-Telegram-Bot-Api-Secret-Token: <LEADS_WEBHOOK_SECRET>
```

Пример Update:

```json
{
  "update_id": 123456789,
  "message": {
    "message_id": 100,
    "from": {
      "id": 987654321,
      "is_bot": false
    },
    "chat": {
      "id": 987654321,
      "type": "private"
    },
    "text": "/leads"
  }
}
```

Обработка:

1. Проверить webhook secret.
2. Проверить структуру Update.
3. Проверить `update_id`.
4. Проверить Telegram User ID для команд, требующих авторизации.
5. Проверить `bot_user_permissions`.
6. Зарегистрировать Update.
7. Выполнить команду через внутренний application service.
8. Сохранить результат обработки.
9. Вернуть Telegram HTTP response.

Неизвестный Telegram User ID не может выполнять защищённые команды.

### 5.3. Incident Bot Webhook

`POST /api/v1/webhooks/telegram/incident`

Headers:

```http
Content-Type: application/json
X-Telegram-Bot-Api-Secret-Token: <INCIDENT_WEBHOOK_SECRET>
```

Обработка аналогична Lead Webhook, но применяется scope `INCIDENTS`.

Команды и действия ограничиваются разрешениями для Incident & Operations Bot.

### 5.4. Webhook Security

Обязательные требования:

- Использовать HTTPS.
- Устанавливать webhook secret через Telegram Bot API.
- Хранить секреты в Cloudflare Worker Secrets.
- Не хранить Telegram Bot Tokens в репозитории.
- Не записывать секреты в логи.
- Сравнивать webhook secret безопасным способом.
- Проверять размер и структуру Update.
- Использовать allowlist Telegram User ID.
- Разделять права двух ботов.
- Не принимать команды от неизвестных пользователей.

Webhook secret подтверждает подлинность отправителя Update. Он не доказывает, что пользователь имеет право менять заявку или управлять инцидентом.

## 6. Internal API — Lead Management

Все endpoints этого раздела требуют авторизации и проверки permissions.

### 6.1. Получение списка заявок

`GET /api/v1/internal/leads`

Query parameters:

- `status` — фильтр по статусу.
- `limit` — количество записей.
- `cursor` — курсор пагинации.
- `from` — дата начала периода.
- `to` — дата окончания периода.

Пример:

```http
GET /api/v1/internal/leads?status=NEW&limit=20
```

Response:

```json
{
  "success": true,
  "data": {
    "items": [
      {
        "id": "RESOURCE_ID",
        "name": "Aram",
        "phone": "+374XXXXXXXX",
        "service": "renovation",
        "preferredContact": "whatsapp",
        "locale": "en",
        "status": "NEW",
        "createdAt": "2026-11-19T12:00:00Z"
      }
    ],
    "nextCursor": null
  }
}
```

Требования:

- Пагинация обязательна.
- По умолчанию новые заявки отображаются первыми.
- Не возвращать внутренние технические поля.
- Ограничивать доступ по permissions.
- Не включать полное содержимое заявки в статистические ответы.

### 6.2. Получение заявки

`GET /api/v1/internal/leads/:leadId`

Возвращает:

- Основные данные заявки.
- Текущий статус.
- Атрибуцию.
- Историю событий.
- Заметки.
- Напоминания.

Response:

```json
{
  "success": true,
  "data": {
    "lead": {
      "id": "RESOURCE_ID",
      "name": "Aram",
      "phone": "+374XXXXXXXX",
      "status": "NEW",
      "service": "renovation",
      "locale": "en"
    },
    "attribution": {},
    "events": [],
    "notes": [],
    "reminders": []
  }
}
```

### 6.3. Изменение статуса

`PATCH /api/v1/internal/leads/:leadId/status`

Request:

```json
{
  "status": "CONTACTED"
}
```

Response:

```json
{
  "success": true,
  "data": {
    "leadId": "RESOURCE_ID",
    "status": "CONTACTED"
  }
}
```

Допустимые статусы:

- `NEW`
- `CONTACTED`
- `QUALIFIED`
- `ESTIMATE_SENT`
- `WON`
- `LOST`
- `SPAM`

Операция должна выполняться транзакционно:

1. Проверить разрешения пользователя.
2. Проверить существование заявки.
3. Проверить допустимость перехода.
4. Изменить статус.
5. Создать `lead_events`.
6. Обновить или отменить ненужные напоминания.
7. Commit.

### 6.4. Добавление заметки

`POST /api/v1/internal/leads/:leadId/notes`

Request:

```json
{
  "note": "Client requested an estimate for the apartment renovation."
}
```

Response:

```json
{
  "success": true,
  "data": {
    "noteId": "RESOURCE_ID",
    "leadId": "RESOURCE_ID"
  }
}
```

Заметка сохраняется в `lead_notes`, а событие `NOTE_ADDED` — в `lead_events`.

### 6.5. Создание напоминания

`POST /api/v1/internal/leads/:leadId/reminders`

Request:

```json
{
  "reminderType": "MANUAL_FOLLOW_UP",
  "scheduledAt": "2026-11-20T10:00:00Z"
}
```

Response:

```json
{
  "success": true,
  "data": {
    "reminderId": "RESOURCE_ID",
    "status": "PENDING"
  }
}
```

Требования:

- Проверка даты и времени.
- Проверка статуса заявки.
- Создание записи в `lead_reminders`.
- Сохранение события в `lead_events`.
- Защита от повторного выполнения.
- Атомарный захват Scheduler при обработке.

### 6.6. Статистика

`GET /api/v1/internal/stats`

Возвращает агрегированные данные:

- Общее количество заявок.
- Количество новых заявок.
- Количество заявок по статусам.
- Количество заявок по услугам.
- Количество заявок за период.
- Количество выигранных и потерянных заявок.

Персональные данные клиентов не включаются в статистический ответ.

## 7. Internal API — Incident & Operations

### 7.1. Получение списка инцидентов

`GET /api/v1/internal/incidents`

Query parameters:

- `status`
- `severity`
- `environment`
- `limit`
- `cursor`

### 7.2. Получение инцидента

`GET /api/v1/internal/incidents/:incidentId`

Возвращает:

- Основные сведения об инциденте.
- Severity.
- Environment.
- Source.
- First/last seen.
- Историю событий.
- Статус обработки.

Диагностические данные должны быть очищены от секретов и персональных данных.

### 7.3. Регистрация инцидента

`POST /api/v1/internal/incidents`

Request:

```json
{
  "source": "sentry",
  "externalId": "INCIDENT_EXTERNAL_ID",
  "fingerprint": "NORMALIZED_FINGERPRINT",
  "environment": "production",
  "severity": "CRITICAL",
  "title": "Lead API unavailable",
  "summary": "Lead API health checks failed."
}
```

Response:

```json
{
  "success": true,
  "data": {
    "incidentId": "RESOURCE_ID",
    "status": "OPEN"
  }
}
```

Правила:

- Повторные события одного инцидента не создают бесконечные дубликаты.
- При наличии `externalId` применяется deduplication по source, externalId и environment.
- При отсутствии externalId используется fingerprint и environment.
- Новое событие обновляет `last_seen_at`.
- Критический инцидент запускает прямой alert path.
- AI-анализ выполняется независимо от прямого уведомления.

### 7.4. Подтверждение инцидента

`PATCH /api/v1/internal/incidents/:incidentId/acknowledge`

Request:

```json
{
  "comment": "Investigating the issue."
}
```

Действия:

- Изменить статус на `ACKNOWLEDGED`.
- Сохранить `acknowledged_at`.
- Создать `incident_events`.
- Сохранить инициатора.

### 7.5. Разрешение инцидента

`PATCH /api/v1/internal/incidents/:incidentId/resolve`

Request:

```json
{
  "comment": "Service restored and health checks passed."
}
```

Действия:

- Изменить статус на `RESOLVED`.
- Сохранить `resolved_at`.
- Создать `incident_events`.
- Сохранить инициатора.

### 7.6. Статус операций

`GET /api/v1/internal/operations/status`

Предоставляет защищённую информацию о состоянии:

- Public API.
- Database connectivity.
- Outbox processing.
- Queue consumers.
- Notification delivery.
- Telegram webhook processing.
- Scheduler.
- Incident processing.

Не возвращает секреты и connection strings.

## 8. Authentication & Authorization

### 8.1. Public API

Не требует пользовательской авторизации.

Защита:

- Turnstile.
- Rate limiting.
- Request validation.
- Idempotency.
- Server-side processing.

### 8.2. Telegram Webhooks

Проверяются:

1. Webhook secret соответствующего бота.
2. Формат Telegram Update.
3. Telegram User ID для защищённых команд.
4. Наличие соответствующих permissions.

### 8.3. Internal API

Internal API не должен быть доступен анонимно.

Для MVP применяется серверная модель авторизации:

- Telegram User ID проверяется по `bot_users`.
- Права проверяются по `bot_user_permissions`.
- Lead Bot получает scope `LEADS`.
- Incident Bot получает scope `INCIDENTS`.
- Команды одного бота не должны автоматически давать доступ к другому.

Вызовы Internal API выполняются через доверенный серверный application layer либо защищённый внутренний механизм авторизации.

Нельзя считать знание URL или webhook secret достаточным правом на выполнение бизнес-операции.

### 8.4. Authorization failure

- `401` — отсутствует или недействительна аутентификация.
- `403` — пользователь не имеет необходимых permissions.
- `404` — ресурс не найден или не должен быть раскрыт вызывающей стороне.

## 9. Rate Limiting и Anti-Spam

### Public API

`POST /api/v1/leads`:

- Rate limiting по IP и дополнительным сигналам защиты.
- Cloudflare Turnstile.
- Ограничение размера request body.
- Zod validation.
- Ограничение длины текстовых полей.
- Нормализация телефонного номера.
- Idempotency-Key.

Конкретные числовые лимиты должны быть заданы конфигурацией окружения и проверены нагрузочными тестами.

### Telegram Webhooks

- Проверка webhook secret.
- Проверка Update ID.
- Ограничение размера запроса.
- Защита от повторной обработки.
- Ограничение частоты выполнения чувствительных команд.

## 10. Idempotency

Idempotency применяется для:

- Создания заявки.
- Обработки Telegram Updates.
- Доставки уведомлений.
- Автоматических напоминаний.
- Регистрации и обновления инцидентов.

### 10.1. Public API

Ключ передаётся через `Idempotency-Key`.

Правила:

- Одинаковый ключ и одинаковый request hash возвращают ранее сохранённый результат.
- Одинаковый ключ с другим request hash возвращает `409 IDEMPOTENCY_CONFLICT`.
- Проверка и сохранение результата выполняются транзакционно.

### 10.2. Telegram Updates

Уникальность Update определяется комбинацией:

`telegram_update_id + bot_scope`

### 10.3. Notifications

Consumer должен безопасно переносить повторную доставку сообщения.

Внешние провайдеры могут не гарантировать exactly-once delivery. Система должна обеспечивать идемпотентность на своей стороне и фиксировать результаты отправки.

## 11. Transactional Outbox и Queues

### 11.1. Общий процесс

```text
Public API / Internal API
          |
          v
   PostgreSQL Transaction
          |
          +--> Business Data
          |
          +--> lead_events
          |
          +--> notification_outbox
                    |
                    v
              Outbox Dispatcher
                    |
                    v
             Cloudflare Queues
                    |
                    v
                 Consumer
                    |
          +---------+----------+
          |                    |
          v                    v
        Resend            Telegram Bot API
```

### 11.2. Queue responsibilities

Логические очереди:

- `lead-notifications` — уведомления о заявках.
- `operations-events` — события операций и инцидентов.

`lead-commands` не включается в обязательный MVP: изменения статуса, заметки и создание напоминаний выполняются синхронно через Internal API и PostgreSQL.

### 11.3. Retry policy

Согласованная политика OPS-001:

- До 5 попыток, включая первоначальную.
- Задержки повторов: 1, 5, 15 и 60 минут.
- После исчерпания попыток — DLQ.
- Reconciliation — каждые 5 минут.

Фактическая реализация должна учитывать доступные настройки Cloudflare Queues, retry semantics и поведение Consumer. Указанные интервалы являются целевой политикой, а не утверждением о готовой конфигурации платформы.

### 11.4. Direct Alert и AI Analysis

Для критических инцидентов действуют два независимых пути:

**Direct Alert**

Monitoring → Worker → Telegram Incident Bot.

Цель — немедленно сообщить владельцу о критическом событии.

**AI Analysis**

Monitoring → AI Agent → диагностический результат → Telegram.

Цель — собрать и объяснить техническую информацию.

Недоступность AI Agent не должна блокировать Direct Alert.

## 12. Error Handling

Стабильные коды ошибок:

- `VALIDATION_ERROR`
- `TURNSTILE_VERIFICATION_FAILED`
- `RATE_LIMIT_EXCEEDED`
- `IDEMPOTENCY_CONFLICT`
- `UNAUTHORIZED`
- `FORBIDDEN`
- `RESOURCE_NOT_FOUND`
- `INVALID_STATUS_TRANSITION`
- `DUPLICATE_UPDATE`
- `DATABASE_ERROR`
- `QUEUE_DELIVERY_ERROR`
- `NOTIFICATION_DELIVERY_ERROR`
- `INTERNAL_ERROR`
- `SERVICE_UNAVAILABLE`

Требования:

- Единый error handler.
- Централизованное логирование.
- Разделение operational и public error messages.
- Correlation ID для трассировки запросов.
- Запрет на возврат stack traces клиенту.
- Sanitization логов.

## 13. Observability

Каждый запрос должен иметь correlation/request ID.

Необходимо отслеживать:

- Request count.
- Error rate.
- API latency.
- Database latency.
- Turnstile failures.
- Rate-limit events.
- Outbox backlog.
- Queue delivery failures.
- Consumer retries.
- DLQ count.
- Telegram webhook failures.
- Notification delivery status.
- Incident alert delivery.

Для наблюдаемости используются Cloudflare Observability, Sentry и Telegram Operations Bot согласно OPS-001.

## 14. API Security Requirements

- HTTPS only.
- Strict input validation.
- Output sanitization.
- Rate limiting.
- Turnstile verification.
- Least privilege.
- Secret isolation by environment.
- Separate bot tokens and webhook secrets.
- Telegram User ID allowlist.
- Permission checks.
- Request size limits.
- Idempotency.
- SQL parameterization через Drizzle.
- No direct DB access from frontend.
- No secrets in logs.
- No destructive production operations by AI Agent.
- No public exposure of Internal API.

## 15. Testing Requirements

### Public API

- Valid lead creation.
- Invalid request validation.
- Turnstile failure.
- Rate limiting.
- Idempotency.
- DB transaction rollback.
- Outbox creation.
- Missing UTM values.
- Localization.

### Telegram Webhooks

- Valid webhook secret.
- Invalid webhook secret.
- Unknown Telegram User ID.
- Missing permissions.
- Duplicate Update ID.
- Invalid Update payload.
- Separate bot scopes.

### Internal API

- Authorized access.
- Unauthorized access.
- Permission boundary checks.
- Lead list pagination.
- Lead status transitions.
- Note creation.
- Reminder creation.
- Incident acknowledgement.
- Incident resolution.

### Queue / Notifications

- Outbox dispatch.
- Duplicate message delivery.
- Retry.
- DLQ.
- Reconciliation.
- Direct Alert without AI.
- AI Analysis failure.

## 16. Acceptance Criteria

API-001 считается реализованным, когда:

- Public API отделён от Internal API.
- Telegram Webhooks имеют отдельные endpoints.
- Каждый бот использует собственные credentials.
- Webhook secret проверяется.
- Telegram User ID проверяется отдельно.
- Permissions разделены по bot scope.
- Public Lead API сохраняет прежний контракт.
- Turnstile и rate limiting работают.
- Idempotency проверена.
- Все бизнес-изменения сохраняются в PostgreSQL.
- Outbox создаётся в одной транзакции с бизнес-данными.
- Queue Consumer идемпотентен.
- Напоминания защищены от конкурентной обработки.
- Direct Alert не зависит от AI Agent.
- Internal API не доступен публично.
- Все критические сценарии покрыты тестами.

## 17. Open Decisions

| ID | Вопрос | Статус |
| --- | --- | --- |
| API-OD-001 | Какой конкретный механизм внутренней авторизации будет использоваться между Worker handlers и application services? | To finalize |
| API-OD-002 | Какие числовые rate limits применяются в production? | To configure |
| API-OD-003 | Какой окончательный каталог service codes? | To align with SRS |
| API-OD-004 | Какой формат correlation ID используется между API, Outbox и Queues? | To specify |
| API-OD-005 | Какой формат event schema и её versioning? | To specify |
| API-OD-006 | Какой фактический Neon driver и connection strategy совместимы с Cloudflare Workers? | Compatibility test |
| API-OD-007 | Какой конкретный механизм проверки и хранения Telegram webhook secrets будет применяться в каждом окружении? | To implement |

## 18. Consolidation & Document Boundaries

### 18.1 Single source of truth
- **API-001 v0.3** defines endpoint contracts, request/response behavior, validation, authentication, authorization, errors, idempotency, API workflows, webhook handling, queue/notification integration and acceptance criteria.
- **DATA-001** defines persistence: entities, columns, types, primary/foreign keys, constraints, indexes, relationships, retention, and SQL/Drizzle schema.
- API-001 does not maintain a second full SQL DDL or independent database schema. Persisted fields are governed by DATA-001.
- Conflicts between API behavior and DATA-001 must be recorded and resolved through a versioned change, not inferred silently.

### 18.2 Earlier combined API/Database draft
The earlier draft covered Public Lead API plus database model and future deals/payments/commissions. Database detail is moved to/maintained in DATA-001. Deals/payments/commissions remain roadmap concepts unless explicitly accepted into MVP scope. Their presence in the earlier draft does not mean tables or endpoints are approved or implemented.

### 18.3 Consolidated scope
1. Public Lead API and Health Check.
2. Telegram webhooks for Lead Management and Incident & Operations bots.
3. Internal Lead Management operations.
4. Internal Incident & Operations endpoints.
5. Shared application-service boundaries and direct in-Worker invocation.
6. Authentication, authorization, rate limiting, anti-spam, idempotency.
7. Transactional Outbox, Cloudflare Queues, retry/DLQ and reconciliation.
8. Notification delivery and Direct Alert / AI Analysis separation.
9. Correlation, logging, test requirements, acceptance criteria and open decisions.

### 18.4 Implementation status
Repository, runtime code, PostgreSQL database and actual Drizzle schema have not been created. Endpoint contracts and persistence references are specifications only. Drizzle schema validation and migration review are implementation-stage checkpoints.

## 19. Связанные документы

- BRD-001 — Business Requirements.
- SRS-001 — Software Requirements Specification.
- SAD-001 — System Architecture.
- DATA-001 — Data Model & Database Specification.
- ADR-001 — Architecture Decision Records.
- TECH-001 — Telegram Bots Technical Architecture.
- OPS-001 — Operations & Monitoring.
- DEPLOY-001 — Deployment & CI/CD.
- TEST-001 — Testing Strategy.
- IMPLEMENTATION-001 — Implementation Plan.

---

## Change Log

| Версия | Дата | Изменения |
| --- | --- | --- |
| 0.1 | 2026-11-17 | Public Lead API, Health Check, Zod, Turnstile, rate limiting, idempotency |
| 0.2 | 2026-11-19 | Public API, Telegram Webhooks и Internal API; два бота, webhook auth, scoped permissions, Incident API, reminders, Outbox/Queues, Direct Alert |
| 0.3 | 2026-11-20 | Consolidated edition: единый API-источник; database/DDL вынесены в DATA-001; legacy deals/payments/commissions отмечены как roadmap/TBD; устранено дублирование и конфликт версий |
