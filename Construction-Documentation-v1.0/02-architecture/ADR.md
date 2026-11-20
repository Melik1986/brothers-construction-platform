# ADR-001 — Architecture Decision Records
**Версия:** 1.1 | **Статус:** Approved по зафиксированным решениям

1. **Transactional Outbox + Cloudflare Queues.** Бизнес-запись и outbox создаются одной DB-транзакцией; внешняя доставка после commit. Нужны idempotent consumers, retry/DLQ, reconciliation.
2. **Ручное управление лидами без CRM.** MVP использует Lead Management Bot: list/detail/status/notes/reminders/stats.
3. **Минимальная атрибуция.** Snapshot только при отправке формы; без cross-session tracking, First/Last Touch и fingerprinting.
4. **Application Services внутри Worker.** Прямые вызовы модулей, без HTTP self-calls.
5. **Unified Correlation ID.** Сквозная трассировка; eventId и idempotencyKey имеют отдельные назначения.
6. **PostgreSQL driver compatibility.** Совместимость драйвера с Neon и Cloudflare Workers была подтверждена на этапе проектирования; версии и runtime configuration фиксируются и тестируются при реализации.
7. **Independent Direct Alert / AI Analysis.** Alert работает без AI; AI — дополнительный контур без DB access, secrets и destructive production actions.
8. **Два Telegram-бота.** Раздельные credentials, webhook secrets, endpoints и permissions.

Последствия: PostgreSQL source of truth; provider calls вне транзакций; асинхронная доставка рассматривается как at-least-once; архитектурные изменения фиксировать новыми ADR.
