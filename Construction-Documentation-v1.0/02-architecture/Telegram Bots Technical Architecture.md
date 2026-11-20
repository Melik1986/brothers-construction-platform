# TECH-001 — Telegram Bots Technical Architecture

**Проект:** «Братья» / եղբայրներ  
**Документ:** TECH-001  
**Версия:** 0.2  
**Статус:** Draft — Technical Consistency Update  
**Дата:** 19.11.2026

---

## 1. Назначение документа

TECH-001 определяет техническую архитектуру двух Telegram-ботов:

1. Lead Management Bot — управление клиентскими заявками.
2. Incident & Operations Bot — мониторинг, уведомления и управление техническими инцидентами.

Оба бота работают на общей инфраструктуре Cloudflare Workers, но имеют независимые Telegram credentials, webhook endpoints, scopes и permissions.

Документ определяет:

- Архитектуру ботов.
- Команды и пользовательские сценарии.
- Webhook processing.
- Авторизацию Telegram-пользователей.
- Application Services integration.
- Database integration.
- Cloudflare Queues.
- Correlation ID.
- Notification delivery.
- Retry и idempotency.
- Error handling.
- Security.
- Testing и acceptance criteria.

## 2. Архитектурные принципы

1. PostgreSQL — источник истины.
2. Telegram Bot API — внешний интерфейс взаимодействия.
3. Боты не имеют прямого доступа к PostgreSQL.
4. Бизнес-логика находится в Application Services внутри Cloudflare Worker.
5. HTTP self-calls между внутренними компонентами Worker не используются.
6. У каждого бота собственный Bot Token и Webhook Secret.
7. Авторизация пользователей выполняется по Telegram User ID и permissions.
8. Все операции управления заявками и инцидентами фиксируются в PostgreSQL.
9. Correlation ID проходит через весь жизненный цикл операции.
10. Внешние уведомления обрабатываются асинхронно там, где это предусмотрено архитектурой.
11. Direct Alert не зависит от доступности AI Agent.
12. AI Agent не выполняет разрушительные production-операции.

## 3. Общая архитектура

```text
                 TELEGRAM PLATFORM
                         |
              +----------+----------+
              |                     |
              v                     v
       LEAD MANAGEMENT BOT    INCIDENT BOT
              |                     |
              v                     v
       Telegram Webhook       Telegram Webhook
              |                     |
              +----------+----------+
                         |
                         v
                CLOUDFLARE WORKER
                         |
              +----------+----------+
              |                     |
              v                     v
       Authentication          Validation
              |                     |
              +----------+----------+
                         |
                         v
                APPLICATION SERVICES
                         |
              +----------+----------+
              |                     |
              v                     v
        LEAD SERVICES          INCIDENT SERVICES
              |                     |
              +----------+----------+
                         |
                         v
                  NEON POSTGRESQL
                         |
              +----------+----------+
              |                     |
              v                     v
         BUSINESS DATA        NOTIFICATION OUTBOX
                                    |
                                    v
                              CLOUDFLARE QUEUES
                                    |
                          +---------+---------+
                          |                   |
                          v                   v
                   LEAD CONSUMER      OPERATIONS CONSUMER
                          |                   |
                          v                   v
                     TELEGRAM API         TELEGRAM API
```

## 4. Bot 1 — Lead Management Bot

### 4.1. Назначение

Lead Management Bot предназначен для оперативного управления заявками клиентов.

Основные задачи:

- Получение уведомлений о новых заявках.
- Просмотр списка заявок.
- Просмотр подробной информации.
- Изменение статусов.
- Добавление заметок.
- Создание напоминаний.
- Получение статистики.

### 4.2. Команды

| Команда | Назначение |
|---|---|
| `/start` | Начало работы и проверка доступа |
| `/help` | Список доступных команд |
| `/leads` | Список заявок |
| `/new` | Новые заявки |
| `/lead ID` | Детали заявки |
| `/status ID` | Просмотр или изменение статуса |
| `/note ID` | Добавление заметки |
| `/remind ID` | Создание напоминания |
| `/stats` | Статистика заявок |
| `/today` | Заявки за текущий день |

Все команды требуют проверки Telegram User ID и соответствующих permissions.

### 4.3. Просмотр заявки

Бот должен показывать:

- ID заявки.
- Имя клиента.
- Телефон.
- Email, если указан.
- Запрошенную услугу.
- Предпочтительный способ связи.
- Сообщение.
- Язык.
- Дату создания.
- Текущий статус.
- Доступную атрибуцию.

Персональные данные не должны отправляться в общие группы. Для MVP предполагается личный чат владельца с ботом.

### 4.4. Изменение статуса

Поддерживаемые статусы:

- NEW
- CONTACTED
- QUALIFIED
- ESTIMATE_SENT
- WON
- LOST
- SPAM

Пример пользовательского сценария:

```text
Owner
  |
  v
/lead 123
  |
  v
Bot displays lead details
  |
  v
Owner selects "Change status"
  |
  v
Bot displays available statuses
  |
  v
Owner selects CONTACTED
  |
  v
Application Service
  |
  v
PostgreSQL Transaction
  |
  +--> Update leads
  +--> Create lead_events
  +--> Update reminders if necessary
  |
  v
Bot confirms operation
```

Изменение статуса и запись события выполняются в одной DB-транзакции.

### 4.5. Заметки

Владелец может добавлять заметки к заявке.

Пример:

```text
/note 123

Client requested an estimate.
Follow up tomorrow morning.
```

Операция:

1. Проверка permissions.
2. Проверка существования заявки.
3. Создание записи в `lead_notes`.
4. Создание `NOTE_ADDED` в `lead_events`.
5. Возврат подтверждения.

### 4.6. Напоминания

Бот поддерживает:

- Ручные напоминания.
- Напоминания о новых заявках.
- Follow-up.
- Уведомления о просроченных действиях.

Пример:

```text
/remind 123

Tomorrow at 10:00
```

Система создаёт запись в `lead_reminders`.

Обработка выполняется Scheduled Worker.

Перед отправкой проверяются:

- Статус заявки.
- Статус напоминания.
- `scheduled_at`.
- `lease_until`.
- `attempt_count`.
- Idempotency key.

### 4.7. Уведомление о новой заявке

Пример сообщения:

```text
NEW LEAD

ID: 123
Name: Aram
Service: Renovation
Contact: WhatsApp
Language: HY
Created: 19.11.2026 12:00

Status: NEW
```

Сообщение может содержать inline actions:

- Open Lead.
- Mark Contacted.
- Add Note.

Inline actions должны проходить те же проверки авторизации, что и обычные команды.

---

## 5. Bot 2 — Incident & Operations Bot

### 5.1. Назначение

Incident & Operations Bot отвечает за мониторинг и оперативные технические уведомления.

Основные задачи:

- Получение критических уведомлений.
- Получение предупреждений.
- Просмотр инцидентов.
- Подтверждение инцидента.
- Разрешение инцидента.
- Получение состояния системы.
- Получение диагностических результатов AI Agent.

### 5.2. Команды

| Команда | Назначение |
|---|---|
| `/start` | Проверка доступа |
| `/help` | Список команд |
| `/status` | Общее состояние системы |
| `/incidents` | Активные инциденты |
| `/incident ID` | Детали инцидента |
| `/ack ID` | Подтвердить инцидент |
| `/resolve ID` | Закрыть инцидент |
| `/health` | Проверить доступность сервисов |

### 5.3. Уровни критичности

- INFO
- WARNING
- ERROR
- CRITICAL

### 5.4. Статусы инцидентов

- OPEN
- ACKNOWLEDGED
- RESOLVED
- IGNORED

### 5.5. Прямое уведомление

Критические уведомления должны отправляться независимо от AI Agent.

```text
Monitoring
    |
    v
Incident Service
    |
    v
PostgreSQL
    |
    v
Direct Alert
    |
    v
Telegram Incident Bot
```

AI Analysis выполняется параллельно или отдельно и не блокирует отправку Direct Alert.

### 5.6. AI Analysis

AI Agent может:

- Анализировать очищенные технические логи.
- Собирать диагностический контекст.
- Группировать связанные ошибки.
- Предлагать возможные причины.
- Формировать рекомендации.
- Передавать диагностический результат в Operations Service.

AI Agent не может:

- Самостоятельно менять production secrets.
- Удалять production-ресурсы.
- Выполнять destructive DB operations.
- Самостоятельно выполнять production deployment.
- Изменять бизнес-данные заявок.
- Получать неограниченный доступ к PostgreSQL.

---

## 6. Webhook Architecture

### 6.1. Endpoints

Lead Management Bot:

`POST /api/v1/webhooks/telegram/leads`

Incident & Operations Bot:

`POST /api/v1/webhooks/telegram/incident`

### 6.2. Credentials

Используются отдельные секреты:

- `TELEGRAM_LEADS_BOT_TOKEN`
- `TELEGRAM_LEADS_WEBHOOK_SECRET`
- `TELEGRAM_INCIDENT_BOT_TOKEN`
- `TELEGRAM_INCIDENT_WEBHOOK_SECRET`

Секреты хранятся в Cloudflare Worker Secrets.

Они разделяются между development, staging и production.

### 6.3. Webhook registration

Webhook регистрируется через Telegram Bot API с использованием HTTPS URL и соответствующего `secret_token`.

Каждый бот регистрируется отдельно.

Проверка webhook secret выполняется до обработки Update.

### 6.4. Update processing

```text
Telegram Update
      |
      v
Webhook Handler
      |
      v
Verify Webhook Secret
      |
      v
Validate Update Payload
      |
      v
Check Telegram Update ID
      |
      v
Resolve Telegram User ID
      |
      v
Check Permissions
      |
      v
Application Service
      |
      v
Persist Result
      |
      v
Telegram Response
```

### 6.5. Idempotency

Для `telegram_updates` используется уникальная комбинация:

`telegram_update_id + bot_scope`

Повторная доставка Update не должна повторно выполнять уже успешно обработанную операцию.

Для Update со статусом обработки, отличным от завершённого, применяется определённая политика восстановления, чтобы не потерять команду после аварийного завершения Worker.

---

## 7. Authorization Model

### 7.1. Telegram User ID

Все защищённые команды проверяют Telegram User ID.

Неизвестные пользователи не получают доступа к операциям.

### 7.2. Database model

Используются:

- `bot_users`
- `bot_user_permissions`

### 7.3. Bot scopes

- `LEADS`
- `INCIDENTS`

### 7.4. MVP permissions

Предусматривается один владелец с правами OWNER.

Разрешения должны ограничиваться scope конкретного бота.

### 7.5. Access rules

- Lead Bot не получает разрешения Incident Bot автоматически.
- Incident Bot не получает разрешения Lead Bot автоматически.
- Деактивированный пользователь не может выполнять команды.
- Проверка прав выполняется для каждой защищённой операции.
- Проверка webhook secret не заменяет проверку пользовательских прав.

---

## 8. Application Services Integration

### 8.1. Общий принцип

Telegram Webhook Handler является транспортным адаптером.

Он не содержит бизнес-логику работы с заявками или инцидентами.

Вызовы выполняются напрямую внутри Worker.

### 8.2. Lead Application Services

- `createLead`
- `getLead`
- `listLeads`
- `updateLeadStatus`
- `addLeadNote`
- `createLeadReminder`
- `getLeadStats`

### 8.3. Incident Application Services

- `createIncident`
- `getIncident`
- `listIncidents`
- `acknowledgeIncident`
- `resolveIncident`
- `getOperationsStatus`

### 8.4. Архитектурный поток

```text
Telegram Webhook
       |
       v
Webhook Handler
       |
       v
Authentication
       |
       v
Authorization
       |
       v
Command Parser
       |
       v
Application Service
       |
       v
Repository
       |
       v
PostgreSQL
```

Запрещено выполнять внутренние HTTP self-calls к собственному Worker для вызова Application Services.

---

## 9. Correlation ID

### 9.1. Назначение

Все операции ботов должны иметь Correlation ID.

Он используется для связи:

- Telegram Update.
- Webhook Handler.
- Application Service.
- PostgreSQL.
- Outbox Event.
- Queue Message.
- Consumer.
- Telegram Bot API.
- Sentry и Cloudflare Logs.

### 9.2. Правила

1. Correlation ID создаётся при приёме Update, если отсутствует.
2. Проверка входящего идентификатора обязательна.
3. ID передаётся через execution context.
4. При создании Outbox Event он сохраняется в event envelope.
5. Consumer использует тот же ID.
6. Все ошибки логируются с этим ID.
7. Повторная обработка сохраняет исходный ID.
8. Correlation ID не должен содержать персональные данные.

### 9.3. Разделение идентификаторов

- `telegram_update_id` — идентификатор Telegram Update.
- `eventId` — идентификатор бизнес-события.
- `correlationId` — идентификатор сквозной операции.
- `idempotencyKey` — ключ защиты от повторного выполнения.

Эти значения не взаимозаменяемы.

---

## 10. Queue Architecture

### 10.1. Logical Queues

**lead-notifications**

Назначение:

- Уведомления о новых заявках.
- Follow-up notifications.
- Служебные уведомления Lead Management.

**operations-events**

Назначение:

- Incident notifications.
- Operations events.
- Диагностические события.
- Уведомления о проблемах инфраструктуры.

### 10.2. Deferred Queue

`lead-commands` не входит в обязательную конфигурацию MVP.

Изменение статуса, создание заметок и напоминаний выполняются синхронно через Application Services и PostgreSQL.

### 10.3. Outbox

Все асинхронные бизнес-события должны сначала сохраняться в `notification_outbox`.

Dispatcher передаёт их в соответствующие Queues.

### 10.4. Consumers

Consumer выполняет:

1. Получение сообщения.
2. Проверку event schema.
3. Восстановление Correlation ID.
4. Проверку идемпотентности.
5. Выполнение необходимой операции.
6. Сохранение результата.
7. Подтверждение сообщения или retry согласно политике обработки.

### 10.5. Retry policy

- 5 попыток, включая первоначальную.
- Retry delays: 1, 5, 15 и 60 минут.
- После исчерпания попыток — DLQ.
- Reconciliation каждые 5 минут.

Фактические настройки должны соответствовать доступным механизмам Cloudflare Queues.

---

## 11. Reminder Scheduler

Scheduled Worker выполняет:

- Проверку новых заявок без изменения статуса.
- Создание напоминания через 2 часа.
- Создание напоминания через 24 часа.
- Обработку ручных follow-up.
- Ежедневную сводку.
- Reconciliation напоминаний.

### 11.1. Защита от конкурентной обработки

Перед отправкой напоминание атомарно переводится в `PROCESSING`.

Используются:

- `attempt_count`
- `lease_until`
- `idempotency_key`
- `bot_notification_deliveries`

Два Worker не должны одновременно отправлять одно и то же напоминание.

### 11.2. Автоматические напоминания

- Через 2 часа без изменения статуса.
- Через 24 часа без изменения статуса.
- Ежедневная сводка.

При изменении статуса заявки система проверяет актуальность автоматических напоминаний.

Для `WON`, `LOST` и `SPAM` ненужные напоминания отменяются.

---

## 12. Notification Templates

Шаблоны сообщений должны поддерживать:

- Armenian.
- Russian.
- English.

Для MVP внутренние уведомления владельцу могут использовать единый выбранный язык интерфейса бота. Язык уведомления клиенту определяется отдельно и не должен автоматически совпадать с языком владельца.

### 12.1. New Lead

```text
NEW LEAD

ID: {leadId}
Name: {name}
Service: {service}
Contact: {preferredContact}
Language: {locale}
Status: {status}

Created: {createdAt}
```

### 12.2. Critical Incident

```text
CRITICAL INCIDENT

ID: {incidentId}
Source: {source}
Environment: {environment}
Title: {title}

Status: OPEN
Detected: {firstSeenAt}
Correlation ID: {correlationId}
```

### 12.3. Reminder

```text
FOLLOW-UP REMINDER

Lead: {leadId}
Client: {name}
Service: {service}
Scheduled: {scheduledAt}

Action required.
```

### 12.4. Security

Сообщения не должны содержать:

- Bot Tokens.
- Webhook Secrets.
- Database credentials.
- Полные технические stack traces.
- Необходимые только разработчику внутренние секреты.

---

## 13. Error Handling

Категории ошибок:

- WEBHOOK_AUTH_ERROR
- INVALID_TELEGRAM_UPDATE
- UNKNOWN_TELEGRAM_USER
- PERMISSION_DENIED
- DUPLICATE_UPDATE
- APPLICATION_ERROR
- DATABASE_ERROR
- QUEUE_DELIVERY_ERROR
- TELEGRAM_DELIVERY_ERROR
- REMINDER_PROCESSING_ERROR
- INCIDENT_PROCESSING_ERROR

Правила:

- Централизованная обработка ошибок.
- Структурированные логи.
- Correlation ID для каждой ошибки.
- Не раскрывать внутренние ошибки пользователю.
- Временные ошибки могут приводить к повторной обработке.
- Постоянные ошибки должны фиксироваться и попадать в соответствующий operational workflow.

---

## 14. Monitoring

Контролируемые показатели:

- Webhook request count.
- Webhook errors.
- Invalid secret attempts.
- Unauthorized Telegram users.
- Duplicate updates.
- Application Service errors.
- Database errors.
- Queue backlog.
- Consumer retries.
- DLQ messages.
- Notification delivery failures.
- Reminder failures.
- Incident notification latency.

Используются:

- Cloudflare Observability.
- Sentry.
- Neon Monitoring.
- Telegram Incident & Operations Bot.

---

## 15. Security Requirements

- Раздельные Telegram Bot Tokens.
- Раздельные webhook secrets.
- HTTPS only.
- Проверка webhook secret.
- Telegram User ID allowlist.
- Permissions по bot scope.
- Ограничение размера Update.
- Валидация входных данных.
- Идемпотентная обработка.
- Секреты в Cloudflare Worker Secrets.
- Отдельные credentials для окружений.
- Отсутствие прямого DB access у ботов.
- Отсутствие секретов в логах.
- Sanitization технических данных перед передачей AI Agent.
- Запрет destructive production actions для AI Agent.

---

## 16. Testing Strategy

### Lead Management Bot

- `/start`
- `/help`
- `/leads`
- `/new`
- `/lead ID`
- Изменение статуса.
- Добавление заметки.
- Создание напоминания.
- Получение статистики.
- Проверка permissions.
- Повторная доставка Update.

### Incident & Operations Bot

- `/start`
- `/help`
- `/status`
- `/incidents`
- `/incident ID`
- `/ack ID`
- `/resolve ID`
- `/health`
- Проверка permissions.
- Проверка Direct Alert.
- Проверка независимости от AI Agent.

### Infrastructure

- Webhook secret validation.
- Correlation ID propagation.
- Queue retry.
- DLQ.
- Outbox reconciliation.
- Consumer idempotency.
- Reminder concurrency.
- DB transaction rollback.
- Telegram API unavailable.
- AI Agent unavailable.

---

## 17. Acceptance Criteria

TECH-001 считается реализованным, когда:

- Два бота зарегистрированы независимо.
- У каждого бота отдельные credentials.
- Webhook endpoints разделены.
- Авторизация Telegram User ID работает.
- Permissions разделены по scopes.
- Бизнес-операции вызываются через Application Services.
- Внутренние HTTP self-calls отсутствуют.
- Все операции имеют Correlation ID.
- Telegram Updates защищены от повторной обработки.
- Lead Management поддерживает основные команды.
- Incident Bot поддерживает основные операции.
- Новые заявки доставляются владельцу.
- Напоминания защищены от конкурентной отправки.
- Direct Alert работает без AI Agent.
- Queue Consumers идемпотентны.
- Секреты не раскрываются в логах.
- Критические тесты проходят.

---

## 18. Open Decisions

| ID | Вопрос | Статус |
|---|---|---|
| TECH-OD-001 | Финальный язык внутреннего интерфейса ботов | To confirm |
| TECH-OD-002 | Финальный формат inline keyboards | To implement |
| TECH-OD-003 | Выбор AI Agent integration provider | To finalize |
| TECH-OD-004 | Финальные настройки Telegram webhook deployment | To configure |
| TECH-OD-005 | Числовые лимиты команд и защиты от злоупотреблений | To configure |

---

## 19. Связанные документы

- SAD-001 — System Architecture.
- ADR-001 — Architecture Decision Records.
- DATA-001 — Data Model.
- API-001 — API Specification.
- OPS-001 — Operations & Monitoring.
- DEPLOY-001 — Deployment & CI/CD.
- TEST-001 — Testing Strategy.
- IMPLEMENTATION-001 — Implementation Plan.

---

## Change Log

| Версия | Дата | Изменения |
|---|---|---|
| 0.1 | 2026-11-18 | Первичная архитектура двух Telegram-ботов |
| 0.2 | 2026-11-19 | Уточнены Application Services, Webhook authentication, permissions, Correlation ID, Queue Consumers, Reminder Scheduler, Direct Alert и AI Analysis |