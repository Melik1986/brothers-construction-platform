# SAD-001 — System Architecture Specification

**Проект:** «Братья» / եղբայրներ  
**Документ:** SAD-001  
**Версия:** 0.2  
**Статус:** Draft — Architecture Consistency Update  
**Дата:** 19.11.2026

---

## 1. Назначение

SAD-001 описывает целевую архитектуру сайта строительной компании «Братья», публичного API, базы данных, системы уведомлений и операционной автоматизации.

Архитектура должна обеспечивать:

- Быстрый многоязычный сайт.
- Надёжный сбор клиентских заявок.
- Автоматическую обработку и уведомление о новых лидах.
- Управление заявками через Telegram.
- Мониторинг инфраструктуры и инцидентов.
- Возможность дальнейшего расширения без преждевременного усложнения MVP.

## 2. Архитектурные принципы

1. PostgreSQL — единственный источник истины для бизнес-данных.
2. Публичный сайт преимущественно статический.
3. Cloudflare Workers — основной runtime для API и серверной логики.
4. Application Services — единый слой бизнес-логики внутри Worker.
5. PostgreSQL-транзакции фиксируют бизнес-изменения и Outbox-события.
6. Cloudflare Queues обеспечивают асинхронную обработку.
7. Consumer должен быть идемпотентным.
8. Уведомления не выполняются внутри DB-транзакций.
9. Все компоненты используют единый Correlation ID.
10. Два Telegram-бота разделены по назначению, credentials и permissions.
11. AI Agent не имеет прямого доступа к PostgreSQL и не выполняет разрушительные production-операции.
12. Архитектура MVP не включает CRM, Admin Panel, VPS, Kubernetes, Redis и NestJS.

## 3. Технологический стек

| Слой | Технология |
| --- | --- |
| Frontend | Astro + TypeScript strict |
| Rendering | Static Site Generation |
| Interactive UI | React Islands |
| Styling | SCSS |
| Animation | GSAP selectively |
| Hosting | Cloudflare Workers + Static Assets |
| API Runtime | Cloudflare Workers |
| Database | Neon PostgreSQL |
| ORM | Drizzle ORM |
| Migrations | Drizzle Kit |
| Media Storage | Cloudflare R2 |
| Async Messaging | Cloudflare Queues |
| Email | Resend |
| Operations | Telegram Bot API |
| Anti-spam | Cloudflare Turnstile |
| CDN / DNS / WAF | Cloudflare |
| CI/CD | GitHub Actions |
| Monitoring | Sentry + Cloudflare Observability |
| AI Operations | Cursor Automations / Cloud Agent — candidate |

## 4. High-Level Architecture

```text
                    CLIENT
                      |
                      v
              CLOUDFLARE CDN / WAF
                      |
                      v
             ASTRO STATIC WEBSITE
                      |
              Interactive Islands
                      |
                      v
              CLOUDFLARE WORKER
                      |
          +-----------+------------+
          |                        |
          v                        v
      PUBLIC API              APPLICATION SERVICES
          |                        |
          +-----------+------------+
                      |
                      v
                NEON POSTGRESQL
                      |
          +-----------+------------+
          |                        |
          v                        v
      BUSINESS DATA          NOTIFICATION OUTBOX
                                   |
                                   v
                            OUTBOX DISPATCHER
                                   |
                                   v
                            CLOUDFLARE QUEUES
                                   |
                    +--------------+--------------+
                    |                             |
                    v                             v
             LEAD CONSUMER                 OPERATIONS CONSUMER
                    |                             |
              +-----+------+                +-----+------+
              |            |                |            |
              v            v                v            v
            RESEND      LEAD BOT        INCIDENT BOT   AI AGENT
```

Схема показывает логические компоненты. В MVP они могут быть реализованы в одном Cloudflare Worker deployment.

## 5. Frontend Architecture

### 5.1. Astro

Astro отвечает за:

- Статические страницы.
- Локализованные маршруты.
- SEO и GEO.
- Content Collections.
- Генерацию sitemap.
- Оптимизацию изображений.
- Серверную подготовку HTML.

### 5.2. React Islands

React применяется только для компонентов, которым необходима интерактивность:

- Lead Form.
- Interactive Gallery.
- Before/After Slider.
- Другие интерактивные компоненты по мере необходимости.

Основной контент должен оставаться доступным без обязательной загрузки клиентского JavaScript.

### 5.3. Localization

Поддерживаемые языки:

- Armenian — `hy`
- Russian — `ru`
- English — `en`

Основная структура URL:

- `/hy/`
- `/ru/`
- `/en/`

## 6. Worker Architecture

### 6.1. Общая модель

Один основной Cloudflare Worker обслуживает:

- Public API.
- Telegram Webhooks.
- Internal API.
- Application Services.
- Outbox Dispatcher.
- Queue Consumers.
- Scheduled Jobs.

Логическое разделение компонентов обязательно, но физическое разделение на несколько Workers в MVP не требуется.

### 6.2. Application Services

Application Services — центральный слой бизнес-логики.

Обработчики HTTP не должны самостоятельно реализовывать бизнес-операции.

Пример структуры:

```text
src/
  worker/
    index.ts
    router.ts

  api/
    public/
      leads.ts
      health.ts
    internal/
      leads.ts
      incidents.ts
      operations.ts
    webhooks/
      telegram-leads.ts
      telegram-incidents.ts

  application/
    leads/
      create-lead.ts
      update-lead-status.ts
      add-lead-note.ts
      create-reminder.ts
      get-leads.ts
    notifications/
      dispatch-outbox.ts
      process-notification.ts
    incidents/
      create-incident.ts
      acknowledge-incident.ts
      resolve-incident.ts
    operations/
      get-system-status.ts

  domain/
    leads/
    incidents/
    notifications/

  infrastructure/
    database/
    queues/
    telegram/
    email/
    monitoring/
    security/

  shared/
    correlation/
    errors/
    validation/
    logging/
```

### 6.3. Правило внутренних вызовов

HTTP handler вызывает соответствующий Application Service напрямую.

Пример:

```text
HTTP Request
     |
     v
Route Handler
     |
     v
Authentication / Validation
     |
     v
Application Service
     |
     v
Repository / Database
```

Запрещённый архитектурный паттерн:

```text
Worker HTTP Handler
        |
        v
HTTP request to own Worker URL
        |
        v
Another Worker HTTP Handler
```

Самовызовы через HTTP не используются для внутренней коммуникации. Это исключает лишние сетевые переходы и дублирование обработки авторизации.

HTTP остаётся границей между внешними системами и Worker.

## 7. Database Architecture

### 7.1. Neon PostgreSQL

PostgreSQL является источником истины для:

- Leads.
- Attribution.
- Lead Events.
- Notification Outbox.
- Idempotency Keys.
- Bot Users.
- Bot Permissions.
- Telegram Updates.
- Lead Notes.
- Lead Reminders.
- Notification Delivery Records.
- Incidents.
- Incident Events.

### 7.2. Database Driver

Совместимость выбранного PostgreSQL-драйвера с Cloudflare Workers и Neon подтверждена.

Принятые требования:

- Использовать серверное подключение.
- Не раскрывать connection strings.
- Разделять credentials по окружениям.
- Контролировать число соединений.
- Использовать безопасные параметризованные запросы через Drizzle.
- Учитывать ограничения runtime Cloudflare Workers.
- Проверять транзакционное поведение в staging.

### 7.3. ORM и migrations

- Drizzle ORM — работа с типизированной схемой.
- Drizzle Kit — миграции.
- SQL migrations — версия в Git.
- Squawk — SQL validation.
- Миграции запускаются отдельным CI/CD этапом.
- Worker не выполняет миграции при старте.

## 8. Correlation ID Architecture

### 8.1. Назначение

Единый Correlation ID используется для сквозной трассировки операции через все компоненты системы.

Он позволяет установить связь между:

- HTTP Request.
- Application Service.
- DB Transaction.
- Outbox Event.
- Queue Message.
- Consumer Execution.
- Email Notification.
- Telegram Notification.
- Error Logs.
- Sentry Events.

### 8.2. Жизненный цикл

```text
Incoming Request
       |
       v
Generate Correlation ID
       |
       v
Application Service
       |
       v
PostgreSQL Transaction
       |
       +----> Business Data
       |
       +----> Outbox Event
                    |
                    v
              Queue Message
                    |
                    v
                 Consumer
                    |
             +------+------+
             |             |
             v             v
           Resend      Telegram
```

### 8.3. Правила

1. Correlation ID создаётся на входной границе системы, если он отсутствует или не прошёл проверку.
2. Внутренние Application Services получают его через execution context.
3. При создании Outbox-записи Correlation ID сохраняется в событии.
4. Dispatcher передаёт его в Queue Message.
5. Consumer восстанавливает Correlation ID из сообщения.
6. Все последующие вызовы внешних провайдеров логируются с тем же ID.
7. Повторная обработка события сохраняет исходный Correlation ID.
8. Новые независимые операции получают новый Correlation ID.
9. Входящие Correlation ID от внешних источников нельзя без проверки считать доверенными.
10. Correlation ID не содержит персональных данных и секретов.

Формат:

`X-Correlation-ID: <opaque-unique-id>`

Предпочтительный формат генерации — UUID.

Для внутренних сообщений Correlation ID передаётся как отдельное поле envelope, а не только как HTTP header.

### 8.4. Queue Event Envelope

Пример:

```json
{
  "eventId": "EVENT_ID",
  "eventType": "LEAD_CREATED",
  "eventVersion": 1,
  "correlationId": "CORRELATION_ID",
  "aggregate": {
    "type": "lead",
    "id": "LEAD_ID"
  },
  "occurredAt": "2026-11-19T12:00:00Z",
  "payload": {}
}
```

`eventId` обеспечивает идентификацию события и дедупликацию. `correlationId` связывает события и операции одного сквозного процесса.

### 8.5. Observability

Correlation ID обязателен в структурированных логах для всех компонентов.

Необходимо обеспечить возможность поиска цепочки событий в Sentry и Cloudflare Observability.

## 9. Lead Processing Architecture

### 9.1. Lead Creation

```text
Client
  |
  v
Astro Lead Form
  |
  v
POST /api/v1/leads
  |
  v
Zod Validation
  |
  v
Turnstile Verification
  |
  v
Rate Limit / Idempotency
  |
  v
CreateLead Application Service
  |
  v
PostgreSQL Transaction
  |
  +--> leads
  +--> lead_attribution
  +--> lead_events
  +--> notification_outbox
  +--> idempotency_keys
  |
  v
COMMIT
  |
  v
HTTP Response 201
```

### 9.2. Notification Processing

```text
notification_outbox
        |
        v
Outbox Dispatcher
        |
        v
Cloudflare Queues
        |
        v
Lead Consumer
        |
        +------> Resend
        |
        +------> Telegram Lead Bot
        |
        v
Delivery Tracking
```

PostgreSQL хранит фактическое состояние заявки и результат обработки.

### 9.3. Recovery

Поскольку PostgreSQL и Cloudflare Queues не используют общую атомарную транзакцию, требуется reconciliation.

Reconciliation проверяет:

- Outbox-записи, которые остались PENDING.
- События, не переданные в Queue.
- Сообщения, которые не были подтверждены Consumer.
- Уведомления с ошибками.
- DLQ и повторную обработку.

Периодичность reconciliation: каждые 5 минут.

## 10. Telegram Bots Architecture

Используются два независимых бота.

### 10.1. Lead Management Bot

Назначение:

- Получение новых заявок.
- Просмотр заявок.
- Просмотр деталей.
- Изменение статуса.
- Добавление заметок.
- Создание напоминаний.
- Получение статистики.

### 10.2. Incident & Operations Bot

Назначение:

- Получение критических уведомлений.
- Получение предупреждений.
- Просмотр инцидентов.
- Acknowledge.
- Resolve.
- Получение состояния инфраструктуры.

### 10.3. Разделение безопасности

Каждый бот имеет:

- Отдельный Bot Token.
- Отдельный Webhook Secret.
- Отдельный webhook endpoint.
- Отдельный scope.
- Отдельные permissions.

Telegram User ID проверяется по allowlist.

Один бот не получает автоматический доступ к операциям другого.

## 11. Incident Processing

### 11.1. Direct Alert

Критические уведомления должны работать независимо от AI Agent.

```text
Monitoring Source
       |
       v
Cloudflare Worker
       |
       v
Incident Application Service
       |
       v
PostgreSQL
       |
       v
Direct Alert Dispatcher
       |
       v
Telegram Incident Bot
```

### 11.2. AI Analysis

```text
Monitoring Source
       |
       v
AI Agent
       |
       v
Technical Diagnostics
       |
       v
Sanitized Analysis
       |
       v
Worker / Operations Service
       |
       v
Telegram Incident Bot
```

AI Agent:

- Не имеет прямого доступа к PostgreSQL.
- Не получает клиентские персональные данные без необходимости.
- Не выполняет destructive production operations.
- Не заменяет прямой механизм критических уведомлений.

## 12. Storage Architecture

### 12.1. Cloudflare R2

Используется для:

- Фотографий проектов.
- Галерей.
- Before/After материалов.
- Видео и медиа-ресурсов.

Публичные медиа доступны через отдельный read-only домен.

Приватные файлы обслуживаются через presigned URLs.

### 12.2. Neon

Используется для структурированных данных:

- Заявки.
- История.
- Атрибуция.
- Состояния уведомлений.
- Операционные события.

Файлы не хранятся в PostgreSQL.

## 13. Deployment Architecture

### 13.1. MVP

Один основной Worker deployment с логическим разделением:

- Public API.
- Telegram Webhooks.
- Internal API.
- Application Services.
- Queue Consumers.
- Scheduled Jobs.

Разделение по секретам и permissions обязательно.

### 13.2. Environments

- Development.
- Staging.
- Production.

Для каждого окружения:

- Отдельные Worker bindings.
- Отдельные секреты.
- Отдельные Neon credentials.
- Отдельные Queue resources.
- Отдельные Telegram bot configuration, если используются отдельные тестовые боты.

### 13.3. Database

Один Neon Project с отдельными ветками для staging и production.

Изменения production выполняются через контролируемый CI/CD pipeline.

### 13.4. Scaling

Разделение на несколько Workers возможно в будущем при появлении обоснованной нагрузки или независимых требований к безопасности.

Это не является условием запуска MVP.

## 14. Security Architecture

- Cloudflare WAF.
- Cloudflare Turnstile.
- HTTPS.
- Least Privilege.
- Worker Secrets.
- Scoped API Tokens.
- MFA для инфраструктурных аккаунтов.
- Разделение staging и production credentials.
- Telegram User ID allowlist.
- Webhook Secret validation.
- Zod validation.
- Rate limiting.
- Parameterized SQL.
- No direct frontend database access.
- No secrets in logs.
- Correlation ID без персональных данных.
- Ограниченный доступ AI Agent.
- Отдельный контроль production deployment.

## 15. Monitoring and Observability

Используются:

- Sentry.
- Cloudflare Observability.
- Neon Monitoring.
- GitHub Actions.
- Telegram Incident & Operations Bot.

Контролируемые показатели:

- API availability.
- API latency.
- Error rate.
- Database connectivity.
- Outbox backlog.
- Queue processing.
- Retry count.
- DLQ messages.
- Notification delivery.
- Reminder processing.
- Telegram webhook failures.
- Critical incidents.
- Correlation ID traces.

## 16. Performance Requirements

Core Web Vitals:

- LCP ≤ 2.5s.
- INP ≤ 200ms.
- CLS ≤ 0.1.

Дополнительные требования:

- Astro SSG для публичных страниц.
- Минимизация JavaScript.
- Lazy loading изображений.
- Responsive image delivery.
- Оптимизация шрифтов.
- CDN caching.
- Минимизация внешних запросов.
- Отдельная проверка производительности мобильных устройств.

## 17. Disaster Recovery

Целевые показатели:

- RPO: 1 час.
- RTO: 4 часа.

Neon backup/PITR зависит от выбранного тарифного плана.

R2 backup рассматривается отдельно.

Обязательны регулярные restore tests и документированная процедура восстановления.

## 18. Architecture Decisions

| ID | Решение | Статус |
| --- | --- | --- |
| SAD-ADR-001 | Astro SSG | Approved |
| SAD-ADR-002 | Cloudflare Workers + Static Assets | Approved |
| SAD-ADR-003 | Neon PostgreSQL | Approved |
| SAD-ADR-004 | Drizzle ORM + Drizzle Kit | Approved |
| SAD-ADR-005 | Cloudflare R2 | Approved |
| SAD-ADR-006 | Cloudflare Queues + Transactional Outbox | Approved |
| SAD-ADR-007 | Manual Lead Management without CRM | Approved |
| SAD-ADR-008 | Minimal Lead Attribution | Approved |
| SAD-ADR-009 | Application Services inside Worker | Approved |
| SAD-ADR-010 | Unified Correlation ID | Approved |
| SAD-ADR-011 | PostgreSQL Driver compatibility with Workers and Neon | Confirmed |
| SAD-ADR-012 | Two independent Telegram Bots | Approved |
| SAD-ADR-013 | Direct Critical Alert independent of AI Agent | Approved |

## 19. Open Decisions

| ID | Вопрос | Статус |
| --- | --- | --- |
| SAD-OD-001 | Финальный формат Event Schema и versioning | To specify |
| SAD-OD-002 | Конкретные Queue resource settings | To configure |
| SAD-OD-003 | Окончательная политика retention | To approve |
| SAD-OD-004 | Финальная реализация AI Agent integration | To implement |
| SAD-OD-005 | Финальная конфигурация Sentry alerts | To configure |

Подтверждённая совместимость PostgreSQL-драйвера не является открытым вопросом.

## 20. Acceptance Criteria

- Astro SSG работает на Cloudflare.
- Public API работает внутри Worker.
- Application Services не используют HTTP self-calls.
- PostgreSQL подключён через подтверждённый совместимый драйвер.
- Все бизнес-изменения фиксируются в PostgreSQL.
- Outbox и Queue работают по согласованной модели.
- Correlation ID проходит через весь жизненный цикл события.
- Consumer обрабатывает повторы безопасно.
- Два Telegram-бота имеют независимые credentials и permissions.
- Direct Alert работает без AI Agent.
- AI Agent не имеет прямого доступа к DB.
- Секреты разделены по окружениям.
- Monitoring и recovery проверены на staging.
- Критические тесты проходят.

## 21. Связанные документы

- BRD-001 — Business Requirements.
- SRS-001 — Software Requirements Specification.
- IA-001 — Information Architecture.
- UX-001 — User Experience Specification.
- BRAND-001 — Brand Specification.
- CONTENT-001 — Content Strategy.
- SEO-001 — SEO & GEO Specification.
- DATA-001 — Data Model.
- API-001 — API Specification.
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
| 0.1 | 2026-11-17 | Первичная архитектура Astro, Cloudflare, Neon, R2 |
| 0.2 | 2026-11-19 | Application Services внутри Worker; единый Correlation ID; подтверждённая совместимость PostgreSQL-драйвера; Cloudflare Queues; два Telegram-бота; Direct Alert; обновлены Outbox, deployment и observability |
