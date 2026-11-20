# Decisions & Open Questions

## Зафиксировано

- «Братья» / «եղբայրներ», строительство и ремонт; Ереван, регионы, диаспора; Medium+ / Premium.
- Услуги: ремонт, частные дома, отделка, интерьерный дизайн/3D, фасады, кровля, коммерческие объекты.
- Языки hy/ru/en; Premium Architecture — Editorial Minimalism; Atelier Verrone как визуальный ориентир.
- Контакты WhatsApp/Telegram/phone; email secondary.
- Astro + TS strict; Workers + Static Assets; R2; Neon PostgreSQL; Drizzle; Queues; Resend; Telegram; Turnstile.
- SSG-first; React Islands selectively; SCSS primary.
- One Worker, direct Application Services, no self HTTP calls.
- PostgreSQL source of truth; Transactional Outbox + Queues; two independent bots.
- Direct Alert independent of AI; unified Correlation ID.
- Minimal attribution on form submit; no cross-session tracking/fingerprinting.
- Separate environments/resources/secrets; Trunk-Based Development; protected main; migrations not on Worker startup.

## Open questions

**Business/content:** legal entity/address/contacts, service area, verified guarantees, response SLA, approved case studies/media rights.
**Brand/UX:** final logo, exact palette values, font families/licenses/glyph coverage, slogan, breakpoints, Figma designs.
**Backend/data:** physical PK/FK, full columns/indexes, DTO/pagination/error catalog, internal auth, retention/deletion, incident/outbox states.
**Operations:** queue configuration, cron expressions, daily summary time, R2 backup, alert thresholds, AI provider, fallback alert channel, restore cadence.
**SEO/analytics:** keyword research by locale, GBP eligibility/address, analytics/consent, regional priorities.

## Boundary of current state

Repository, source code, database, actual Drizzle schema and Figma mockups do not exist yet. All schemas here are specifications. Actual Drizzle verification happens after implementation exists.
