# IMPLEMENTATION-001 — План реализации системы

**Проект:** «Братья» / եղբայրներ  
**Версия:** 0.2  
**Статус:** Draft  
**Тип документа:** Implementation Plan  
**Связанные документы:** BRD-001, SRS-001, SAD-001, API-001, DATA-001, ADR-001, DEPLOY-001, OPS-001, TECH-001, TEST-001

---

## 1. Цель документа

Документ определяет последовательность технической реализации серверной инфраструктуры и операционной системы обработки заявок.

Цель — реализовать MVP, включающий:

- Public API для приёма заявок с сайта.
- PostgreSQL как основной источник данных.
- Application Services внутри Cloudflare Worker.
- Transactional Outbox и Cloudflare Queues.
- Два независимых Telegram-бота.
- Уведомления через Telegram и Resend.
- Управление заявками через Lead Management Bot.
- Мониторинг инцидентов через Incident & Operations Bot.
- Scheduled Jobs для напоминаний и сверки Outbox.
- Независимый Direct Alert Path.
- AI Analysis как отдельный необязательный операционный компонент.
- CI/CD и контролируемое развёртывание.

Реализация ведётся последовательно. Каждый этап должен завершаться проверяемым результатом и критериями приёмки.

## 2. Базовые архитектурные ограничения

### 2.1. Технологический стек

| Область | Технология |
| --- | --- |
| Runtime | Cloudflare Workers |
| Backend | TypeScript |
| Validation | Zod |
| Database | Neon PostgreSQL |
| ORM | Drizzle ORM |
| Migrations | Drizzle Kit |
| Async processing | Cloudflare Queues |
| Media storage | Cloudflare R2 |
| Email | Resend |
| Operations interface | Telegram Bot API |
| Anti-spam | Cloudflare Turnstile |
| Monitoring | Sentry + Cloudflare Observability |
| CI/CD | GitHub Actions |
| Package manager | pnpm |
| Toolchain | mise |
| Git Hooks | hk |
| SQL lint | Squawk |

### 2.2. Обязательные архитектурные правила

1. Application Services находятся внутри Cloudflare Worker.
2. HTTP handlers, Telegram webhook handlers, Queue Consumers и Scheduled Jobs вызывают Application Services напрямую.
3. Внутренние операции не должны обращаться к собственному Worker через HTTP.
4. PostgreSQL является источником истины.
5. Внешние уведомления не отправляются внутри DB-транзакций.
6. Для асинхронной доставки используется Transactional Outbox + Cloudflare Queues.
7. Обработка событий должна быть идемпотентной.
8. Correlation ID проходит через всю цепочку обработки.
9. Direct Alert не зависит от доступности AI Agent.
10. AI Agent не получает прямого доступа к базе данных и не выполняет destructive production operations.
11. Два Telegram-бота имеют независимые токены, webhook secrets и наборы разрешений.
12. Production deployment требует отдельного разрешения через GitHub Environment.

## 3. Предварительный этап — Repository & Environment Preparation

**Цель:** подготовить единый процесс разработки и воспроизводимое окружение.

### Задачи

- [ ] Проверить существующую структуру GitHub-репозитория.
- [ ] Зафиксировать Node.js через mise.
- [ ] Настроить pnpm и `pnpm-lock.yaml`.
- [ ] Добавить `packageManager` в `package.json`.
- [ ] Настроить TypeScript strict mode.
- [ ] Настроить Oxlint и oxfmt.
- [ ] Настроить Stylelint, если проект содержит SCSS.
- [ ] Настроить Vitest.
- [ ] Настроить hk hooks.
- [ ] Настроить commitlint.
- [ ] Подключить Betterleaks.
- [ ] Настроить GitHub Actions.
- [ ] Защитить ветку `main`.
- [ ] Ввести Pull Request workflow.
- [ ] Создать отдельные GitHub Environments: `staging` и `production`.

### Git workflow

Постоянная ветка:

`main`

Краткосрочные ветки:

- `feature/*`
- `fix/*`
- `agent/*`
- `hotfix/*`

Прямые изменения в `main` запрещены. Все изменения проходят через Pull Request и CI checks.

### Результат

Воспроизводимое окружение разработки и CI pipeline с базовыми quality gates.

### Acceptance Criteria

- [ ] Проект устанавливается через `pnpm install --frozen-lockfile`.
- [ ] Локальные и CI-инструменты используют согласованные версии.
- [ ] Hooks выполняются локально.
- [ ] GitHub Actions проверяет Pull Requests.
- [ ] `main` защищена от прямых изменений.

---

## 4. Этап 1 — Cloudflare & Neon Infrastructure

**Цель:** подготовить инфраструктуру для staging и production.

### Cloudflare

- [ ] Создать или проверить Cloudflare Worker.
- [ ] Настроить Static Assets, если Worker обслуживает статические ресурсы.
- [ ] Создать Queue `lead-notifications`.
- [ ] Создать Queue `operations-events`.
- [ ] Настроить Queue Consumers.
- [ ] Создать R2 bucket для публичных медиа.
- [ ] Подготовить отдельное хранилище для приватных файлов, если оно необходимо.
- [ ] Настроить Scheduled Worker.
- [ ] Настроить Turnstile.
- [ ] Настроить DNS, CDN и WAF.
- [ ] Подготовить отдельные Cloudflare resources для staging и production.

### Neon

- [ ] Создать Neon project.
- [ ] Подготовить production branch.
- [ ] Подготовить staging branch.
- [ ] Настроить отдельные credentials.
- [ ] Проверить PostgreSQL connection из Cloudflare Worker.
- [ ] Проверить совместимость выбранного PostgreSQL driver с Cloudflare Workers и Neon.
- [ ] Настроить резервное копирование согласно доступному тарифу Neon.

### Секреты

Секреты хранятся отдельно для каждого окружения.

Примеры:

- `DATABASE_URL`
- `RESEND_API_KEY`
- `TELEGRAM_LEADS_BOT_TOKEN`
- `TELEGRAM_LEADS_WEBHOOK_SECRET`
- `TELEGRAM_OPS_BOT_TOKEN`
- `TELEGRAM_OPS_WEBHOOK_SECRET`
- `TURNSTILE_SECRET_KEY`
- `SENTRY_DSN`

Точные имена и разделение публичных и секретных переменных должны соответствовать реализации Worker.

Секреты не хранятся в Git, `.env.example` или клиентском bundle.

### Результат

Работающие staging и production инфраструктурные ресурсы с раздельными credentials.

### Acceptance Criteria

- [ ] Worker подключается к staging Neon.
- [ ] Queue resources созданы.
- [ ] R2 доступен Worker.
- [ ] Секреты отсутствуют в репозитории.
- [ ] Production и staging используют независимые ресурсы и ключи.

---

## 5. Этап 2 — Backend Foundation

**Цель:** создать основу серверного приложения.

### Предлагаемая структура

```text
src/
├── application/
│   ├── services/
│   │   ├── leads/
│   │   ├── incidents/
│   │   ├── notifications/
│   │   ├── reminders/
│   │   └── operations/
│   └── contracts/
│
├── domain/
│   ├── leads/
│   ├── incidents/
│   ├── notifications/
│   └── shared/
│
├── infrastructure/
│   ├── database/
│   ├── repositories/
│   ├── queues/
│   ├── email/
│   ├── telegram/
│   ├── storage/
│   └── monitoring/
│
├── interfaces/
│   ├── http/
│   ├── telegram/
│   ├── queues/
│   └── scheduled/
│
├── shared/
│   ├── errors/
│   ├── validation/
│   ├── correlation/
│   ├── idempotency/
│   └── logging/
│
└── index.ts
```

Это логическое разделение приложения. Оно не предполагает отдельные Worker deployments для каждого модуля.

### Задачи

- [ ] Настроить Worker entry point.
- [ ] Создать конфигурацию окружений.
- [ ] Подключить Drizzle ORM.
- [ ] Настроить Drizzle Kit.
- [ ] Подготовить database client.
- [ ] Создать общий error handling.
- [ ] Создать Zod validation layer.
- [ ] Реализовать Correlation ID middleware.
- [ ] Подготовить структурированное логирование.
- [ ] Реализовать базовые Application Services.
- [ ] Реализовать health endpoint.

### Correlation ID

Единый Correlation ID должен проходить через:

`HTTP → Application Service → Database → Outbox → Queue → Consumer → Resend / Telegram`

Требования:

- генерировать ID, если входящий ID отсутствует;
- проверять формат внешнего ID;
- сохранять ID в event envelope;
- включать ID в структурированные логи;
- использовать ID для поиска связанных событий;
- не смешивать Correlation ID с eventId и idempotencyKey.

### Результат

Базовая серверная архитектура с единым механизмом обработки ошибок, валидации и трассировки.

### Acceptance Criteria

- [ ] Worker запускается в staging.
- [ ] `/api/v1/health` возвращает корректный статус.
- [ ] Correlation ID доступен в логах.
- [ ] Application Services вызываются напрямую из handlers.
- [ ] Нет внутренних HTTP self-calls.

---

## 6. Этап 3 — Database Schema & Migrations

**Цель:** реализовать и проверить схему данных.

### Критическая предварительная проверка

До создания новых FK-миграций необходимо проверить фактическую существующую Drizzle schema и PostgreSQL DDL.

Особенно важно установить реальные типы:

- `leads.id`
- `bot_users.id`

**UUID нельзя считать подтверждённым типом.** Все внешние ключи и SQL-примеры должны соответствовать реальной схеме.

### Таблицы

Основные сущности:

- `leads`
- `lead_attribution`
- `lead_events`
- `notification_outbox`
- `idempotency_keys`
- `bot_users`
- `bot_user_permissions`
- `telegram_updates`
- `lead_notes`
- `lead_reminders`
- `bot_notification_deliveries`
- `incidents`
- `incident_events`

### Задачи

- [ ] Проверить текущую Drizzle schema.
- [ ] Проверить реальные PK/FK типы.
- [ ] Проверить существующие constraints и indexes.
- [ ] Согласовать схему с DATA-001.
- [ ] Добавить недостающие таблицы.
- [ ] Добавить необходимые индексы.
- [ ] Настроить Drizzle migrations.
- [ ] Подключить Squawk для проверки SQL.
- [ ] Подготовить миграции для staging.
- [ ] Проверить rollback strategy.

### Напоминания

Состояния:

- `PENDING`
- `PROCESSING`
- `SENT`
- `COMPLETED`
- `CANCELLED`
- `FAILED`

Для конкурентно-безопасной обработки используются:

- `attempt_count`
- `lease_until`
- `idempotency_key`

Напоминания должны атомарно захватываться обработчиком. Для PostgreSQL может применяться `FOR UPDATE SKIP LOCKED`.

При переводе заявки в `WON`, `LOST` или `SPAM` нерелевантные автоматические напоминания отменяются.

### Миграции

- Миграции являются неизменяемыми после применения.
- Изменения схемы выполняются через новые миграции.
- Миграции не запускаются при старте Worker.
- Для миграций используется отдельная CI/CD стадия и ограниченные credentials.
- Перед production migration обязательна проверка staging.

### Результат

Согласованная схема базы данных и воспроизводимые миграции.

### Acceptance Criteria

- [ ] PK/FK типы проверены по фактической схеме.
- [ ] Миграции успешно применяются к staging.
- [ ] Squawk не выявляет блокирующих SQL-проблем.
- [ ] Constraints и индексы проверены.
- [ ] Проверены повторные операции и уникальность idempotency keys.

---

## 7. Этап 4 — Authentication & Telegram Authorization

**Цель:** обеспечить безопасный доступ двух Telegram-ботов.

### Lead Management Bot

Предназначен для работы с заявками.

### Incident & Operations Bot

Предназначен для мониторинга и управления инцидентами.

### Задачи

- [ ] Создать два независимых Telegram Bot.
- [ ] Настроить отдельные токены.
- [ ] Настроить отдельные webhook secrets.
- [ ] Реализовать проверку webhook secret.
- [ ] Реализовать Telegram User ID allowlist.
- [ ] Реализовать scoped permissions.
- [ ] Реализовать проверку прав на каждую операцию.
- [ ] Сохранять обработанные Telegram updates.
- [ ] Реализовать защиту от повторной обработки updates.

### Разделение проверок

Webhook secret подтверждает подлинность входящего webhook-запроса.

Telegram User ID allowlist и permissions определяют, имеет ли конкретный пользователь право выполнять операцию.

Одна проверка не заменяет другую.

### Результат

Два независимых канала Telegram с раздельной авторизацией.

### Acceptance Criteria

- [ ] Неавторизованный пользователь не может выполнять команды.
- [ ] Пользователь Lead Bot не получает автоматически права Operations Bot.
- [ ] Повторный Telegram update не приводит к повторной операции.
- [ ] Webhook secrets проверяются отдельно для каждого бота.

---

## 8. Этап 5 — Public Lead API

**Цель:** реализовать надёжный приём заявок с сайта.

### Endpoint

`POST /api/v1/leads`

Дополнительный endpoint:

`GET /api/v1/health`

### Задачи

- [ ] Создать lead schema через Zod.
- [ ] Реализовать валидацию входящих данных.
- [ ] Подключить Cloudflare Turnstile.
- [ ] Реализовать rate limiting.
- [ ] Реализовать idempotency.
- [ ] Сохранять заявку в PostgreSQL.
- [ ] Сохранять минимальную lead attribution.
- [ ] Записывать lead event.
- [ ] Создавать notification outbox event.
- [ ] Возвращать корректный API response.
- [ ] Добавить интеграционные тесты.

### Минимальная атрибуция

Атрибуция фиксируется при отправке формы.

В MVP не реализуются:

- Cross-session tracking.
- Fingerprinting.
- First Touch Attribution.
- Last Touch Attribution.

### Транзакционная граница

В одной PostgreSQL-транзакции сохраняются:

1. Lead.
2. Attribution.
3. Lead event.
4. Notification outbox event.
5. Idempotency record, если он относится к данной операции.

Внешние вызовы Resend и Telegram внутри этой транзакции запрещены.

### Результат

Работающий Public Lead API с защитой от спама и дублирования.

### Acceptance Criteria

- [ ] Валидная заявка сохраняется.
- [ ] Невалидная заявка отклоняется.
- [ ] Turnstile проверяется серверной стороной.
- [ ] Повторный запрос с тем же idempotency key не создаёт дубликат.
- [ ] Outbox event создаётся атомарно с заявкой.
- [ ] Correlation ID сохраняется в связанных событиях.

---

## 9. Этап 6 — Lead Management Bot

**Цель:** обеспечить ручное управление заявками без отдельной CRM.

### Основные функции

- Уведомление о новой заявке.
- Список заявок.
- Детальная информация.
- Изменение статуса.
- Добавление заметок.
- Создание напоминаний.
- Просмотр статистики.

### Статусы

- `NEW`
- `CONTACTED`
- `QUALIFIED`
- `ESTIMATE_SENT`
- `WON`
- `LOST`
- `SPAM`

### Webhook

`POST /api/v1/webhooks/telegram/leads`

### Задачи

- [ ] Реализовать webhook handler.
- [ ] Подключить Lead Application Services.
- [ ] Реализовать команды и inline keyboard.
- [ ] Реализовать список заявок.
- [ ] Реализовать просмотр заявки.
- [ ] Реализовать изменение статуса.
- [ ] Реализовать заметки.
- [ ] Реализовать создание напоминаний.
- [ ] Реализовать статистику.
- [ ] Добавить аудит действий.

### Результат

Lead Management Bot позволяет работать с заявками через Telegram.

### Acceptance Criteria

- [ ] Новая заявка отображается в Telegram.
- [ ] Статус заявки изменяется и сохраняется.
- [ ] Заметки сохраняются.
- [ ] Напоминания создаются.
- [ ] Все операции проверяют права пользователя.
- [ ] Действия пользователей фиксируются.

---

## 10. Этап 7 — Transactional Outbox & Cloudflare Queues

**Цель:** обеспечить надёжную асинхронную доставку уведомлений.

### Очереди

`lead-notifications`

Ответственность:

- уведомления о новых заявках;
- email через Resend;
- сообщения Lead Management Bot.

`operations-events`

Ответственность:

- операционные события;
- критические уведомления;
- события мониторинга и инцидентов.

`lead-commands` — отложено, не входит в MVP.

### Задачи

- [ ] Реализовать Outbox dispatcher.
- [ ] Реализовать публикацию событий в Queues.
- [ ] Реализовать Consumers.
- [ ] Реализовать idempotent processing.
- [ ] Реализовать retry handling.
- [ ] Реализовать DLQ workflow.
- [ ] Реализовать Outbox reconciliation.
- [ ] Добавить мониторинг возраста необработанных событий.

### Retry policy

Целевая политика:

- 5 попыток, включая первоначальную;
- задержки: 1, 5, 15 и 60 минут;
- после исчерпания попыток — DLQ;
- reconciliation каждые 5 минут.

Это целевые параметры политики приложения. Конкретная реализация должна учитывать реальные возможности и ограничения Cloudflare Queues, включая настройки retry и DLQ.

### Важное ограничение

PostgreSQL transaction и Cloudflare Queue publish не являются общей атомарной транзакцией.

Поэтому система должна предусматривать:

- сохранение события в Outbox;
- публикацию события в Queue;
- безопасную повторную публикацию;
- идемпотентность Consumer;
- периодическую сверку Outbox;
- обработку событий, которые были сохранены, но не доставлены в Queue.

### Результат

Надёжная система асинхронной доставки событий.

### Acceptance Criteria

- [ ] Outbox events публикуются в Queue.
- [ ] Повторная доставка не вызывает дублирования бизнес-операций.
- [ ] Ошибки доставки обрабатываются.
- [ ] DLQ содержит события после исчерпания попыток.
- [ ] Reconciliation обнаруживает зависшие события.
- [ ] Correlation ID сохраняется в event envelope.

---

## 11. Этап 8 — Scheduled Jobs & Reminder Engine

**Цель:** автоматизировать напоминания и контроль обработки заявок.

### Плановые операции

- Немедленное уведомление о новой заявке.
- Напоминание через 2 часа, если статус не изменился.
- Напоминание через 24 часа.
- Ежедневная сводка.
- Ручные follow-up reminders.
- Outbox reconciliation каждые 5 минут.
- Восстановление истёкших processing leases.

### Задачи

- [ ] Реализовать Scheduled Worker handler.
- [ ] Реализовать Reminder Service.
- [ ] Реализовать атомарное получение задач.
- [ ] Реализовать lease expiration recovery.
- [ ] Реализовать повторную обработку.
- [ ] Реализовать отмену нерелевантных напоминаний.
- [ ] Реализовать ежедневную сводку.
- [ ] Добавить тесты конкурентной обработки.

### Конкурентная обработка

Одна reminder task не должна отправляться несколькими параллельными обработчиками.

Для этого используются атомарное получение задачи, статус `PROCESSING`, `lease_until` и idempotency key.

### Результат

Автоматизированная система напоминаний и периодических задач.

### Acceptance Criteria

- [ ] Одно напоминание не отправляется повторно из-за конкурентной обработки.
- [ ] Истёкший lease позволяет восстановить обработку.
- [ ] Завершённые заявки не получают нерелевантные автоматические напоминания.
- [ ] Ошибки доставки регистрируются.
- [ ] Плановые задачи наблюдаемы через логи и метрики.

---

## 12. Этап 9 — Incident & Operations Bot

**Цель:** создать отдельный канал мониторинга и управления операционными инцидентами.

### Webhook

`POST /api/v1/webhooks/telegram/incident`

### Основные функции

- Просмотр инцидентов.
- Детальная информация.
- Подтверждение инцидента — ACK.
- Закрытие инцидента — Resolve.
- Просмотр состояния операций.
- Получение критических уведомлений.

### Задачи

- [ ] Реализовать Incident Application Services.
- [ ] Реализовать создание инцидента.
- [ ] Реализовать дедупликацию.
- [ ] Реализовать ACK.
- [ ] Реализовать Resolve.
- [ ] Реализовать Operations Status.
- [ ] Реализовать отдельные permissions.
- [ ] Реализовать уведомления через `operations-events`.
- [ ] Добавить аудит действий.

### Дедупликация

При наличии внешнего ID:

`source + external_id + environment`

При отсутствии внешнего ID:

`fingerprint + environment`

### Результат

Работающий Operations Bot с независимым контролем доступа и журналом инцидентов.

### Acceptance Criteria

- [ ] Инциденты сохраняются в PostgreSQL.
- [ ] Повторные события корректно дедуплицируются.
- [ ] ACK и Resolve изменяют состояние инцидента.
- [ ] Все действия фиксируются.
- [ ] Уведомления не зависят от Lead Management Bot.

---

## 13. Этап 10 — Direct Alert & AI Analysis

**Цель:** разделить критические уведомления и AI-диагностику.

### Direct Alert Path

Критические уведомления должны работать независимо от AI Agent.

Задачи:

- [ ] Определить критические события.
- [ ] Реализовать прямую маршрутизацию.
- [ ] Подключить Operations Bot.
- [ ] Реализовать подтверждение инцидента.
- [ ] Настроить fallback и регистрацию ошибок доставки.

### AI Analysis

AI Agent может использоваться для:

- анализа технических логов;
- группировки ошибок;
- подготовки диагностических сводок;
- классификации инцидентов;
- рекомендаций по устранению проблем.

Ограничения:

- Нет прямого доступа к PostgreSQL.
- Нет доступа к секретам.
- Нет самостоятельного изменения production configuration.
- Нет destructive operations.
- Нет права одобрять production deployment.
- Перед отправкой диагностических данных внешнему AI provider выполняется sanitization.

### Результат

Критические уведомления работают без AI. AI используется только как дополнительный аналитический инструмент.

### Acceptance Criteria

- [ ] Direct Alert работает при недоступности AI.
- [ ] AI не блокирует создание и обработку инцидентов.
- [ ] Секреты исключены из AI-контекста.
- [ ] AI не выполняет production actions.
- [ ] AI-рекомендации не считаются подтверждённым исправлением.

---

## 14. Этап 11 — Monitoring & Observability

**Цель:** обеспечить видимость состояния приложения и инфраструктуры.

### Инструменты

- Sentry.
- Cloudflare Observability.
- Neon Monitoring.
- GitHub Actions.
- Telegram Operations Bot.

### Задачи

- [ ] Подключить Sentry.
- [ ] Настроить error tracking.
- [ ] Настроить структурированные логи.
- [ ] Добавить Correlation ID в диагностические события.
- [ ] Контролировать Queue backlog.
- [ ] Контролировать Outbox age.
- [ ] Контролировать ошибки Consumers.
- [ ] Контролировать ошибки Resend и Telegram.
- [ ] Контролировать доступность Neon.
- [ ] Настроить критические alert rules.

### Результат

Централизованное наблюдение за ошибками и операционным состоянием.

### Acceptance Criteria

- [ ] Ошибки приложения видны в Sentry.
- [ ] Связанные события можно найти по Correlation ID.
- [ ] Состояние Queues и Outbox контролируется.
- [ ] Критические события поступают в Operations Bot.
- [ ] Диагностические данные не содержат секретов.

---

## 15. Этап 12 — Testing & Security Validation

**Цель:** проверить корректность, безопасность и устойчивость MVP.

### Тестирование

- Unit tests.
- Integration tests.
- API tests.
- Database tests.
- Queue/Outbox tests.
- Telegram webhook tests.
- Reminder concurrency tests.
- Security tests.
- E2E tests.
- Staging smoke tests.

### Инструменты

- Vitest.
- Testing Library.
- Playwright.
- axe-core.
- Squawk.
- OSV-Scanner.

### Обязательные сценарии

1. Успешная отправка заявки.
2. Ошибка валидации.
3. Turnstile rejection.
4. Повторный запрос с тем же idempotency key.
5. Ошибка записи в PostgreSQL.
6. Сохранение Outbox event.
7. Ошибка Queue publish.
8. Повторная публикация события.
9. Повторная доставка Consumer.
10. Ошибка Resend.
11. Ошибка Telegram API.
12. Недоступность AI Agent.
13. Прямое критическое уведомление без AI.
14. Неавторизованный Telegram User ID.
15. Неверный webhook secret.
16. Повторный Telegram update.
17. Конкурентное получение reminder task.
18. Восстановление истёкшего lease.
19. Дедупликация инцидента.
20. Отмена автоматических напоминаний после завершения заявки.

### Security

- Проверка webhook secrets.
- Проверка Telegram User ID и permissions.
- Rate limiting.
- Turnstile.
- Idempotency.
- Secret scanning.
- Dependency vulnerability scanning.
- Проверка SQL migrations.
- Проверка логирования чувствительных данных.
- Проверка разделения staging и production credentials.

### Результат

Подтверждённая работоспособность основных сценариев и отсутствие известных блокирующих дефектов.

---

## 16. Этап 13 — Staging Deployment

**Цель:** проверить всю систему в отдельном окружении до production.

### Порядок

1. Проверить GitHub Actions.
2. Применить миграции к staging Neon.
3. Развернуть Worker.
4. Проверить API.
5. Проверить R2.
6. Проверить Queues и Consumers.
7. Зарегистрировать Telegram webhooks staging-ботов.
8. Проверить Resend.
9. Проверить Turnstile.
10. Проверить Scheduled Jobs.
11. Проверить Lead Management Bot.
12. Проверить Incident & Operations Bot.
13. Провести E2E tests.
14. Провести security smoke tests.
15. Проверить мониторинг.

### Acceptance Criteria

- [ ] Все основные сценарии проходят в staging.
- [ ] Оба бота работают независимо.
- [ ] Очереди и плановые задачи работают.
- [ ] Уведомления доставляются.
- [ ] Ошибки видны в Sentry.
- [ ] Нет критических дефектов.

---

## 17. Этап 14 — Production Deployment

**Цель:** безопасно запустить MVP.

### Обязательный порядок

1. Создать release candidate.
2. Проверить успешное прохождение CI.
3. Проверить staging acceptance.
4. Проверить миграции.
5. Проверить production secrets.
6. Проверить Cloudflare production resources.
7. Проверить Neon production connection.
8. Применить production migrations отдельной CI/CD стадией.
9. Выполнить Worker deployment.
10. Зарегистрировать production webhooks двух ботов.
11. Проверить Queues и Consumers.
12. Проверить Scheduled Jobs.
13. Выполнить smoke tests.
14. Проверить Direct Alert.
15. Проверить мониторинг.
16. Зафиксировать release и deployment result.

### Production approval

Production deployment проходит через GitHub Environment `production`.

AI Agent и автоматические процессы не имеют права самостоятельно одобрять production deployment.

### Acceptance Criteria

- [ ] Production deployment одобрен уполномоченным человеком.
- [ ] Миграции выполнены отдельно от запуска Worker.
- [ ] Все production secrets настроены.
- [ ] Оба Telegram-бота работают.
- [ ] Direct Alert проверен.
- [ ] Smoke tests пройдены.
- [ ] Зафиксирована версия релиза.

---

## 18. Backup & Disaster Recovery

Целевые параметры:

- RPO: 1 час.
- RTO: 4 часа.

### Задачи

- [ ] Проверить доступные возможности Neon backup/PITR.
- [ ] Зафиксировать политику восстановления PostgreSQL.
- [ ] Определить независимую политику резервного копирования R2.
- [ ] Проверить доступность резервных копий.
- [ ] Документировать восстановление staging.
- [ ] Провести restore test.
- [ ] Документировать восстановление production.
- [ ] Зафиксировать результаты тестов.

Параметры RPO/RTO являются целевыми требованиями и должны быть подтверждены фактическими возможностями выбранных тарифов и процедур восстановления.

---

## 19. Implementation Dependencies

Последовательность реализации:

```text
Repository & CI
      ↓
Cloudflare & Neon
      ↓
Backend Foundation
      ↓
Database Schema
      ↓
Authentication & Authorization
      ↓
Public Lead API
      ↓
Lead Management Bot
      ↓
Transactional Outbox & Queues
      ↓
Scheduled Jobs & Reminders
      ↓
Incident & Operations Bot
      ↓
Direct Alert & AI Analysis
      ↓
Monitoring & Testing
      ↓
Staging
      ↓
Production
```

Некоторые подготовительные работы могут выполняться параллельно, но production deployment зависит от завершения всех обязательных проверок.

## 20. Quality Gates

### Gate 1 — Foundation

- TypeScript strict.
- Oxlint.
- oxfmt.
- Astro Check, если затронут Astro frontend.
- Stylelint, если затронут SCSS.
- Vitest.
- Betterleaks.
- commitlint.

### Gate 2 — Application

- Integration tests.
- API tests.
- Database tests.
- Queue/Outbox tests.
- Playwright.
- axe-core.
- Squawk.
- OSV-Scanner.
- Sentry.
- size-limit.

### Gate 3 — Release

- Staging acceptance.
- Security smoke tests.
- Migration validation.
- Production approval.
- Production smoke tests.
- Monitoring verification.

PR-Agent используется только как advisory-инструмент. Автоматическое слияние и production approval не разрешаются.

## 21. Definition of Done

Этап считается завершённым, если:

1. Реализация соответствует связанным архитектурным документам.
2. Код проходит обязательные quality gates.
3. Добавлены необходимые тесты.
4. Нет открытых блокирующих дефектов.
5. Обновлена техническая документация.
6. Изменения прошли Pull Request review.
7. Результат проверен в соответствующем окружении.

## 22. Open Decisions

Перед завершением реализации необходимо закрыть следующие вопросы:

| ID | Вопрос | Приоритет |
| --- | --- | --- |
| IMP-001 | Проверка фактических PK типов `leads.id` и `bot_users.id` | Critical |
| IMP-002 | Финальные Queue retry/consumer settings | High |
| IMP-003 | Точное расписание Scheduled Triggers | Medium |
| IMP-004 | Автоматизация регистрации Telegram webhooks | High |
| IMP-005 | Политика резервного копирования R2 | High |
| IMP-006 | Пороговые значения Queue backlog и Outbox age | Medium |
| IMP-007 | Время ежедневной сводки | Low |
| IMP-008 | AI provider и границы передаваемых данных | Medium |
| IMP-009 | Альтернативный канал критических уведомлений | Medium |
| IMP-010 | Частота restore tests | High |

---

**Итог:** IMPLEMENTATION-001 v0.2 задаёт последовательность реализации MVP от подготовки репозитория до production deployment. Документ не утверждает, что инфраструктура уже создана или что все открытые технические вопросы решены.


## 23. Consolidated Supplement — Details Retained from v0.1

This section preserves implementation details from the earlier v0.1 plan that complement v0.2 and are not repeated in the main roadmap. The main phases and acceptance criteria in sections 1–22 remain authoritative.

### 23.1. Explicit MVP exclusions

The following are outside the MVP scope:

- Full CRM.
- Separate Admin Panel.
- Mobile application.
- Automated sending of commercial proposals to clients.
- AI Agent changing production infrastructure automatically.
- Automated payments and commission calculations.
- ERP integration.
- Full AI sales assistant.

### 23.2. AI Incident Agent — provider candidate and integration boundary

**Primary candidate from the earlier plan:** Cursor Automations / Cloud Agents. This remains a candidate, not a locked provider decision; capabilities, access model, cost controls, and security must be validated before implementation.

Expected integration options include monitoring webhooks, limited diagnostic context, MCP or Worker API for Telegram notification delivery, and read-only technical data access. AI execution provider must be replaceable without changing Lead Management Bot or its API.

### 23.3. Service identities

The implementation should retain distinct service identities:

- `telegram-incident-service`
- `telegram-leads-service`

Each identity receives only the permissions required for its operations. Authorization must be server-side and every operation must carry an actor identity.

### 23.4. Internal Lead Management API

The earlier plan defined the following internal endpoints as a concrete implementation baseline, subject to API-001 alignment:

```text
GET    /api/v1/internal/leads
GET    /api/v1/internal/leads/:id
PATCH  /api/v1/internal/leads/:id/status
POST   /api/v1/internal/leads/:id/notes
POST   /api/v1/internal/leads/:id/reminders
GET    /api/v1/internal/reminders
GET    /api/v1/internal/stats
```

Implementation expectations:

- cursor pagination;
- status filters and search;
- lead details;
- status transition service;
- notes and reminders services;
- audit service;
- integration tests.

Changing lead status and recording its `lead_events` entry must occur in one PostgreSQL transaction. Creating a note and its corresponding audit event must also be atomic.

### 23.5. Incident Management API

The earlier plan specified the following endpoints as a baseline, subject to API-001 alignment:

```text
GET  /api/v1/internal/operations/status
GET  /api/v1/internal/operations/incidents
GET  /api/v1/internal/operations/incidents/:id
POST /api/v1/internal/operations/incidents/:id/acknowledge
POST /api/v1/internal/operations/incidents/:id/resolve
```

Incident lifecycle rules:

- repeated events should be grouped/deduplicated;
- incident history is retained;
- acknowledge does not resolve the incident;
- resolve requires evidence/confirmation of recovery;
- critical incidents notify the owner;
- recovery notification is considered where applicable.

### 23.6. Reminder processor cadence

Retained schedule baseline from v0.1:

- Reminder Processor — every minute.
- Lead Follow-up — every five minutes; create follow-ups at two and twenty-four hours where the lead remains eligible.
- Daily Summary — once per day.
- Outbox Reconciliation — every five minutes.

Exact timezone and daily summary time remain open decisions. Before sending, jobs must re-check lead state. A reminder is marked `SENT` only after Telegram API confirms delivery.

### 23.7. Failure simulation checklist

Additional failure scenarios retained from v0.1:

- Telegram API unavailable;
- Neon unavailable;
- Queue Consumer failure;
- duplicate Telegram updates;
- expired callback;
- failed database migration;
- AI Agent unavailable;
- notification retry;
- DLQ recovery.

These scenarios supplement the testing matrix in section 15 and must be included in staging/recovery validation where applicable.

### 23.8. Earlier schema examples — non-authoritative

The v0.1 document included example SQL definitions for `bot_users`, `telegram_updates`, `lead_notes`, `lead_reminders`, `bot_notification_deliveries`, `incidents`, and `incident_events`. Those examples are **not copied as authoritative schema** here because IMPLEMENTATION-001 v0.2 explicitly requires checking the real PK/FK types and aligning all changes with DATA-001. DATA-001 remains the source of truth for database schema and migration definitions.

---

## 24. Document Status

**Version:** 0.3  
**Status:** Draft — Consolidated  
**Owner:** Project Owner  
**Approval:** Pending

IMPLEMENTATION-001 v0.3 consolidates the v0.2 system implementation plan with complementary scope, API baselines, service identities, AI provider candidate, reminder cadence, and failure-simulation details from v0.1. It is a target plan and does not claim that the infrastructure or features are already implemented or tested.

---

**End of Document — IMPLEMENTATION-001 v0.3 Consolidated**
