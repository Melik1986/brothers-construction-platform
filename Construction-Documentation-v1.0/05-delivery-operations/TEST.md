# TEST-001 — Testing & Quality Assurance Strategy

**Project:** «Братья» / «եղբայրներ»
**Document ID:** TEST-001
**Version:** 0.1
**Status:** Draft
**Document Type:** Testing & Quality Assurance Strategy
**Related Documents:** BRD-001, SRS-001, SAD-001, UX-001, API-001 / DATA-001, ADR-001
**Architecture:** Astro + TypeScript + Cloudflare Workers + Neon PostgreSQL + Cloudflare R2

---

## 1. Purpose

TEST-001 defines the testing strategy, quality assurance processes, test coverage requirements, and release quality gates for the «Братья» website and its supporting backend infrastructure.

The objective is to ensure that the system:

- Meets the functional requirements defined in SRS-001.
- Preserves the reliability and integrity of customer inquiries.
- Processes notifications asynchronously without losing submitted leads.
- Protects public API endpoints against abuse and invalid input.
- Provides consistent multilingual and responsive user experiences.
- Meets the accessibility and performance requirements.
- Can be deployed and operated reliably on Cloudflare infrastructure.

This document establishes the testing approach for MVP and identifies additional quality controls that may be introduced in later development phases.

## 2. Testing Principles

The project follows these principles:

1. **Risk-based testing:** prioritize customer inquiry processing, data integrity, security, and availability.
2. **Shift-left testing:** validate requirements, types, schemas, and business logic as early as possible.
3. **Automation first:** automate repeatable checks and regression tests.
4. **Production-like environments:** test infrastructure-dependent functionality in environments that reproduce production configuration.
5. **Data integrity:** never treat successful notification delivery as the source of truth for a submitted lead.
6. **Fail-safe behavior:** failures in external notification providers must not result in lost inquiries.
7. **Accessibility by design:** include accessibility checks throughout implementation rather than only before release.
8. **Traceability:** connect test cases to requirements, API contracts, architecture decisions, and acceptance criteria.

## 3. Testing Scope

### 3.1 In Scope

The MVP testing scope includes:

- Astro page generation and routing.
- Multilingual content and localized navigation.
- Responsive layouts and interactive components.
- Contact and project inquiry forms.
- Client-side and server-side validation.
- Cloudflare Turnstile verification.
- API rate limiting and idempotency.
- Lead persistence in Neon PostgreSQL.
- Lead attribution.
- Transactional Outbox processing.
- Cloudflare Queues producer and consumer workflows.
- Notification delivery through Resend and Telegram Bot API.
- Outbox recovery and reconciliation.
- Protected manual lead management.
- R2 media delivery and access configuration.
- SEO technical requirements.
- Accessibility and performance.
- CI/CD quality gates and deployment smoke tests.

### 3.2 Out of Scope

The following capabilities are not part of MVP testing because they are not included in the current MVP scope:

- Full CRM workflows.
- Public administrative dashboard.
- Customer accounts and authentication.
- Online payments.
- Automated commission calculations.
- Deal and payment management.
- Persistent cross-session attribution.
- First Touch / Last Touch attribution.
- Automated marketing campaigns.
- Advanced personalization.

These capabilities require separate requirements and test specifications if introduced in future releases.

## 4. Test Strategy and Levels

### 4.1 Unit Testing

**Objective:** Verify isolated functions and business logic without requiring external infrastructure.

Unit tests should cover:

- Zod validation schemas.
- Phone and email normalization.
- Required and optional form fields.
- Locale validation.
- Attribution parsing and normalization.
- UTM parameter handling.
- Idempotency key validation.
- Rate-limit decision logic.
- Lead status transition rules.
- Outbox state transitions.
- Retry and backoff calculations.
- Notification message formatting.
- Utility functions and reusable UI logic.

**Expected characteristics:**

- Tests are deterministic.
- External services are mocked.
- Tests do not depend on production credentials.
- Each test verifies one clearly defined behavior.
- Edge cases and invalid input are included.

**Tools — Proposed:**

- Vitest.
- Testing Library for React components.
- Zod for schema validation.

### 4.2 Component Testing

**Objective:** Verify individual UI components and their interaction behavior.

Components to test:

- Header and navigation.
- Mobile navigation.
- Language selector.
- Service cards.
- Project cards.
- Gallery and media components.
- Contact form.
- Form validation messages.
- Loading and submission states.
- Success and error states.
- CTA components.
- Interactive React Islands.

Required checks:

- Correct rendering with valid props.
- Correct behavior with missing optional content.
- Keyboard interaction.
- Accessible labels and roles.
- Responsive behavior.
- Correct localized text.
- No unexpected layout or runtime errors.

**Tools — Proposed:**

- Vitest.
- React Testing Library.
- jest-axe or axe-core integration.

### 4.3 Integration Testing

**Objective:** Verify interactions between application modules and infrastructure services.

Integration tests must cover:

- Astro application and Worker API communication.
- Worker API and Neon PostgreSQL.
- Worker API and Turnstile verification.
- Database transaction and Outbox creation.
- Outbox publisher and Cloudflare Queues.
- Queue consumer and Resend.
- Queue consumer and Telegram Bot API.
- Worker API and R2, where applicable.
- Environment-specific configuration and secrets.

Integration tests should use dedicated test resources and isolated test data.

External provider failures must be simulated to verify system behavior without relying exclusively on live services.

### 4.4 API Testing

**Objective:** Validate the API contract and its behavior under normal, invalid, and abusive requests.

#### POST /api/v1/leads

Test scenarios:

- Valid lead submission.
- Missing required fields.
- Invalid phone number.
- Invalid email format.
- Unsupported locale.
- Invalid preferred contact method.
- Missing privacy consent.
- Invalid Turnstile token.
- Expired or rejected Turnstile verification.
- Missing or malformed attribution.
- Excessively large request body.
- Malformed JSON.
- Rate limit exceeded.
- Duplicate idempotency key.
- Repeated request after a successful submission.
- Database unavailable.
- Notification provider unavailable.
- Internal processing error.

Expected behavior:

- Invalid requests are rejected before lead persistence.
- Valid requests are persisted reliably.
- A successful database commit is not invalidated by a notification provider outage.
- Duplicate submissions are handled according to the idempotency contract.
- Internal errors do not expose secrets, SQL details, stack traces, or sensitive infrastructure information.
- API responses follow the documented response contract in API-001.

#### GET /api/v1/health

Test scenarios:

- API process is available.
- Required configuration is present.
- Database connectivity is checked according to the endpoint's defined health policy.
- Internal dependency details are not exposed publicly.

The health endpoint must not disclose connection strings, credentials, database identifiers, or internal infrastructure configuration.

**Tools — Proposed:**

- Vitest.
- Cloudflare Workers test utilities.
- HTTP integration tests using a dedicated test environment.

## 5. Database Testing

**Objective:** Verify schema correctness, data integrity, constraints, transactions, and migration safety.

Database tests must cover:

### 5.1 Schema and Constraints

- Required columns reject invalid null values.
- Unique constraints work as intended.
- Foreign keys preserve relational integrity.
- Indexes support expected query patterns.
- Enum or status values are validated.
- Timestamps are stored consistently.
- Migrations can be applied to a clean database.

### 5.2 Lead Persistence

Verify that:

- A valid inquiry creates one lead record.
- Attribution is associated with the correct lead.
- Initial lead events are recorded.
- Outbox notification records are created as part of the intended database transaction.
- A failed transaction does not leave partially persisted lead data.
- Sensitive information is not written into unnecessary fields or logs.

### 5.3 Transactional Integrity

A critical test scenario:

1. Submit a valid lead.
2. Begin the database transaction.
3. Create the lead record.
4. Create attribution data.
5. Create the initial lead event.
6. Create the Outbox record.
7. Commit the transaction.
8. Verify that all expected records exist.

If the transaction fails, verify that none of its intended records are partially committed.

### 5.4 Migration Testing

Every schema migration must be tested against:

- A clean database.
- A database containing representative test data.
- The previous supported schema version.

Production migrations must have a documented execution and recovery procedure.

Destructive migrations require explicit review and a verified backup or recovery strategy.

## 6. Asynchronous Processing and Notification Testing

**Critical subsystem — ADR-001-01**

The notification pipeline uses:

**Lead API → Neon PostgreSQL → Transactional Outbox → Cloudflare Queues → Consumer Worker → Resend / Telegram**

PostgreSQL remains the source of truth.

The system must not assume that a PostgreSQL transaction and Queue publication form one atomic transaction.

### 6.1 Outbox Tests

Test scenarios:

- Outbox record is created after successful lead submission.
- Outbox record is not created when the lead transaction fails.
- Pending Outbox records are detected by the publisher.
- Successfully published records are marked appropriately.
- Failed publication does not silently discard the Outbox record.
- Reconciliation detects records that were not published.
- Reconciliation can safely retry publication.

### 6.2 Queue Producer Tests

Verify:

- Correct message structure.
- Required message identifiers.
- Correct lead and notification references.
- Valid message serialization.
- Handling of Queue publication failures.
- Duplicate publication behavior.

### 6.3 Queue Consumer Tests

Verify:

- Valid messages are processed.
- Invalid messages are rejected or routed according to the failure policy.
- Duplicate message delivery does not create duplicate logical notifications.
- Successful delivery updates the appropriate processing state.
- Transient errors trigger retry behavior.
- Permanent errors follow the defined dead-letter handling policy.

The consumer must be idempotent because message delivery may occur more than once.

### 6.4 External Provider Failure Tests

Simulate:

- Resend API unavailable.
- Resend timeout.
- Resend rate limit response.
- Telegram Bot API unavailable.
- Invalid Telegram credentials.
- Telegram rate limit response.
- Network interruption during delivery.

Expected behavior:

- Lead records remain intact.
- Notification failures are recorded.
- Retry policy is applied where appropriate.
- Repeated attempts do not create uncontrolled duplicate notifications.
- Permanent failures remain visible for operational investigation.

### 6.5 Recovery and Reconciliation Tests

A mandatory failure scenario:

1. A lead is successfully committed to PostgreSQL.
2. Its Outbox record is committed.
3. Queue publication fails.
4. The system remains available to accept other valid inquiries.
5. Reconciliation identifies the pending Outbox record.
6. The record is published successfully after recovery.
7. The consumer processes the message.
8. The notification is delivered or the final failure is recorded.

The test must demonstrate that a Queue outage does not cause the loss of a previously committed lead.

## 7. Security Testing

**Objective:** Verify that public endpoints, stored data, and operational access follow the security requirements defined in SAD-001 and API-001.

### 7.1 Input Validation

Test:

- SQL injection attempts.
- Cross-site scripting payloads.
- HTML injection.
- Malformed JSON.
- Unexpected object properties.
- Oversized payloads.
- Invalid content types.
- Invalid Unicode and special characters.

### 7.2 Abuse Prevention

Test:

- Turnstile verification.
- Invalid and expired tokens.
- Rate limiting.
- Repeated submissions.
- Idempotency behavior.
- Automated request bursts.
- Direct API calls bypassing frontend validation.

Frontend validation must never be treated as a security boundary.

### 7.3 Access Control

Verify:

- No public access to internal operational endpoints.
- No direct database access from the browser.
- No unauthorized lead status changes.
- No exposure of administrative credentials.
- No excessive privileges for service credentials.
- Secrets are stored in Cloudflare Worker Secrets or the appropriate protected environment.

### 7.4 Data Exposure

Verify:

- Error messages do not expose stack traces.
- Logs do not contain secrets or full credentials.
- API responses do not expose internal database details.
- Customer contact information is not unnecessarily exposed.
- R2 private objects cannot be accessed without authorization.

### 7.5 Dependency and Configuration Checks

- Dependency vulnerability scanning.
- Secret scanning in the repository.
- Review of environment-specific configuration.
- Validation that production credentials are not used in development or staging.
- Review of Cloudflare token permissions.

**Tools — Proposed:**

- OWASP ZAP for selected dynamic API checks.
- GitHub security features and dependency scanning.
- Secret scanning.
- Manual security review.

## 8. End-to-End Testing

**Objective:** Validate complete user journeys from the visitor's perspective.

E2E tests must cover the principal business journeys.

### 8.1 Homepage Journey

1. Visitor opens the homepage.
2. Main content loads.
3. Visitor navigates between sections.
4. Visitor changes the language.
5. Visitor opens a service or project page.
6. Visitor uses the primary CTA.

### 8.2 Service Discovery Journey

1. Visitor opens the services overview.
2. Visitor selects a service.
3. Visitor reviews service information.
4. Visitor opens a relevant project.
5. Visitor proceeds to the contact form.

### 8.3 Project Discovery Journey

1. Visitor opens the project portfolio.
2. Visitor views project details.
3. Images and media load correctly.
4. Visitor navigates to another project or service.
5. Visitor submits an inquiry.

### 8.4 Lead Submission Journey

1. Visitor opens the contact form.
2. Visitor enters required information.
3. Visitor selects a preferred contact method.
4. Visitor accepts the privacy consent.
5. Turnstile verification is completed.
6. Visitor submits the form.
7. The API validates the request.
8. The lead is stored in PostgreSQL.
9. The Outbox record is created.
10. The visitor receives the appropriate success response.
11. The notification pipeline processes the request.

### 8.5 Error and Recovery Journey

Verify the user experience when:

- Required fields are missing.
- Validation fails.
- Turnstile rejects the request.
- The API is temporarily unavailable.
- The database is unavailable.
- The notification provider is unavailable.

A notification outage must not be presented as a failed lead submission when the lead has already been successfully committed.

**Tools — Proposed:**

- Playwright.
- Browser-based testing in Chromium, Firefox, and WebKit where supported.

## 9. Localization Testing

The website supports:

- Armenian (`hy`)
- Russian (`ru`)
- English (`en`)

Test requirements:

- Every MVP page has the correct localized route.
- Language switching preserves the equivalent page where a translation exists.
- Missing translations follow a defined fallback policy.
- Navigation and CTA labels are localized.
- Form validation and error messages are localized.
- Metadata is localized.
- `hreflang` references are correct.
- Canonical URLs point to the appropriate localized page.
- Armenian and Russian text render correctly.
- Text expansion does not break layouts.
- No untranslated placeholder content appears in production.

Localization tests must include representative long strings and mobile layouts.

## 10. Accessibility Testing

**Target:** WCAG 2.2 Level AA.

Accessibility checks include:

- Semantic HTML.
- Correct heading hierarchy.
- Keyboard navigation.
- Visible focus indicators.
- Form labels and descriptions.
- Accessible validation messages.
- Appropriate alternative text.
- Color contrast.
- Accessible navigation.
- Reduced-motion support.
- Screen-reader compatibility.
- Accessible modal and interactive behavior.
- Focus management after form submission and errors.

Automated accessibility scans are necessary but not sufficient. Critical user journeys must also be reviewed manually using keyboard navigation and screen-reader checks.

**Tools — Proposed:**

- axe-core.
- Playwright accessibility checks.
- Lighthouse.
- Manual keyboard and screen-reader testing.

## 11. Performance Testing

**Objective:** Verify that the website provides fast and stable user experiences on desktop and mobile devices.

### 11.1 Performance Metrics

Target Core Web Vitals:

- LCP: ≤ 2.5 seconds.
- INP: ≤ 200 milliseconds.
- CLS: ≤ 0.1.

Targets should be evaluated using representative mobile and desktop conditions. Field data should be reviewed after sufficient production traffic becomes available.

### 11.2 Performance Checks

- Initial page load.
- JavaScript bundle size.
- CSS delivery.
- Image optimization.
- Responsive image selection.
- Lazy loading.
- Font loading.
- Third-party scripts.
- Animation performance.
- R2 media delivery.
- API response time.
- Static asset caching.

Heavy animations must not compromise the primary content or form usability.

### 11.3 Performance Tools

**Proposed:**

- Lighthouse.
- Chrome DevTools.
- WebPageTest.
- Cloudflare analytics and observability tools.

Lighthouse scores are diagnostic indicators, not substitutes for real-user Core Web Vitals.

## 12. SEO and Technical Quality Testing

SEO testing must verify the technical requirements defined in SEO-001.

Checks include:

- Correct page titles and meta descriptions.
- Canonical URLs.
- `hreflang` implementation.
- XML sitemap generation.
- Robots directives.
- Structured data validity.
- Open Graph metadata.
- Image alternative text.
- Correct heading structure.
- Indexable public pages.
- Correct HTTP status codes.
- No unintended indexing of internal endpoints.
- No broken internal links.
- No broken localized routes.

Structured data must represent information actually present on the corresponding page.

**Tools — Proposed:**

- Google Search Console.
- Lighthouse.
- Schema Markup Validator.
- Rich Results Test.
- Automated link checking.

## 13. Media and Storage Testing

**Objective:** Verify correct delivery and access control for media stored in Cloudflare R2.

Test cases:

- Public media is accessible through the configured media domain.
- Private media cannot be accessed publicly.
- Presigned URLs work within their validity period.
- Expired presigned URLs are rejected.
- Missing media returns an appropriate response.
- Incorrect content types are handled safely.
- Large media files do not block page rendering.
- Responsive images are delivered appropriately.
- Media references remain valid after deployment.

Public media must be read-only from the website visitor's perspective.

## 14. Manual Lead Management Testing

**Reference:** ADR-001-02.

The MVP does not include a public CRM or Admin Panel.

Testing must verify the selected protected operational workflow once that mechanism is defined.

Required checks:

- Only authorized operators can access lead data.
- Unauthorized access is rejected.
- Lead status changes are validated.
- Status changes create corresponding `lead_events`.
- Operational access follows least-privilege principles.
- No public endpoint exposes lead management functionality.
- Manual updates do not corrupt existing attribution or event history.

The exact operational access mechanism remains an implementation decision.

## 15. Attribution Testing

**Reference:** ADR-001-03.

MVP attribution is captured at the time of form submission.

Test cases:

- Valid UTM parameters are captured.
- Missing UTM parameters do not block submission.
- Landing page is recorded correctly.
- Referrer is handled safely.
- Locale is recorded.
- Unsupported or malformed attribution values are normalized or rejected according to the API contract.
- Attribution is linked to the correct lead.
- Attribution values are treated as untrusted input.
- No persistent cross-session tracking is introduced unintentionally.

WhatsApp and Telegram clicks may be recorded as analytics events but must not be treated as confirmed leads.

## 16. Test Environments and Test Data

### 16.1 Environments

**Development**

- Local development environment.
- Mocked or isolated external integrations.
- Dedicated development database.
- No production credentials.

**Staging**

- Production-like Cloudflare configuration.
- Dedicated staging Worker.
- Dedicated staging database.
- Dedicated Queue and consumer configuration.
- Test Resend and Telegram credentials.
- Turnstile test configuration where supported.
- Isolated R2 test assets.

**Production**

- Real customer data.
- Production secrets.
- Production database and Queue.
- Monitoring and operational alerts.
- No destructive automated test execution.

### 16.2 Test Data

Test data must include:

- Valid and invalid contact information.
- All supported locales.
- Different service selections.
- Missing optional fields.
- Representative UTM combinations.
- Duplicate submissions.
- Invalid Turnstile tokens.
- Provider failure scenarios.
- Different lead statuses.
- Boundary cases for payload sizes and validation.

Use synthetic customer information in automated tests. Real customer data must not be copied into development or staging without an approved and documented reason.

## 17. CI/CD Quality Gates

Every pull request must pass the required automated quality checks before merging.

### 17.1 Pull Request Checks

1. Dependency installation.
2. TypeScript type checking.
3. ESLint.
4. Unit tests.
5. Component tests.
6. Production build.
7. Critical integration tests.
8. Dependency and secret scanning.

### 17.2 Pre-Deployment Checks

- Staging deployment succeeds.
- Database migrations pass.
- API smoke tests pass.
- Lead submission test passes.
- Queue processing test passes.
- Notification integration test passes.
- Localization routes are valid.
- Critical E2E tests pass.
- Accessibility checks pass.
- Performance budgets are not exceeded.

### 17.3 Production Deployment Checks

- Production deployment succeeds.
- Health endpoint responds correctly.
- Critical public routes load.
- Lead API is operational.
- Database connectivity is verified.
- Queue consumer is operational.
- Notification processing is monitored.
- No critical errors appear in application logs.

Production smoke tests must not create real customer notifications unless a dedicated, controlled test recipient and test procedure are configured.

## 18. Defect Severity and Release Criteria

### 18.1 Severity Levels

**Critical (P0)**

Examples:

- Customer inquiries are lost.
- Unauthorized access to customer data.
- Production application is unavailable.
- Database corruption or major data integrity failure.

Release decision: deployment blocked or immediate rollback required.

**High (P1)**

Examples:

- Contact form cannot submit valid requests.
- Lead records are not persisted correctly.
- Queue processing permanently fails without recovery.
- Major localized pages are unavailable.
- Critical mobile navigation or user journey is broken.

Release decision: deployment blocked until resolved or formally accepted with a documented mitigation.

**Medium (P2)**

Examples:

- Non-critical layout defects.
- Incorrect optional content.
- Minor accessibility issues.
- Non-critical SEO metadata defects.

Release decision: may be accepted with an owner and planned resolution.

**Low (P3)**

Examples:

- Minor visual inconsistencies.
- Non-blocking text or spacing issues.
- Minor non-functional improvements.

Release decision: may be deferred to a later iteration.

### 18.2 MVP Release Criteria

A release candidate may be approved when:

- All critical and high-severity defects are resolved.
- All critical lead submission tests pass.
- Database integrity tests pass.
- Outbox and Queue recovery tests pass.
- Idempotency tests pass.
- Turnstile and rate-limit checks pass.
- Protected manual access has been verified.
- Required localized routes work.
- No unresolved critical security findings remain.
- Core E2E journeys pass.
- WCAG 2.2 AA checks have been completed and critical accessibility issues are resolved.
- Performance targets have been measured and significant regressions addressed.
- Production deployment and recovery procedures are documented.

## 19. Test Coverage and Traceability

Test coverage should be measured at multiple levels.

Required traceability:

- SRS functional requirements → corresponding test cases.
- API-001 contracts → API test cases.
- DATA-001 entities and constraints → database tests.
- SAD-001 components → integration and infrastructure tests.
- ADR-001 decisions → architecture-specific regression tests.
- UX-001 user journeys → E2E and accessibility tests.
- SEO-001 requirements → technical SEO tests.

Coverage percentages are indicators and must not be used as the only measure of software quality.

Recommended initial targets:

- Critical business logic: at least 90% statement coverage.
- General unit-tested application logic: at least 80% statement coverage.
- Critical lead submission and notification journeys: 100% coverage of defined acceptance scenarios.

These are proposed project targets, not guarantees that all possible defects have been detected.

## 20. Test Automation Structure

Proposed test directory:

```text
tests/
├── unit/
│   ├── validation/
│   ├── attribution/
│   ├── leads/
│   ├── outbox/
│   └── notifications/
│
├── component/
│   ├── navigation/
│   ├── forms/
│   ├── cards/
│   └── interactive/
│
├── integration/
│   ├── api/
│   ├── database/
│   ├── queue/
│   ├── turnstile/
│   └── notifications/
│
├── e2e/
│   ├── homepage/
│   ├── localization/
│   ├── services/
│   ├── projects/
│   └── lead-submission/
│
├── security/
│   ├── validation/
│   ├── rate-limits/
│   └── access-control/
│
└── fixtures/
    ├── leads/
    ├── attribution/
    └── notifications/
```

This is a proposed structure. The final directory layout should follow the repository architecture established during implementation.

## 21. Reporting and Quality Metrics

The development process should track:

- Automated test pass rate.
- Failed CI checks.
- Open defects by severity.
- Critical user journey coverage.
- Lead API success and failure rates.
- Queue processing failures.
- Outbox backlog and reconciliation results.
- Notification delivery failures.
- API response times.
- Core Web Vitals.
- Accessibility findings.
- Production deployment failures.

Operational metrics must not expose customer contact details or sensitive request payloads.

## 22. Risks and Mitigation

| Risk | Impact | Mitigation |
| --- | --- | --- |
| Queue publication failure | Delayed notifications | Outbox reconciliation |
| Duplicate Queue delivery | Duplicate notifications | Idempotent consumer |
| External provider outage | Delayed notification | Retry and failure tracking |
| Invalid or abusive form submissions | API abuse | Zod, Turnstile, rate limiting |
| Database transaction failure | Incomplete lead persistence | Transactional integrity tests |
| Unauthorized operational access | Data exposure | Authentication and least privilege |
| Localization inconsistencies | Poor user experience | Automated route and content checks |
| Heavy media and animations | Poor performance | Performance budgets and profiling |
| Production configuration errors | Deployment failure | Environment validation and smoke tests |

## 23. Acceptance Criteria

TEST-001 is considered implemented when:

- A repeatable unit and component testing setup exists.
- API and database integration tests are operational.
- Critical lead submission scenarios are automated.
- Outbox, Queue, retry, and reconciliation behavior is verified.
- Idempotency behavior is verified.
- Turnstile and rate limiting are tested.
- Security checks are integrated into CI.
- Critical E2E journeys are automated.
- Multilingual routes and form behavior are tested.
- Accessibility and performance checks are documented and repeatable.
- Staging and production smoke-test procedures exist.
- Test results and release decisions can be traced to requirements.

## 24. Open Decisions

The following implementation details remain open and should not be treated as approved technology choices:

**OD-TEST-001-01 — Testing Framework**

Confirm the final combination of Vitest, Testing Library, Playwright, and axe-core after repository initialization.

**OD-TEST-001-02 — Coverage Enforcement**

Confirm whether coverage thresholds will be enforced as hard CI gates or introduced progressively.

**OD-TEST-001-03 — Staging Infrastructure**

Confirm the exact staging topology, including Cloudflare Worker, Neon branch/database, Queues, and R2 resources.

**OD-TEST-001-04 — Notification Test Accounts**

Confirm dedicated test recipients and credentials for Resend and Telegram.

**OD-TEST-001-05 — Operational Access Testing**

Confirm the selected protected manual lead management mechanism before implementing its authorization tests.

**OD-TEST-001-06 — Performance Budgets**

Confirm maximum JavaScript bundle size, image payload budgets, and API latency thresholds after the first measurable build.

**OD-TEST-001-07 — Browser Support Matrix**

Confirm the minimum supported browser versions and device coverage.

---

## 25. Document Status

**Current status:** Draft v0.1

**Approval required from:**

- Project Owner
- Technical Lead

**Next document:** DEPLOY-001 — Deployment & Operations Specification

**Document dependencies:** SRS-001, SAD-001, API-001 / DATA-001, ADR-001.

**Change policy:** Any significant change to the testing architecture, infrastructure, security controls, or release criteria must be reflected in a new document revision.
