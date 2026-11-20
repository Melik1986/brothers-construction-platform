# DATA-001 — Data Model & Database Specification

**Проект:** «Братья» / եղբայրներ
**Документ:** DATA-001
**Версия:** 0.2
**Статус:** Draft — Technical Consistency Update
**Дата:** 19.11.2026
**База данных:** Neon PostgreSQL
**ORM:** Drizzle ORM
**Migration Tool:** Drizzle Kit
**Источник истины:** PostgreSQL

---

## 1. Назначение документа

DATA-001 определяет структуру базы данных, отношения между сущностями, ограничения целостности, индексы и правила обработки данных для сайта строительной компании «Братья».

Документ охватывает:

- Приём и хранение клиентских заявок.
- Сохранение маркетинговой атрибуции.
- Историю событий по заявкам.
- Асинхронную доставку уведомлений.
- Идемпотентность API-запросов.
- Управление заявками через Telegram Lead Management Bot.
- Напоминания и автоматические follow-up.
- Учёт пользователей и разрешений Telegram-ботов.
- Инциденты и события Operations Bot.

PostgreSQL является единственным источником истины для бизнес-данных. Telegram, Cloudflare Queues и внешние провайдеры не заменяют базу данных.

## 2. Технологические решения

| Компонент | Решение |
| --- | --- |
| Database | Neon PostgreSQL |
| ORM | Drizzle ORM |
| Migrations | Drizzle Kit |
| SQL validation | Squawk |
| Runtime | Cloudflare Workers |
| Async transport | Cloudflare Queues |
| Notifications | Resend, Telegram Bot API |
| Schema validation | Zod |
| Secrets | Cloudflare Worker Secrets |

### 2.1. Правила работы с базой данных

1. Все изменения бизнес-состояния сохраняются в PostgreSQL.
2. Внешние API не вызываются внутри DB-транзакций.
3. Асинхронные события фиксируются через Transactional Outbox.
4. Cloudflare Queues используются для доставки и обработки событий, но не являются источником истины.
5. Повторная обработка сообщений должна быть безопасной.
6. Изменения схемы выполняются только через Drizzle Kit migrations.
7. Миграции не запускаются при старте Worker.
8. Все SQL-изменения должны проходить проверку на совместимость и безопасность.

## 3. Архитектура данных

Основные логические группы:

**Public Lead Domain**

- leads
- lead_attribution
- lead_events
- idempotency_keys
- notification_outbox

**Lead Operations Domain**

- bot_users
- bot_user_permissions
- telegram_updates
- lead_notes
- lead_reminders
- bot_notification_deliveries

**Incident Operations Domain**

- incidents
- incident_events

### 3.1. Логические связи

- Один Lead может иметь одну запись атрибуции.
- Один Lead может иметь множество событий.
- Один Lead может иметь множество заметок.
- Один Lead может иметь множество напоминаний.
- Один Lead может инициировать множество уведомлений.
- Telegram-пользователь может иметь несколько разрешений.
- Один Incident может иметь множество Incident Events.
- Telegram Update должен обрабатываться идемпотентно.
- Уведомления должны иметь собственный статус доставки и историю попыток.

## 4. Сущности и спецификация таблиц

### 4.1. leads

Основная таблица клиентских заявок.

| Поле | Назначение |
| --- | --- |
| id | Primary Key; точный тип должен быть подтверждён по текущей схеме |
| name | Имя клиента |
| phone | Контактный телефон |
| email | Email, nullable |
| service | Запрошенная услуга |
| preferred_contact | Предпочтительный способ связи |
| message | Сообщение клиента, nullable |
| locale | Язык заявки: hy, ru, en |
| status | Текущий статус заявки |
| privacy_consent | Согласие на обработку данных |
| created_at | Дата создания |
| updated_at | Дата последнего изменения |

Допустимые статусы:

- NEW
- CONTACTED
- QUALIFIED
- ESTIMATE_SENT
- WON
- LOST
- SPAM

Правила:

- При создании заявки статус устанавливается в `NEW`.
- Изменение статуса выполняется через Internal API.
- Каждое изменение статуса сопровождается записью в `lead_events`.
- Удаление заявки не должно использоваться как обычный способ управления её состоянием.
- Поля контактов не должны попадать в технические логи и AI-диагностику.

**Критическое замечание:** тип `leads.id` в предыдущей реализации не подтверждён. До подготовки миграций необходимо открыть фактический DDL или Drizzle schema и установить тип PK. Все внешние ключи должны использовать тот же тип. В этой редакции тип не фиксируется как UUID.

### 4.2. lead_attribution

Хранит минимальную маркетинговую атрибуцию на момент отправки формы.

| Поле | Назначение |
| --- | --- |
| id | Primary Key |
| lead_id | FK → leads.id |
| landing_page | Страница отправки |
| referrer | Referrer |
| utm_source | Источник |
| utm_medium | Канал |
| utm_campaign | Кампания |
| utm_content | Содержание кампании |
| utm_term | Ключевой термин |
| locale | Язык |
| created_at | Дата записи |

Ограничения:

- Одна запись атрибуции на одну заявку.
- `lead_id` — UNIQUE.
- UTM-поля могут быть NULL.
- Отсутствие UTM не блокирует отправку формы.
- Атрибуция считается недоверенным пользовательским вводом.
- Данные фиксируются при отправке формы.
- Cross-session tracking не применяется.
- First Touch / Last Touch attribution не реализуется.
- Fingerprinting не используется.

Переходы по WhatsApp и Telegram не считаются подтверждёнными лидами.

### 4.3. lead_events

Аудит-история действий по заявкам.

| Поле | Назначение |
| --- | --- |
| id | Primary Key |
| lead_id | FK → leads.id |
| event_type | Тип события |
| old_value | Предыдущее значение, nullable |
| new_value | Новое значение, nullable |
| actor_type | Источник действия |
| actor_id | Идентификатор инициатора, nullable |
| metadata | JSONB с дополнительными данными |
| created_at | Дата события |

Возможные `event_type`:

- LEAD_CREATED
- STATUS_CHANGED
- NOTE_ADDED
- REMINDER_CREATED
- REMINDER_COMPLETED
- REMINDER_CANCELLED
- CONTACT_UPDATED
- LEAD_MARKED_SPAM

Возможные `actor_type`:

- SYSTEM
- OWNER
- TELEGRAM_BOT
- AUTOMATION

Правила:

- События добавляются, а не перезаписываются.
- Изменение статуса и создание соответствующего события выполняются в одной транзакции.
- История должна позволять восстановить последовательность операционных действий.
- В `metadata` запрещено сохранять секреты, токены и ненужные персональные данные.

### 4.4. notification_outbox

Transactional Outbox для надёжной передачи событий в асинхронную обработку.

| Поле | Назначение |
| --- | --- |
| id | Primary Key |
| event_type | Тип события |
| aggregate_type | Тип бизнес-сущности |
| aggregate_id | ID сущности |
| payload | JSONB |
| status | Состояние обработки |
| attempt_count | Количество попыток |
| available_at | Время следующей обработки |
| processed_at | Время завершения |
| last_error | Последняя ошибка |
| created_at | Дата создания |
| updated_at | Дата изменения |

Статусы:

- PENDING
- PROCESSING
- PROCESSED
- FAILED

Правила:

1. Outbox-запись создаётся в той же DB-транзакции, что и бизнес-изменение.
2. Dispatcher выбирает доступные записи и передаёт события в Cloudflare Queues.
3. Успешная передача фиксируется в Outbox.
4. Consumer обрабатывает сообщения идемпотентно.
5. Ошибки доставки и обработки должны быть наблюдаемыми.
6. Для окончательно неуспешных сообщений применяется предусмотренный механизм DLQ и reconciliation.

**Важно:** транзакция PostgreSQL и публикация сообщения в Cloudflare Queues не являются общей атомарной транзакцией. Поэтому Outbox и reconciliation обязательны.

### 4.5. idempotency_keys

Предотвращает повторное создание заявок при повторной отправке одного API-запроса.

| Поле | Назначение |
| --- | --- |
| id | Primary Key |
| key | Idempotency Key |
| request_hash | Хеш нормализованного запроса |
| response_status | HTTP status ответа |
| response_body | Сохранённый результат |
| created_at | Дата создания |
| expires_at | Срок хранения |

Правила:

- Ключ уникален в рамках определённого контекста API.
- Повтор с тем же ключом и тем же содержимым возвращает ранее сохранённый результат.
- Повтор с тем же ключом, но другим содержимым должен отклоняться.
- Срок хранения должен быть определён политикой API.
- Необходимо исключить сохранение лишних персональных данных в `response_body`.

### 4.6. bot_users

Реестр разрешённых пользователей Telegram-ботов.

| Поле | Назначение |
| --- | --- |
| id | Primary Key |
| telegram_user_id | Telegram User ID |
| is_active | Активность пользователя |
| created_at | Дата создания |
| updated_at | Дата изменения |

Ограничения:

- `telegram_user_id` уникален.
- Неизвестный Telegram User ID не получает доступа к операциям.
- Деактивация пользователя блокирует дальнейшие команды.

На MVP предполагается один владелец с ролью OWNER. Дополнительные пользователи и роли не входят в обязательный MVP.

### 4.7. bot_user_permissions

Разрешения пользователя на отдельные операции и боты.

| Поле | Назначение |
| --- | --- |
| id | Primary Key |
| bot_user_id | FK → bot_users.id |
| bot_scope | LEADS или INCIDENTS |
| permission | Код разрешения |
| created_at | Дата создания |

Ограничение уникальности: комбинация `bot_user_id + bot_scope + permission`.

Это разделяет доступ к Lead Management Bot и Incident & Operations Bot. Один общий `role` без разграничения областей доступа не считается достаточной моделью авторизации.

### 4.8. telegram_updates

Регистр обработанных Telegram Updates для защиты от повторного исполнения команд.

| Поле | Назначение |
| --- | --- |
| id | Primary Key |
| telegram_update_id | Telegram Update ID |
| bot_scope | LEADS или INCIDENTS |
| status | Состояние обработки |
| processed_at | Время завершения |
| created_at | Дата получения |

Ограничение уникальности: `telegram_update_id + bot_scope`.

Повторная доставка одного Update не должна приводить к повторному выполнению команды.

### 4.9. lead_notes

Заметки владельца по заявке.

| Поле | Назначение |
| --- | --- |
| id | Primary Key |
| lead_id | FK → leads.id |
| author_id | FK → bot_users.id |
| note | Текст заметки |
| created_at | Дата создания |
| updated_at | Дата изменения |

Заметки доступны через защищённый Internal API и Lead Management Bot.

### 4.10. lead_reminders

Напоминания и автоматические follow-up по заявкам.

| Поле | Назначение |
| --- | --- |
| id | Primary Key |
| lead_id | FK → leads.id |
| reminder_type | Тип напоминания |
| status | Состояние |
| scheduled_at | Запланированное время |
| attempt_count | Количество попыток |
| lease_until | Срок блокировки обработки |
| idempotency_key | Ключ идемпотентности |
| sent_at | Время отправки |
| completed_at | Время завершения |
| last_error | Последняя ошибка |
| created_at | Дата создания |
| updated_at | Дата изменения |

Статусы:

- PENDING
- PROCESSING
- SENT
- COMPLETED
- CANCELLED
- FAILED

Типы напоминаний:

- NEW_LEAD_2H
- NEW_LEAD_24H
- DAILY_SUMMARY
- MANUAL_FOLLOW_UP

Правила:

1. Напоминания создаются с определённым `scheduled_at`.
2. Scheduler атомарно захватывает доступные записи.
3. При захвате устанавливается `PROCESSING` и `lease_until`.
4. Если Worker аварийно завершился, просроченная блокировка позволяет повторно обработать запись.
5. Повторная отправка предотвращается через idempotency key и журнал доставки.
6. При изменении статуса заявки автоматические напоминания должны проверяться на актуальность.
7. Для `WON`, `LOST` и `SPAM` ненужные автоматические напоминания отменяются.
8. Обновление состояния напоминания выполняется транзакционно.

**Concurrency requirement:** два Scheduler/Worker не должны одновременно отправлять одно и то же напоминание. Для этого требуется атомарное получение задания в PostgreSQL, например через `FOR UPDATE SKIP LOCKED` либо эквивалентный механизм.

### 4.11. bot_notification_deliveries

История отправки сообщений ботами.

| Поле | Назначение |
| --- | --- |
| id | Primary Key |
| notification_type | Тип уведомления |
| recipient_id | Идентификатор получателя |
| aggregate_type | Тип сущности |
| aggregate_id | ID сущности |
| idempotency_key | Ключ идемпотентности |
| status | Состояние доставки |
| attempt_count | Количество попыток |
| sent_at | Время успешной отправки |
| last_error | Последняя ошибка |
| created_at | Дата создания |

Таблица позволяет отслеживать отправки, ошибки и повторы уведомлений.

### 4.12. incidents

Регистр инцидентов инфраструктуры и приложения.

| Поле | Назначение |
| --- | --- |
| id | Primary Key |
| source | Источник инцидента |
| external_id | Внешний ID, nullable |
| fingerprint | Нормализованный отпечаток |
| environment | Окружение |
| severity | Критичность |
| status | Состояние |
| title | Краткое описание |
| summary | Безопасное диагностическое описание |
| first_seen_at | Первое обнаружение |
| last_seen_at | Последнее обнаружение |
| acknowledged_at | Время подтверждения |
| resolved_at | Время разрешения |
| created_at | Дата создания |
| updated_at | Дата изменения |

Статусы:

- OPEN
- ACKNOWLEDGED
- RESOLVED
- IGNORED

Уровни критичности:

- INFO
- WARNING
- ERROR
- CRITICAL

Требования:

- Дубликаты инцидентов должны объединяться.
- При наличии `external_id` используется уникальность по источнику, external ID и окружению.
- При отсутствии external ID используется нормализованный fingerprint и окружение.
- Прямое критическое уведомление не должно зависеть от доступности AI Agent.

### 4.13. incident_events

История событий инцидента.

| Поле | Назначение |
| --- | --- |
| id | Primary Key |
| incident_id | FK → incidents.id |
| event_type | Тип события |
| actor_type | SYSTEM, OWNER, AI_AGENT |
| actor_id | Идентификатор инициатора, nullable |
| metadata | JSONB |
| created_at | Дата события |

Примеры событий:

- INCIDENT_CREATED
- INCIDENT_UPDATED
- INCIDENT_ACKNOWLEDGED
- INCIDENT_RESOLVED
- AI_ANALYSIS_COMPLETED
- ALERT_SENT
- ALERT_FAILED

AI Agent может создавать диагностические результаты, но не получает права самостоятельно выполнять разрушительные production-операции.

## 5. Статусы и бизнес-правила

### 5.1. Lead lifecycle

`NEW → CONTACTED → QUALIFIED → ESTIMATE_SENT → WON / LOST`

`SPAM` — отдельное состояние для нежелательных заявок.

Переходы должны контролироваться на уровне бизнес-логики API. Изменения сопровождаются `lead_events`.

### 5.2. Reminder lifecycle

`PENDING → PROCESSING → SENT → COMPLETED`

Дополнительные переходы:

- PENDING → CANCELLED
- PROCESSING → PENDING при истечении lease и возможности повтора
- PROCESSING → FAILED при исчерпании допустимых попыток

Повторная обработка должна учитывать `attempt_count`, `lease_until` и idempotency key.

### 5.3. Notification lifecycle

`PENDING → PROCESSING → PROCESSED`

При ошибках обработки применяются retry и DLQ согласно OPS-001.

## 6. Индексы и ограничения

Предусмотреть следующие индексы после проверки фактических запросов:

- `leads(status, created_at)`
- `leads(created_at)`
- `lead_events(lead_id, created_at)`
- `lead_notes(lead_id, created_at)`
- `lead_reminders(status, scheduled_at)`
- `lead_reminders(lease_until)`
- `notification_outbox(status, available_at)`
- `incidents(status, severity, last_seen_at)`
- `incident_events(incident_id, created_at)`

Уникальные ограничения:

- `lead_attribution(lead_id)`
- `bot_users(telegram_user_id)`
- `bot_user_permissions(bot_user_id, bot_scope, permission)`
- `telegram_updates(telegram_update_id, bot_scope)`
- `bot_notification_deliveries(idempotency_key)`
- `incidents(source, external_id, environment)` — для записей с external ID.
- `incidents(fingerprint, environment)` — для записей без external ID.

Для nullable `external_id` необходимо использовать частичный уникальный индекс или эквивалентное условие, чтобы избежать нежелательного поведения PostgreSQL с NULL.

Внешние ключи должны соответствовать фактическому типу первичного ключа родительской таблицы.

## 7. Транзакционные границы

### 7.1. Создание заявки

В одной DB-транзакции:

1. Проверить idempotency key.
2. Создать `leads`.
3. Создать `lead_attribution`.
4. Создать `lead_events` с `LEAD_CREATED`.
5. Создать `notification_outbox` для дальнейшего уведомления.
6. Сохранить результат идемпотентного запроса.

После commit асинхронный dispatcher передаёт событие в очередь.

### 7.2. Изменение статуса заявки

В одной транзакции:

1. Проверить полномочия инициатора.
2. Получить текущую заявку.
3. Проверить допустимость перехода.
4. Изменить статус и `updated_at`.
5. Создать `lead_events`.
6. При необходимости обновить/отменить напоминания.
7. Зафиксировать транзакцию.

### 7.3. Создание заметки

В одной транзакции:

1. Проверить пользователя и разрешения.
2. Создать `lead_notes`.
3. Создать `lead_events` с `NOTE_ADDED`.

### 7.4. Захват напоминания

1. Найти доступные записи `PENDING` или записи с истёкшим lease.
2. Атомарно захватить запись.
3. Установить `PROCESSING`.
4. Увеличить `attempt_count`.
5. Установить `lease_until`.
6. Зафиксировать транзакцию.
7. Отправить уведомление вне DB-транзакции.
8. Зафиксировать результат доставки.

## 8. Retention и защита данных

- Доступ к DB осуществляется только через серверные компоненты.
- Клиентский frontend не подключается к Neon напрямую.
- Telegram Bot API не получает прямого доступа к базе.
- Все операции проходят через защищённый Internal API.
- Telegram User ID проверяется по allowlist.
- Webhook secret подтверждает подлинность запроса Telegram, но не заменяет проверку прав пользователя.
- Необходимо минимизировать хранение персональных данных.
- Логи не должны содержать токены, пароли, полные контактные данные и чувствительное содержимое заявок.
- Срок хранения заявок и связанных персональных данных должен быть утверждён в отдельной политике retention/privacy.
- Удаление и анонимизация должны учитывать требования применимого законодательства и необходимость сохранения допустимой бизнес-истории.

## 9. Drizzle ORM и миграции

Структура должна быть представлена в Drizzle schema и изменяться только через миграции.

Рекомендуемая организация:

```text
src/
  db/
    schema/
      leads.ts
      attribution.ts
      lead-events.ts
      notifications.ts
      idempotency.ts
      bots.ts
      reminders.ts
      incidents.ts
      index.ts
    client.ts
    migrations/
```

Правила:

- Drizzle schema — типизированное представление модели.
- Drizzle Kit — генерация и применение миграций.
- SQL migrations хранятся в Git.
- Применённые миграции не редактируются задним числом.
- Squawk используется для проверки SQL.
- Миграции выполняются отдельным CI/CD этапом.
- Для миграций используется ограниченная учётная запись.
- Worker не выполняет миграции при старте.

## 10. Проверки и тестирование

Обязательные сценарии:

**Leads**

- Создание заявки.
- Валидация обязательных полей.
- Сохранение локали.
- Сохранение атрибуции.
- Повторная отправка с тем же idempotency key.
- Конфликт idempotency key с другим содержимым.

**Events**

- Запись события при создании заявки.
- Запись события при изменении статуса.
- Запись заметки и соответствующего события.
- Сохранение корректной последовательности событий.

**Outbox**

- Создание Outbox-записи в той же транзакции.
- Повторная публикация.
- Восстановление после сбоя между DB commit и Queue publish.
- Идемпотентность Consumer.
- Reconciliation.

**Reminders**

- Захват задания одним Worker.
- Защита от конкурентной обработки.
- Истечение lease.
- Повторная попытка.
- Отмена после изменения статуса заявки.
- Защита от повторной отправки.

**Telegram**

- Проверка webhook secret.
- Проверка Telegram User ID.
- Проверка permissions.
- Повторная доставка одного Update.
- Запрет доступа неизвестным пользователям.

**Incidents**

- Создание нового инцидента.
- Объединение дубликатов.
- Отсутствие external ID.
- Прямое критическое уведомление без AI Agent.
- Запись acknowledge и resolve.

## 11. Критерии приёмки

DATA-001 считается реализованным, когда:

- Все таблицы согласованы с Drizzle schema.
- Типы PK/FK проверены по фактической схеме.
- Все отношения и ограничения целостности реализованы.
- Миграции проходят на staging.
- Создание заявки и Outbox выполняются атомарно.
- Идемпотентность API проверена.
- История событий сохраняется корректно.
- Telegram Updates защищены от повторной обработки.
- Доступ к ботам разграничен.
- Напоминания защищены от конкурентной обработки.
- Incident deduplication работает.
- Тесты критических транзакционных сценариев проходят.
- SQL migrations проходят Squawk.
- Не существует прямого доступа frontend к PostgreSQL.

## 12. Открытые вопросы и обязательные проверки

| ID | Вопрос | Статус |
| --- | --- | --- |
| DATA-OD-001 | Какой фактический тип `leads.id` в текущей схеме? | Must verify |
| DATA-OD-002 | Какой фактический тип `bot_users.id` и остальных PK? | Must verify |
| DATA-OD-003 | Какой формат `idempotency_keys.key` и область уникальности? | To finalize |
| DATA-OD-004 | Какие сроки retention для заявок, событий и логов? | To approve |
| DATA-OD-005 | Нужны ли дополнительные Telegram-пользователи кроме OWNER в MVP? | To confirm |
| DATA-OD-006 | Какой формат `payload` Outbox и версия event schema? | To specify |
| DATA-OD-007 | Какой фактический Neon connection/driver будет использоваться в Worker? | Compatibility test |

**Запрещено считать DATA-001 полностью готовым к production migrations, пока DATA-OD-001 и DATA-OD-002 не проверены по реальной схеме.**

## 13. Связанные документы

- BRD-001 — Business Requirements.
- SRS-001 — Software Requirements Specification.
- SAD-001 — System Architecture.
- API-001 — API Specification.
- ADR-001 — Architecture Decision Records.
- OPS-001 — Operations & Monitoring.
- TECH-001 — Telegram Bots Technical Architecture.
- DEPLOY-001 — Deployment & CI/CD.
- TEST-001 — Testing Strategy.
- IMPLEMENTATION-001 — Implementation Plan.

---

**История изменений**

| Версия | Дата | Изменения |
| --- | --- | --- |
| 0.1 | 2026-11-18 | Первичная модель заявок, атрибуции, событий, Outbox и идемпотентности |
| 0.2 | 2026-11-19 | Добавлены сущности Telegram-ботов, permissions, updates, notes, reminders, delivery tracking и incidents; уточнены транзакционные границы, конкурентная обработка, индексы, ограничения и открытые проверки типов PK/FK |
