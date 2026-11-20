# OPS-001 — Operations, Monitoring & Incident Response Runbook (Consolidated)

**Project:** «Братья» / եղբայրներ
**Document ID:** OPS-001
**Version:** 0.4
**Status:** Draft — Consolidated Operational Specification
**Document Type:** Operations & Incident Response
**Related Documents:** BRD-001, SRS-001, SAD-001, API-001, DATA-001, ADR-001, TEST-001, DEPLOY-001

---

## 1. Purpose

OPS-001 определяет операционную модель эксплуатации веб-платформы строительной компании «Братья».

Документ устанавливает:

- мониторинг доступности и состояния компонентов;
- автоматическое обнаружение и диагностику инцидентов;
- уведомление владельца проекта;
- автоматизированное управление заявками через Telegram;
- обработку ошибок и повторные попытки;
- восстановление после сбоев;
- резервное копирование и проверку восстановления;
- безопасность операционных инструментов;
- регулярные проверки и критерии готовности production.

Основная цель — обеспечить доступность сайта, сохранность заявок и максимальную автоматизацию операционных процессов без необходимости постоянного ручного контроля.

## 2. Operational Principles

### 2.1. Core Principles

1. **Availability:** публичный сайт и API должны оставаться доступными.
2. **Data Integrity:** сохранность заявок имеет приоритет над доступностью вторичных интеграций.
3. **Automation First:** повторяющиеся операционные задачи должны выполняться автоматически.
4. **AI-Assisted Operations:** AI-агенты могут анализировать инциденты и предлагать решения, но критические изменения production требуют подтверждения владельца.
5. **Observability:** проблемы должны обнаруживаться через мониторинг, логи и уведомления.
6. **Recoverability:** критические компоненты должны иметь документированные процедуры восстановления.
7. **Security:** доступ к production и персональным данным предоставляется по принципу Least Privilege.
8. **Traceability:** все значимые действия должны оставлять audit trail.

### 2.2. Operational Priorities

1. Сохранность заявок.
2. Работоспособность Lead Generation API.
3. Доступность публичного сайта.
4. Надёжность обработки очередей.
5. Доставка уведомлений.
6. Автоматизация управления заявками.
7. Аналитика и вторичные интеграции.

## 3. Operational Architecture

### 3.1. Production Components

| Component | Responsibility | Priority |
| --- | --- | --- |
| Cloudflare Workers + Static Assets | Публичный сайт и API | Critical |
| Neon PostgreSQL | Хранение заявок и операционных данных | Critical |
| Cloudflare Queues | Асинхронная обработка | High |
| Cloudflare R2 | Медиафайлы | Medium |
| Resend | Email-уведомления | Medium |
| Telegram Bot API | Операционные уведомления и управление заявками | High |
| Cloudflare Turnstile | Anti-spam | High |
| Sentry | Application Error Monitoring | High |
| Cursor Automations или аналогичный AI Agent | Автоматическая диагностика инцидентов | High |
| GitHub Actions | CI/CD | High |
| Cloudflare Observability | Инфраструктурная наблюдаемость | High |

### 3.2. System of Record

Neon PostgreSQL является основным источником истины для заявок.

Telegram Bot — интерфейс управления, а не самостоятельное хранилище данных.

Cloudflare Queues обеспечивает асинхронную обработку, но не заменяет PostgreSQL.

Сбой Telegram, Resend или AI-агента не должен приводить к потере заявок.

### 3.3. Environment Separation

Используются два окружения:

- Staging;
- Production.

Они имеют отдельные Workers, Queue resources, R2 resources, Neon branches, credentials, secrets и Sentry environments.

Staging не должен использовать production credentials или автоматически получать production data.

## 4. Monitoring Strategy

### 4.1. Monitoring Layers

**Infrastructure Monitoring**

- Cloudflare Worker availability;
- Worker errors и latency;
- Queue backlog;
- consumer failures;
- Neon availability;
- database connections;
- R2 access errors.

**Application Monitoring**

- API errors;
- unhandled exceptions;
- failed lead submissions;
- validation errors;
- Turnstile failures;
- transaction failures.

**Integration Monitoring**

- Resend delivery failures;
- Telegram API errors;
- Queue retry events;
- dead-letter events;
- notification processing delays.

**User Experience Monitoring**

- Core Web Vitals;
- JavaScript errors;
- critical page availability;
- contact form completion;
- failed form submissions.

### 4.2. Monitoring Tools

| Tool | Purpose | Status |
| --- | --- | --- |
| Sentry | Application errors and stack traces | Approved |
| Cloudflare Observability | Worker logs and metrics | Planned |
| Neon Monitoring | Database health | Planned |
| GitHub Actions | CI/CD monitoring | Approved |
| Telegram Bot | Critical alerts and lead operations | Approved |
| Cursor Automations or equivalent | AI incident triage | Proposed implementation |
| Lighthouse CI | Performance regression checks | Planned |

Тарифы, retention periods и конкретные настройки провайдеров проверяются перед внедрением.

### 4.3. Sentry Configuration

Sentry используется для сбора ошибок приложения.

Обязательные настройки:

- environments: `staging`, `production`;
- release identifier: Git commit SHA или release version;
- source maps для production builds;
- группировка повторяющихся ошибок;
- alerts для критических и повторяющихся ошибок;
- выборочный performance monitoring.

Перед отправкой событий необходимо исключать или маскировать:

- телефоны;
- Email;
- сообщения клиентов;
- Turnstile tokens;
- API keys;
- authorization headers;
- database connection strings;
- другие secrets.

### 4.4. Cloudflare Monitoring

Контролировать:

- Worker invocation errors;
- HTTP 5xx responses;
- execution exceptions;
- latency;
- request volume;
- Queue consumer errors;
- retry и dead-letter events.

### 4.5. Database Monitoring

Контролировать:

- database availability;
- connection failures;
- connection saturation;
- query failures;
- transaction failures;
- storage consumption;
- resource usage;
- migration failures.

Особое внимание уделяется connection management при использовании PostgreSQL driver в Cloudflare Workers.

Hyperdrive не входит в MVP.

## 5. AI Incident Response Agent

### 5.1. Purpose

Для автоматизации мониторинга и первичной диагностики используется AI Incident Response Agent.

Предпочтительный кандидат — Cursor Automations либо аналогичная платформа, поддерживающая:

- event-driven triggers;
- webhook integrations;
- Sentry integration;
- доступ к диагностическим данным;
- MCP integrations;
- отправку структурированных уведомлений.

Cursor Bugbot не является основным инструментом этого процесса: его основное назначение — анализ изменений в Pull Requests.

### 5.2. Responsibilities

AI Incident Response Agent отвечает за:

1. Получение событий от Sentry и инфраструктурного мониторинга.
2. Определение предполагаемой серьёзности инцидента.
3. Группировку повторяющихся ошибок.
4. Анализ stack traces и связанных логов.
5. Проверку последних deployments и изменений.
6. Поиск возможной первопричины.
7. Подготовку технического резюме.
8. Формирование рекомендаций по устранению.
9. Отправку критических уведомлений владельцу проекта через Telegram.
10. Повторное информирование при изменении состояния инцидента.

### 5.3. Owner Notification

Владелец проекта является основным получателем критических уведомлений.

Telegram используется как основной канал доставки.

Формат уведомления:

- Incident ID;
- Severity;
- Environment;
- Affected Component;
- Description;
- Detected At;
- Suspected Cause;
- Business Impact;
- Recommended Action;
- Monitoring Link.

Персональные данные клиентов и secrets в уведомления не включаются.

### 5.4. AI Agent Permissions

AI Agent имеет ограниченные права.

Разрешено:

- читать разрешённые логи;
- анализировать Sentry events;
- получать deployment metadata;
- читать диагностическую информацию;
- создавать incident summaries;
- отправлять уведомления;
- готовить рекомендации;
- создавать draft issues или Pull Requests при наличии соответствующего workflow.

Запрещено без отдельного подтверждения владельца:

- удалять production data;
- выполнять production database migrations;
- изменять production secrets;
- отключать security controls;
- удалять Queue messages;
- выполнять destructive operations;
- самостоятельно менять production deployment;
- отправлять сообщения клиентам от имени компании.

### 5.5. Incident Automation Flow

1. Monitoring system обнаруживает событие.
2. Event передаётся AI Agent.
3. Agent собирает разрешённую диагностическую информацию.
4. Agent определяет предполагаемую severity.
5. Agent формирует incident summary.
6. При SEV-1/SEV-2 уведомляет владельца через Telegram.
7. Владелец принимает решение по критическим действиям.
8. Agent может продолжить анализ и подготовить безопасный технический план.
9. После устранения проблемы Agent получает recovery event.
10. Incident закрывается после проверки восстановления.

AI Agent является автоматизированным техническим помощником, а не единственным механизмом обнаружения и устранения неисправностей. Независимый мониторинг и прямые критические alerts должны продолжать работать даже при недоступности AI Agent.

## 6. Health Checks

### 6.1. API Health Endpoint

`GET /api/v1/health`

Ответ содержит:

- API status;
- application version;
- environment;
- timestamp.

Не раскрываются credentials, внутренние адреса, stack traces, персональные данные и конфигурационные secrets.

### 6.2. Dependency Health

Отдельно проверяются:

- PostgreSQL;
- Queue processing;
- R2;
- Turnstile;
- Resend;
- Telegram Bot API.

Недоступность необязательного провайдера уведомлений не должна автоматически переводить весь API в состояние unavailable.

### 6.3. Health Check Security

- Не выполнять тяжёлые SQL-запросы.
- Не создавать уведомления в процессе проверки.
- Не раскрывать внутреннюю конфигурацию.
- Не использовать endpoint для массовой диагностики зависимостей.

## 7. Logging Strategy

### 7.1. Structured Logs

Использовать JSON logs.

Базовые поля:

- `timestamp`
- `level`
- `environment`
- `service`
- `release`
- `requestId`
- `correlationId`
- `event`
- `durationMs`
- `statusCode`
- `errorCode`

### 7.2. Log Levels

| Level | Usage |
| --- | --- |
| DEBUG | Подробная диагностика |
| INFO | Нормальные операционные события |
| WARN | Предупреждения и retry |
| ERROR | Ошибки операций |
| FATAL | Критические сбои |

DEBUG по умолчанию отключён в production.

### 7.3. Sensitive Data Policy

Запрещено логировать:

- passwords;
- API tokens;
- authorization headers;
- database URLs;
- полные клиентские формы;
- необработанные сообщения клиентов без операционной необходимости.

Для идентификации заявок использовать `leadId`, а не телефон или Email.

### 7.4. Correlation

Для трассировки процессов используются:

- `requestId`;
- `leadId`;
- `correlationId`;
- `notificationId`;
- `incidentId`.

## 8. Alerting and Notification Policy

### 8.1. Alert Channels

- Telegram — критические операционные уведомления владельцу.
- Sentry — application error monitoring.
- GitHub Actions — CI/CD status.
- AI Incident Agent — диагностика и формирование уведомлений.

### 8.2. Alert Severity

**Critical**

- Production API недоступен.
- Массовые HTTP 5xx.
- Потеря соединения с основной базой.
- Подтверждённая потеря или повреждение данных.
- Компрометация production credentials.
- Невозможность сохранять новые заявки.

**High**

- Queue Consumer систематически завершается ошибкой.
- Значительный рост dead-letter messages.
- Production migration failed.
- Массовая недоступность Turnstile.
- Резкий рост application errors.

**Medium**

- Resend недоступен.
- Telegram Bot API недоступен.
- Рост Queue processing delay.
- Ухудшение Core Web Vitals.
- Рост database resource consumption.

**Low**

- Единичные некритичные ошибки.
- Предупреждения CI.
- Незначительные performance regressions.

### 8.3. Alert Deduplication

- Повторяющиеся ошибки группируются.
- Одинаковые события не должны создавать бесконечный поток уведомлений.
- Новое уведомление отправляется при изменении severity или состояния.
- После восстановления отправляется recovery notification.

## 9. Incident Management

### 9.1. Incident Definition

Инцидент — событие, которое влияет или может повлиять на доступность, сохранность данных, безопасность или работу бизнес-функций.

### 9.2. Severity

**SEV-1 — Critical:** полная недоступность production, потеря данных, серьёзный security incident.

**SEV-2 — High:** существенное нарушение API, приёма заявок или обработки данных.

**SEV-3 — Medium:** частичная деградация без подтверждённой потери данных.

**SEV-4 — Low:** некритичные ошибки и предупреждения.

### 9.3. Incident Lifecycle

1. Detection.
2. Triage.
3. Containment.
4. Investigation.
5. Resolution.
6. Recovery.
7. Verification.
8. Postmortem.

### 9.4. Incident Ownership

Владелец проекта является ответственным лицом за принятие решений по критическим инцидентам.

AI Incident Response Agent выполняет автоматизированную первичную диагностику и направляет критические уведомления владельцу.

Владелец утверждает критические изменения production и принимает решение об эскалации.

### 9.5. Incident Record

Для SEV-1 и SEV-2 создаётся incident record:

- Incident ID;
- severity;
- start time;
- detection time;
- affected components;
- impact;
- suspected cause;
- actions taken;
- recovery time;
- data integrity status;
- root cause;
- preventive actions.

## 10. Queue Retry and Dead-Letter Policy

### 10.1. General Policy

Cloudflare Queues используется для асинхронной доставки уведомлений.

Consumer должен быть идемпотентным.

Transactional Outbox сохраняет информацию о событиях в PostgreSQL. Reconciliation process проверяет события, которые были сохранены в Outbox, но не попали в очередь из-за сбоя между операциями.

### 10.2. Retry Strategy

Для временных ошибок применяется exponential backoff с ограничением количества попыток.

**Утверждённая начальная политика:**

- 1-я попытка — первоначальная доставка;
- 2-я попытка — через 1 минуту;
- 3-я попытка — через 5 минут;
- 4-я попытка — через 15 минут;
- 5-я попытка — через 60 минут.

После пятой неудачной попытки сообщение переводится в Dead-Letter Queue.

<text>Общее количество попыток: 5, включая первоначальную.</text>

Для реализации используются управляемые retry delays и настройки Cloudflare Queues. Конкретные параметры должны соответствовать ограничениям выбранной конфигурации Cloudflare.

### 10.3. Error Classification

**Transient Errors**

Примеры:

- HTTP 429;
- временные HTTP 5xx;
- network timeout;
- временная недоступность провайдера;
- database connection timeout.

Действие: повторная обработка по retry policy.

**Permanent Errors**

Примеры:

- некорректный payload;
- отсутствующий обязательный идентификатор;
- неподдерживаемый тип события;
- некорректная конфигурация получателя.

Действие: не выполнять бессмысленные повторы; фиксировать ошибку и переводить сообщение в контролируемую обработку.

### 10.4. Dead-Letter Queue

После исчерпания попыток:

1. Сообщение сохраняется в DLQ.
2. Создаётся error event.
3. Владелец получает Telegram notification для критических случаев.
4. Сообщение остаётся доступным для диагностики.
5. После исправления причины разрешается повторная обработка.
6. Повторная обработка должна сохранять идемпотентность.

### 10.5. Duplicate Prevention

Использовать:

- `notificationId`;
- idempotency key;
- уникальные идентификаторы бизнес-событий;
- проверку состояния notification delivery;
- обработку повторной доставки без дублирования бизнес-операций.

### 10.6. Reconciliation

Отдельный периодический процесс проверяет:

- Outbox events без подтверждённой отправки;
- сообщения, которые не были опубликованы в Queue;
- застрявшие notification records;
- несогласованные состояния delivery.

Рекомендуемая начальная периодичность reconciliation — каждые 5 минут.

## 11. Telegram Lead Management Bot

### 11.1. Purpose

Telegram Bot является основным операционным интерфейсом управления заявками в MVP.

Цель — обеспечить автоматизированную обработку входящих обращений без создания отдельной CRM или полноценной Admin Panel.

Neon PostgreSQL остаётся единственным источником истины.

### 11.2. Bot Responsibilities

Бот должен поддерживать:

1. Уведомление о новой заявке.
2. Просмотр списка новых заявок.
3. Просмотр карточки заявки.
4. Поиск по внутреннему ID и статусу.
5. Изменение статуса.
6. Добавление внутренней заметки.
7. Назначение следующего действия.
8. Установку напоминания.
9. Просмотр истории обработки.
10. Получение статистики по заявкам.

### 11.3. Initial Commands

Предлагаемые команды:

- `/start` — авторизация и главное меню.
- `/new` — новые заявки.
- `/leads` — список заявок.
- `/lead ID` — карточка заявки.
- `/search` — поиск.
- `/stats` — статистика.
- `/help` — справка.

Основной интерфейс — inline keyboard, без необходимости запоминать команды.

### 11.4. Lead Card

Карточка заявки должна содержать:

- Lead ID;
- имя клиента;
- телефон;
- Email, если предоставлен;
- услугу;
- предпочитаемый способ связи;
- сообщение;
- язык обращения;
- дату создания;
- landing page и UTM attribution;
- текущий статус;
- историю действий.

Персональные данные показываются только авторизованным пользователям.

### 11.5. Lead Status Lifecycle

Начальные статусы:

- `NEW` — новая заявка;
- `CONTACTED` — контакт установлен;
- `QUALIFIED` — обращение квалифицировано;
- `ESTIMATE_SENT` — отправлена оценка/предложение;
- `WON` — заказ получен;
- `LOST` — заказ не получен;
- `SPAM` — спам или нерелевантное обращение.

Изменение статуса фиксируется в `lead_events`.

### 11.6. Automated Follow-up

Бот может автоматически:

- напоминать о необработанных заявках;
- сообщать о заявках без следующего действия;
- формировать ежедневную сводку;
- напоминать о запланированном контакте;
- выявлять заявки, которые долго находятся в одном статусе.

Начальные правила:

- новая заявка — немедленное уведомление;
- новая заявка без изменения статуса в течение 2 часов — напоминание;
- заявка без обработки в течение 24 часов — повторное уведомление;
- ежедневная сводка — один раз в день.

Периоды и расписание можно будет изменить после накопления реальной статистики.

Бот не должен автоматически отправлять клиентам коммерческие предложения или сообщения без отдельного утверждённого сценария.

### 11.7. Security

- Доступ разрешён только заранее зарегистрированным Telegram user IDs.
- Неавторизованные пользователи не могут получать информацию о заявках.
- Webhook Telegram должен проверяться.
- Bot token хранится только в защищённых secrets.
- Все изменения статусов и заметок фиксируются в `lead_events`.
- API проверяет права пользователя при каждой операции.
- Прямого доступа Telegram Bot к PostgreSQL нет.
- Бот использует защищённый internal API.

### 11.8. API Integration

Для управления заявками предусматривается отдельный защищённый internal API.

Предлагаемые endpoints:

- `GET /api/v1/internal/leads`
- `GET /api/v1/internal/leads/{id}`
- `PATCH /api/v1/internal/leads/{id}/status`
- `POST /api/v1/internal/leads/{id}/notes`
- `POST /api/v1/internal/leads/{id}/reminders`
- `GET /api/v1/internal/stats`

Эти endpoints доступны только авторизованному боту и не являются публичным API.

Они требуют отдельной проверки авторизации, rate limiting и audit logging.

### 11.9. Future Automation

В дальнейшем можно добавить:

- автоматическую классификацию заявок по услугам;
- AI-generated lead summaries;
- приоритизацию по типу проекта;
- определение региона и языка обращения;
- подготовку ответа клиенту;
- интеграцию с календарём;
- автоматическое создание Deal после подтверждения заказа;
- расчёт комиссии по партнёрской модели.

AI-классификация не должна самостоятельно принимать окончательные бизнес-решения.

## 12. Incident Response Procedures

### 12.1. Production Website Unavailable

1. Проверить Cloudflare status и deployment status.
2. Проверить последние изменения в `main`.
3. Проверить GitHub Actions.
4. Проверить Worker logs и Sentry.
5. Определить связь с последним deployment.
6. При подтверждённой регрессии подготовить rollback.
7. Проверить главную страницу, услуги и проекты.
8. Проверить `/api/v1/health`.
9. Зафиксировать инцидент.

### 12.2. Lead API Unavailable

1. Проверить Worker logs и Sentry.
2. Проверить Neon availability.
3. Проверить connection string и secrets.
4. Проверить database connections.
5. Проверить последние API и DB changes.
6. Не запускать повторные migrations вслепую.
7. При необходимости откатить application release.
8. Проверить сохранение новых заявок после восстановления.
9. Зафиксировать результат.

**Data Integrity Rule:** нельзя сообщать клиенту об успешной отправке формы без подтверждённого сохранения заявки.

### 12.3. Database Unavailable

1. Проверить Neon status.
2. Проверить branch и connection string.
3. Проверить connection limits.
4. Проверить migrations.
5. Не пересоздавать production branch для устранения временной ошибки.
6. При необходимости использовать Neon recovery/PITR в рамках выбранного тарифа.
7. Проверить таблицы leads, lead_attribution, lead_events, notification_outbox, idempotency_keys.
8. Проверить API после восстановления.

### 12.4. Queue Processing Failure

1. Проверить Queue metrics.
2. Проверить Consumer logs.
3. Проверить PostgreSQL.
4. Проверить Resend и Telegram API.
5. Определить тип ошибки.
6. Проверить retry policy и DLQ.
7. Устранить причину.
8. Повторно обработать сообщения безопасным способом.
9. Проверить отсутствие дублирования.
10. Убедиться, что заявки сохранены.

### 12.5. Notification Provider Failure

При недоступности Resend или Telegram:

1. Проверить статус провайдера.
2. Проверить credentials и provider errors.
3. Проверить Queue backlog.
4. Убедиться, что событие сохранено в Outbox.
5. Повторить обработку по retry policy.
6. После восстановления проверить доставку.

Сбой уведомлений не должен влиять на сохранение заявки.

### 12.6. Turnstile Failure

1. Проверить Cloudflare Turnstile.
2. Проверить site key и secret.
3. Проверить серверную verification.
4. Проверить API и frontend errors.
5. Не отключать anti-spam защиту без решения владельца.
6. Использовать контролируемую recovery procedure.

### 12.7. Security Incident

1. Определить затронутые системы.
2. Ограничить скомпрометированный доступ.
3. Отозвать или заменить credentials.
4. Проверить GitHub, Cloudflare, Neon и CI/CD activity.
5. Сохранить необходимые audit logs.
6. Проверить целостность code и data.
7. Оценить возможное влияние на персональные данные.
8. Уведомить владельца.
9. Зафиксировать инцидент.

При нарушении защиты персональных данных отдельно оцениваются применимые требования GDPR и необходимость уведомления компетентного надзорного органа и затронутых лиц.

## 13. Database Operations

### 13.1. Migrations

Используются:

- Drizzle ORM;
- Drizzle Kit;
- SQL migration files;
- Squawk.

Правила:

- миграции создаются через PR;
- миграции проверяются до применения;
- production migration выполняется отдельным CI/CD этапом;
- используются ограниченные migration credentials;
- миграции не запускаются при старте Worker;
- применённые миграции не редактируются задним числом;
- destructive migrations требуют отдельной recovery procedure.

### 13.2. Manual Data Operations

- Frontend не подключается к PostgreSQL напрямую.
- Telegram Bot работает через internal API.
- Все изменения заявок фиксируются в `lead_events`.
- Production credentials не хранятся в Git.
- Для диагностики не используется privileged DB role без необходимости.

## 14. Backup and Recovery

### 14.1. Recovery Objectives

- RPO: 1 hour.
- RTO: 4 hours.

Это целевые показатели, которые должны быть проверены с учётом возможностей выбранного тарифа.

### 14.2. Backup Sources

**Neon PostgreSQL**

- backup/PITR;
- проверка retention;
- восстановление branch.

**Cloudflare R2**

- независимое хранение важных оригиналов;
- возможность повторной загрузки медиа;
- отдельная backup/versioning policy.

**GitHub**

- source code;
- migration history;
- deployment configuration.

### 14.3. Recovery Procedure

1. Определить масштаб повреждения.
2. Остановить действия, усугубляющие ситуацию.
3. Определить точку восстановления.
4. Проверить доступность backup.
5. Восстановить данные в отдельное окружение.
6. Проверить schema и integrity.
7. Проверить заявки за необходимый период.
8. Согласовать переключение production.
9. Проверить API и пользовательские сценарии.
10. Зафиксировать фактические RPO/RTO.

### 14.4. Restore Testing

Проверять:

- восстановление PostgreSQL;
- schema;
- целостность данных;
- восстановление медиа;
- работоспособность приложения.

Частота restore tests утверждается отдельно.

## 15. Deployment Operations

### 15.1. Deployment Rules

- постоянная ветка — `main`;
- рабочие ветки — `feature/*`, `fix/*`, `agent/*`, `hotfix/*`;
- все изменения проходят PR;
- `main` защищена;
- merge в `main` инициирует staging deployment;
- production deployment требует ручного approval через GitHub Environment `production`;
- production secrets защищены.

### 15.2. Pre-deployment Checklist

- [ ] PR approved.
- [ ] CI checks passed.
- [ ] TypeScript и Astro checks passed.
- [ ] Unit и integration tests passed.
- [ ] Security checks passed.
- [ ] Build completed.
- [ ] Migrations reviewed.
- [ ] Staging verified.
- [ ] Critical flows tested.
- [ ] Production approval obtained.

### 15.3. Post-deployment Checklist

- [ ] Worker deployment successful.
- [ ] Health endpoint works.
- [ ] Homepage loads.
- [ ] Localized pages load.
- [ ] Contact form works.
- [ ] Test lead saved in correct environment.
- [ ] Queue processing works.
- [ ] Notification integrations work.
- [ ] No new critical Sentry errors.
- [ ] No critical performance regression.

### 15.4. Rollback

1. Определить последний стабильный release.
2. Проверить совместимость schema.
3. Выполнить application rollback.
4. Проверить сайт и API.
5. Проверить DB connectivity.
6. Проверить Queue Consumer.
7. Проверить Sentry.
8. Зафиксировать rollback.

Database rollback не выполняется автоматически вместе с application rollback.

## 16. Performance Operations

### 16.1. Core Web Vitals

- LCP ≤ 2.5 seconds.
- INP ≤ 200 milliseconds.
- CLS ≤ 0.1.

### 16.2. Performance Monitoring

Контролировать:

- critical page loading;
- image optimization;
- JavaScript bundle size;
- React Islands hydration;
- third-party scripts;
- frontend errors;
- API latency.

Точные JS bundle budgets фиксируются после первого production build.

## 17. Security Operations

### 17.1. Access Control

- MFA для критических аккаунтов;
- Least Privilege;
- scoped API tokens;
- раздельные staging и production secrets;
- запрет хранения secrets в Git;
- регулярная проверка permissions;
- своевременный отзыв доступа.

### 17.2. Secrets Rotation

При подозрении на утечку:

1. Отозвать credential.
2. Выпустить новый.
3. Обновить Worker Secrets.
4. Проверить интеграции.
5. Проверить activity logs.
6. Зафиксировать событие.

### 17.3. Dependency Security

Использовать согласованные quality gates:

- Betterleaks;
- OSV-Scanner;
- Renovate после внедрения Stage 3.

Dependency updates проходят стандартный PR и CI/CD process.

## 18. Routine Operations

### 18.1. Daily Checks

- [ ] Проверка критических Sentry alerts.
- [ ] Проверка production availability.
- [ ] Проверка критических Queue errors.
- [ ] Проверка уведомлений Incident Bot.
- [ ] Проверка новых заявок и автоматических напоминаний.

### 18.2. Weekly Checks

- [ ] Queue backlog и DLQ.
- [ ] Database resource usage.
- [ ] Повторяющиеся application errors.
- [ ] Cloudflare Worker metrics.
- [ ] GitHub Actions failures.
- [ ] Staging health.
- [ ] Dependency security alerts.

### 18.3. Monthly Checks

- [ ] Проверка account access.
- [ ] Проверка credentials и tokens.
- [ ] Backup availability.
- [ ] Выборочная проверка восстановления.
- [ ] Performance trends.
- [ ] Sentry data sanitization.
- [ ] Incident records.
- [ ] Операционные расходы и лимиты.

## 19. Incident Communication

### 19.1. Owner Notification

Критические события отправляются владельцу через Telegram.

Сообщение должно включать severity, компонент, влияние, время обнаружения, предполагаемую причину и рекомендуемое действие.

### 19.2. Customer Communication

Если сбой влияет на приём заявок:

- не подтверждать отправку без сохранения данных;
- проверить форму после восстановления;
- при необходимости разместить уведомление;
- не раскрывать внутренние технические детали.

### 19.3. Post-Incident Review

Для SEV-1 и SEV-2:

1. Incident Summary.
2. Timeline.
3. Business Impact.
4. Root Cause.
5. Detection Gaps.
6. Resolution.
7. What Went Well.
8. What Went Wrong.
9. Corrective Actions.
10. Preventive Actions.

## 20. Operational Documentation

В защищённом месте должны храниться:

- Cloudflare project references;
- Neon project and branch references;
- GitHub repository and environments;
- Sentry project references;
- Queue configuration;
- R2 bucket configuration;
- Telegram Bot configuration;
- AI Incident Agent configuration;
- deployment procedures;
- database migration procedures;
- backup/recovery procedures;
- incident contacts.

OPS-001 не содержит credentials, API tokens, passwords или connection strings.

## 21. Operational Readiness Checklist

**Infrastructure**

- [ ] Production и staging разделены.
- [ ] Worker deployment работает.
- [ ] Neon production branch доступна.
- [ ] Queue и Consumer настроены.
- [ ] R2 доступен.
- [ ] Secrets разделены по окружениям.

**Monitoring**

- [ ] Sentry подключён.
- [ ] Source maps загружаются.
- [ ] Cloudflare logs доступны.
- [ ] Database monitoring настроен.
- [ ] Critical alerts протестированы.
- [ ] Telegram alerts работают.
- [ ] Incident Agent протестирован.

**Queue**

- [ ] Retry policy настроена.
- [ ] DLQ настроена.
- [ ] Idempotency проверена.
- [ ] Reconciliation работает.
- [ ] Повторная обработка проверена.

**Lead Management**

- [ ] Telegram Bot авторизует только разрешённых пользователей.
- [ ] Internal API защищён.
- [ ] Просмотр заявок работает.
- [ ] Изменение статуса работает.
- [ ] Notes и reminders сохраняются.
- [ ] Audit trail проверен.
- [ ] Автоматические напоминания работают.

**Recovery**

- [ ] Backup/PITR доступен.
- [ ] RPO/RTO документированы.
- [ ] Recovery procedure проверена.
- [ ] R2 recovery определена.
- [ ] Application rollback проверен.

**Security**

- [ ] MFA включена.
- [ ] Production credentials защищены.
- [ ] Branch protection настроена.
- [ ] Production approval настроен.
- [ ] Logging sanitization проверена.
- [ ] Security incident procedure доступна.

## 22. Approved Operational Decisions Register

### OD-OPS-001-01 — Incident Response Automation

**Decision: Approved**

AI Incident Response Agent (Cursor Automations или аналогичный инструмент) отвечает за автоматизированное обнаружение, первичную диагностику, классификацию и уведомление.

Владелец проекта является основным получателем критических Telegram alerts и принимает решения по критическим действиям production.

AI Agent не имеет права самостоятельно выполнять destructive production operations.

### OD-OPS-001-02 — Queue Retry Policy

**Decision: Approved**

Пять попыток доставки, включая первоначальную.

Retry delays:

- 1 minute;
- 5 minutes;
- 15 minutes;
- 60 minutes.

После пятой неудачи сообщение попадает в DLQ.

Reconciliation выполняется каждые 5 минут.

Consumer должен быть идемпотентным.

### OD-OPS-001-03 — Automated Lead Management

**Decision: Approved**

Telegram Bot является операционным интерфейсом управления заявками.

Neon PostgreSQL остаётся источником истины.

Бот работает через защищённый internal API и поддерживает просмотр заявок, изменение статусов, заметки, напоминания и статистику.

Все изменения фиксируются в `lead_events`.

Полноценная CRM/Admin Panel не входит в MVP.

### OD-OPS-001-04 — Automated Follow-up

**Decision: Approved**

- Немедленное уведомление о новой заявке.
- Напоминание через 2 часа без изменения статуса.
- Повторное уведомление через 24 часа.
- Ежедневная сводка один раз в день.

Автоматическая отправка коммерческих сообщений клиентам не входит в MVP.

## 23. Remaining Implementation Decisions

Следующие вопросы относятся к реализации и требуют технической проверки:

1. Выбор конкретного AI Incident Agent: Cursor Automations или альтернативный runtime.
2. Настройка webhook/MCP интеграции между Sentry, Incident Agent и Telegram.
3. Настройка точных Cloudflare Queue retry parameters.
4. Реализация internal API для Telegram Bot.
5. Реализация авторизации Telegram user IDs.
6. Выбор механизма периодического reconciliation.
7. Определение retention periods логов и Sentry events.
8. Проверка Neon backup/PITR возможностей на выбранном тарифе.
9. Определение точных alert thresholds.
10. Определение расписания ежедневной сводки с учётом часового пояса компании.

## 24. Acceptance Criteria

OPS-001 готов к утверждению, если:

1. Production-компоненты имеют определённые процедуры мониторинга.
2. Критические события обнаруживаются независимо от AI Agent.
3. AI Incident Agent отправляет критические уведомления владельцу.
4. AI Agent не имеет неконтролируемых production permissions.
5. Queue retry и DLQ работают согласно утверждённой политике.
6. Повторная обработка не приводит к дублированию бизнес-операций.
7. Telegram Bot обеспечивает безопасный доступ к заявкам.
8. Изменения заявок сохраняются в audit trail.
9. Автоматические reminders работают.
10. Backup и recovery procedures проверены.
11. Deployment и rollback документированы.
12. Production и staging credentials разделены.
13. RPO/RTO проверены или ограничения документированы.

## 25. Document Status

**Version:** 0.4
**Status:** Draft — Consolidated Operational Specification
**Owner:** Project Owner
**Approval:** Pending final technical validation

Документ связан с SAD-001, ADR-001, API-001, DATA-001 и DEPLOY-001.

При изменении архитектуры, провайдеров, deployment workflow или модели хранения данных OPS-001 должен быть пересмотрен.

---

**End of Document — OPS-001 v0.4 Consolidated**

## 26. Operational Consistency Updates (Consolidated from v0.3)

This section consolidates operational clarifications from the shorter v0.3 specification. If an older section is less specific or conflicts with this section, the clarification below takes precedence. These updates are requirements for the target design; they do not imply that the system has already been implemented.

### 26.1. Independent Direct Alert Path

Critical incident notification must not depend on the AI Incident Response Agent. The direct path is: Monitoring Source → Worker → Incident Application Service → PostgreSQL transaction (`incidents`, `incident_events`, `notification_outbox`) → `operations-events` Queue → Operations Consumer → Telegram Incident Bot → Owner.

The AI analysis path is separate and may run alongside or after direct alerting. If AI is unavailable, the incident record and direct critical notification flow must continue.

### 26.2. Separate Telegram Bot Scopes

Two operational bots are maintained as separate service scopes:

- **Lead Management Bot** — lead intake notifications and owner-facing lead operations.
- **Incident & Operations Bot** — incident alerts, operational events, and recovery notifications.

Each bot uses its own token, webhook secret, authorization scope, and configuration. Permissions are checked against the bot scope; access granted to one bot does not automatically grant access to the other.

### 26.3. Incident Severity Mapping

The consolidated document uses the SEV model from the detailed v0.2 runbook as the canonical classification. The severity labels from the short v0.3 update map as follows:

| Short v0.3 label | Canonical severity | Typical meaning |
| --- | --- | --- |
| CRITICAL | SEV-1 | Production unavailable, data loss/corruption, or serious security incident |
| ERROR | SEV-2 | Significant failure affecting lead processing or core application functionality |
| WARNING | SEV-3 | Partial degradation, elevated backlog, or risk requiring attention |
| INFO | SEV-4 / informational event | Non-urgent operational event such as a successful deployment; not necessarily an incident |

Severity and incident status remain separate concepts. Incident states are `OPEN`, `ACKNOWLEDGED`, `RESOLVED`, and `IGNORED`; lifecycle may include detection, triage, containment, investigation, resolution, recovery, verification, and postmortem.

### 26.4. Queue Boundaries and Processing Guarantees

The logical queues are:

- `lead-notifications` — new-lead notifications, follow-up and lead-related notification events.
- `operations-events` — incident notifications, technical alerts, operational and diagnostic events.
- `lead-commands` — deferred and not part of mandatory MVP configuration.

Transactional Outbox remains the durable source of publishable events. Consumers must validate event schema, preserve Correlation ID, be idempotent, record delivery outcomes, and avoid repeating business operations on duplicate delivery. The retry policy remains five total attempts (initial delivery plus four retries), with target delays of 1, 5, 15, and 60 minutes; Cloudflare Queue retry semantics must be verified during implementation. Reconciliation is targeted every five minutes and must not blindly republish all old records.

### 26.5. Reminder Processing Consistency

Reminder processing is targeted to run every minute. Automated lead follow-up is checked on a five-minute cadence, with reminders at two and twenty-four hours after lead creation when the lead has not progressed, plus a once-daily summary and owner-created reminders. Exact daily summary time and company timezone are configuration decisions.

Reminder processing must use atomic database claiming, a processing lease, and idempotency. Expired leases may return work to retryable processing after checking delivery history. Automated reminders are cancelled when a lead becomes `WON`, `LOST`, or `SPAM`. Reminder status lifecycle: `PENDING`, `PROCESSING`, `SENT`, `COMPLETED`, `CANCELLED`, `FAILED`.

### 26.6. Operational Security and Traceability

All significant actions should be auditable. Correlation ID should connect the request/update, application service, database transaction, Outbox event, Queue message, consumer, provider delivery, and incident event. Logs must be structured and sanitized. Telegram user IDs are allowlisted; bot scopes are isolated; internal API is authenticated, rate-limited, and audited. AI receives only sanitized, necessary technical context and remains read-only by default with no direct database access, secrets, destructive actions, or autonomous production changes.

### 26.7. Operational Metrics Baseline

At minimum, monitor and expose metrics for API request count, latency, error rate and rate limits; database availability, connection failures, query latency and transaction errors; queue backlog, consumer errors, retries, DLQ count and Outbox age; notification success, latency and provider errors; reminder pending/overdue/failed counts, expired leases and duplicate-prevention events; incident count, severity, acknowledgement time, resolution time and recurrence. Numerical alert thresholds remain open decisions and must be set from baseline observations.

### 26.8. Owner and Service Responsibilities

- **Owner:** receives critical alerts, manages leads, acknowledges incidents, approves production deployment, and decides on production recovery.
- **Application Services:** enforce authorization, execute business operations, persist state, and manage state transitions.
- **Scheduled Worker:** reminder checks, follow-up, daily summary, and reconciliation.
- **Queue Consumers:** event processing, notification delivery, delivery result persistence, and retry handling.
- **AI Agent:** technical analysis and recommendations only; no dangerous production actions.

### 26.9. Open Operational Decisions Consolidated

The following remain unresolved/configurable and are not silently treated as approved implementation details:

- Daily summary time and timezone.
- Queue backlog and Outbox age alert thresholds.
- Final R2 backup/versioning and recovery policy.
- AI Agent provider and integration method.
- Alternate critical alert channel if Telegram is unavailable.
- Restore-test cadence.
- Log and Sentry retention periods.
- Final alert thresholds and scheduled reconciliation implementation.
- Provider plan limits and exact Cloudflare Queue configuration.

### 26.10. Consolidation Notes

- The detailed v0.2 runbook is the primary base because it contains the broader operational scope, procedures, security requirements, checklists, and approved operational decisions.
- The shorter v0.3 specification is incorporated as clarifications and consistency updates in this section, including direct alert independence, separate bot scopes, queue boundaries, reminder concurrency, reconciliation safety, operational metrics, and responsibility model.
- Where both versions repeat the same rule, it is retained once in the main body and referenced here only when needed to clarify precedence.
- This consolidated v0.4 is a planning/specification document. It does not claim that integrations, monitoring, queues, bots, runbooks, backups, or recovery tests are already deployed or verified.

## Change Log (Consolidated)

| Version | Date | Changes |
| --- | --- | --- |
| 0.1 | 2026-11-17 | Initial operations model |
| 0.2 | 2026-11-18 | AI Agent, Lead Bot, retry policy, reminders, monitoring, operational procedures and readiness checklists |
| 0.3 | 2026-11-19 | Direct Alert and AI Analysis separation; queue boundaries; reminder concurrency; Outbox reconciliation; Telegram permissions; correlation, recovery, and responsibilities clarified |
| 0.4 | 2026-11-20 | Consolidated detailed v0.2 and consistency updates from v0.3 into one expanded document; severity mapping and unresolved decisions made explicit |

**End of Document — OPS-001 v0.4 Consolidated**
