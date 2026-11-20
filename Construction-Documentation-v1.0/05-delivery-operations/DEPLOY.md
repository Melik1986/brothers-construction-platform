# DEPLOY-001 — Deployment & Operations Specification

**Project:** «Братья» / «եղբայրներ»  
**Document ID:** DEPLOY-001  
**Version:** 0.4  
**Status:** Draft — Consolidated Specification  
**Document Type:** Deployment, CI/CD & Operations Specification  
**Related Documents:** BRD-001, SRS-001, SAD-001, API-001 / DATA-001, ADR-001, TEST-001  
**Primary Infrastructure:** Cloudflare Workers, Neon PostgreSQL, Cloudflare R2  
**Alternative Deployment Platform:** Vercel  
**CI/CD:** GitHub Actions  
**Application Stack:** Astro, TypeScript, React Islands, SCSS, Drizzle ORM

---

## 1. Purpose

This document defines the deployment architecture, CI/CD pipeline, environment management, release process, infrastructure configuration, operational procedures, and production monitoring for the «Братья» website.

The objectives are:

- Establish a repeatable deployment process.
- Maintain consistency between local development, CI, staging, and production.
- Enforce code quality and security checks before deployment.
- Protect production data and credentials.
- Support safe database migrations.
- Provide controlled deployment and rollback procedures.
- Monitor application health and asynchronous lead processing.
- Minimize operational complexity.
- Preserve the possibility of migrating to an alternative hosting platform.

The deployment strategy follows an infrastructure-light approach based on managed services.

## 2. Deployment Architecture

### 2.1 Primary Architecture

The approved primary deployment architecture is:

**Cloudflare Workers + Static Assets + Neon PostgreSQL + R2 + Queues.**

```text
Developer / AI Agents
        |
        v
GitHub Repository
        |
        v
GitHub Actions CI
        |
        +-- mise / pnpm
        +-- Quality Gates
        +-- PR-Agent Review
        +-- Type Checking
        +-- Tests
        +-- Security Scanning
        +-- Production Build
        |
        v
GitHub Actions CD
        |
        +-- Database Migration
        +-- Cloudflare Deployment
        +-- Smoke Tests
        |
        v
Cloudflare Workers + Static Assets
        |
        +-- Astro Website
        +-- Worker API
        +-- Lead Processing
        +-- Queue Producer / Consumer
        |
        +----------------------+
        |                      |
        v                      v
Neon PostgreSQL          Cloudflare R2
        |
        v
Cloudflare Queues
        |
        v
Notification Consumer
        |
        +-- Resend
        +-- Telegram Bot API
```

### 2.2 Infrastructure Components

**Cloudflare Workers + Static Assets**

Responsibilities:

- Serve the generated Astro website.
- Execute API routes.
- Process incoming lead submissions.
- Integrate with Turnstile.
- Connect to Neon PostgreSQL.
- Publish notification tasks to Cloudflare Queues.

**Neon PostgreSQL**

Responsibilities:

- Store customer inquiries.
- Store lead attribution.
- Maintain lead event history.
- Maintain transactional Outbox records.
- Maintain idempotency records.
- Provide the source of truth for lead processing.

**Cloudflare R2**

Responsibilities:

- Store website images.
- Store project photography.
- Store video-related assets.
- Provide public read-only media delivery.
- Support private media access through presigned URLs.

**Cloudflare Queues**

Responsibilities:

- Process notification tasks asynchronously.
- Decouple lead persistence from external notification delivery.
- Support retry and dead-letter handling.

**GitHub Actions**

Responsibilities:

- Validate code quality.
- Execute automated tests.
- Build the application.
- Run security checks.
- Deploy staging and production.
- Execute post-deployment verification.

**Sentry**

Responsibilities:

- Application error tracking.
- Exception monitoring.
- Stack trace collection.
- Release and environment tracking.
- Performance diagnostics where configured.

**Vercel**

Responsibilities:

- Alternative deployment platform.
- Potential replacement for Cloudflare infrastructure.
- Independent hosting and deployment option.

Vercel is not part of the primary MVP deployment pipeline.

## 3. Deployment Environments

The project uses three environments.

### 3.1 Development

Purpose: local development and feature implementation.

Characteristics:

- Local Astro development server.
- Local Worker development environment where applicable.
- Development Neon branch.
- Development R2 resources.
- Mocked or dedicated notification providers.
- Development-only credentials.
- Debug logging where appropriate.
- Sentry development environment where useful.

Development must never use production credentials by default.

### 3.2 Staging

Purpose: integration testing and release validation.

Characteristics:

- Dedicated Cloudflare Worker deployment.
- Production-like Worker configuration.
- Dedicated Neon staging branch.
- Separate Queue configuration.
- Separate R2 test resources.
- Dedicated Resend and Telegram test recipients.
- Turnstile test configuration where supported.
- Sentry staging environment.

Staging is the required verification environment before production deployment.

### 3.3 Production

Purpose: public website and real customer inquiries.

Characteristics:

- Production Cloudflare Worker.
- Production Neon branch.
- Production Queue and consumer.
- Production R2 media.
- Production notification credentials.
- Production Turnstile configuration.
- Restricted operational access.
- Production monitoring and alerting.
- Sentry production environment.

Production data must not be used for routine automated testing.

### 3.4 Environment Isolation

Each environment must have separate:

- Database connection credentials.
- Worker secrets.
- Queue configuration.
- Notification provider credentials.
- R2 access configuration.
- Turnstile keys.
- Sentry environment configuration.
- API and integration configuration.

Shared infrastructure is allowed only when isolation and access boundaries are explicitly documented and verified.

## 4. Toolchain Management — mise

**Selected tool:** mise.

The project uses one version-controlled toolchain definition to maintain consistent versions across developers and CI.

### 4.1 Responsibilities

mise manages:

- Node.js.
- pnpm.
- Runtime and CLI tool versions.
- Quality Gate utilities.
- Project task execution.
- Local toolchain consistency.

### 4.2 Approved Configuration

File:

```text
mise.toml
```

Requirements:

- Node.js uses the current LTS release selected at repository initialization.
- pnpm version is pinned.
- Quality Gate CLI versions are pinned.
- Local development and CI use the same configuration.
- Tool updates are submitted through reviewed Pull Requests.
- No silent or unreviewed toolchain upgrades.

### 4.3 Package Manager

**Approved:** pnpm.

Configuration:

- Lockfile: `pnpm-lock.yaml`.
- Package manager version: pinned in `package.json`.
- Node.js version: pinned in `mise.toml`.
- CI installation:

```bash
pnpm install --frozen-lockfile
```

npm and Yarn must not be mixed with pnpm in the same project.

### 4.4 Shared Commands

Proposed task interface:

```bash
mise install
mise run check
mise run fix
mise run test
mise run build
mise run deploy:staging
mise run deploy:production
```

The exact task definitions will be established during repository initialization.

## 5. Git Branching Strategy

**Approved model:** simplified Trunk-Based Development with short-lived feature branches.

### 5.1 Permanent Branch

`main`

Responsibilities:

- Single permanent integration branch.
- Protected by GitHub Branch Protection.
- No direct developer or AI-agent pushes.
- Production release source.
- All changes enter through Pull Requests.

### 5.2 Temporary Branches

Naming conventions:

```text
feature/*
fix/*
agent/*
hotfix/*
```

Responsibilities:

- Isolated work for developers and AI agents.
- Independent Pull Requests.
- Short lifetime.
- Deleted after successful merge.

### 5.3 AI Agent Rules

- Each agent works in its own branch.
- Parallel agents should use separate Git worktrees where practical.
- Agents must not directly modify the production branch.
- Agents create Pull Requests.
- CI must validate every Pull Request.
- Merge requires successful mandatory checks.
- Conflicting changes must be reconciled before merging.

### 5.4 Deployment Behavior

- Pull Request: CI validation only.
- Merge into `main`: staging deployment and verification.
- Production: separate manual approval.

## 6. CI/CD Strategy

### 6.1 Pipeline Overview

The pipeline consists of:

1. Pull Request validation.
2. Merge and staging deployment.
3. Production release.
4. Scheduled maintenance and security scans.

### 6.2 Pull Request Pipeline

Trigger:

- Pull Request opened.
- New commit pushed.
- Pull Request synchronized.

Required checks:

1. Toolchain validation.
2. Dependency installation.
3. Configuration validation.
4. Formatting.
5. Linting.
6. Type checking.
7. Astro diagnostics.
8. Unit and component tests.
9. Integration tests where applicable.
10. Security and secret scanning.
11. Production build.
12. Bundle size checks.
13. Critical accessibility checks.
14. Migration validation when database migration files change.
15. AI-assisted PR review through PR-Agent.

A Pull Request cannot be merged while mandatory automated checks are failing.

### 6.3 Staging Deployment

Trigger:

- Successful merge into `main`.

Process:

1. Install pinned toolchain.
2. Run aggregate quality checks.
3. Build application.
4. Validate environment configuration.
5. Execute approved staging migrations.
6. Deploy staging Worker.
7. Publish or update Queue consumers and bindings.
8. Run API smoke tests.
9. Run critical E2E tests.
10. Verify lead persistence and asynchronous processing.
11. Report deployment status.

A failed migration or critical smoke test blocks release promotion.

### 6.4 Production Deployment

Trigger:

- Approved production deployment from `main`.

Process:

1. Verify release commit.
2. Verify all mandatory CI checks.
3. Verify staging acceptance.
4. Confirm migration compatibility.
5. Verify database recovery point.
6. Apply approved production migrations.
7. Deploy production Worker.
8. Verify static asset delivery.
9. Verify API availability.
10. Verify Queue consumer operation.
11. Execute controlled smoke tests.
12. Record deployment metadata.

Production deployment requires manual approval through GitHub Environment `production`.

### 6.5 GitHub Actions Security

- Pin third-party Actions to full commit SHAs.
- Use minimum required `GITHUB_TOKEN` permissions.
- Do not expose secrets to untrusted Pull Request code.
- Separate validation permissions from deployment permissions.
- Use protected GitHub Environments.
- Keep production secrets isolated.
- Avoid unnecessary privileged self-hosted runners.
- Review Action updates before changing pinned SHAs.

## 7. CI/CD Quality Gates

### 7.1 Adoption Policy

The Dev Quality Gates repository is used as a curated selection source.

Only tools relevant to the approved project stack are adopted.

Requirements:

- One pinned toolchain configuration.
- One aggregate `check` command.
- Fast local staged-file checks.
- Full-project checks before push where configured.
- Mandatory CI checks.
- Non-zero exit codes for violations.
- Narrow documented exceptions.
- No global rule suppression to bypass CI.

### 7.2 Three-Stage Adoption

#### Stage 1 — Development Foundation

<badge color="success">Approved: Initial setup</badge>

Tools:

- mise
- pnpm
- Oxlint
- oxfmt
- TypeScript strict
- Astro Check
- Stylelint
- Vitest
- hk
- commitlint
- Betterleaks

Responsibilities:

- Toolchain consistency.
- JavaScript and TypeScript quality.
- Astro diagnostics.
- SCSS quality.
- Basic test automation.
- Git hook enforcement.
- Initial secret detection.

#### Stage 2 — Advanced Quality and AI Review

<badge color="info">Approved: After baseline structure</badge>

Tools:

- Playwright
- axe-core
- Knip
- Squawk
- OSV-Scanner
- Sentry
- size-limit
- jactionlint
- PR-Agent

Responsibilities:

- E2E testing.
- Accessibility testing.
- Unused code and dependency detection.
- Database migration safety.
- Dependency vulnerability scanning.
- Application error tracking.
- Bundle size control.
- GitHub Actions workflow validation.
- AI-assisted Pull Request review.

#### Stage 3 — Extended Automation

<badge color="secondary">Approved: Progressive adoption</badge>

Tools:

- dependency-cruiser
- Renovate
- lychee
- typos
- rumdl
- sqruff
- Additional architectural checks.

Responsibilities:

- Dependency update automation.
- Architecture boundary enforcement.
- Documentation quality.
- SQL style and formatting.
- Extended repository maintenance.

### 7.3 JavaScript and TypeScript

**Oxlint**

- JavaScript and TypeScript linting.
- Project-specific rules.
- CI execution with `--deny-warnings`.
- Violations must fail the pipeline.

**oxfmt**

- Source formatting.
- CI check mode.
- Version pinned.
- No automatic rewriting in CI.

**TypeScript**

- `strict: true`.
- `tsc --noEmit`.
- No unresolved type errors in production code.

**Astro Check**

- Astro component diagnostics.
- Framework-specific type validation.
- Separate from general TypeScript checks.

Required command:

```bash
astro check
```

**Knip**

- Detect unused files.
- Detect unused exports.
- Detect unused dependencies.
- Introduced after repository structure and integrations stabilize.

**dependency-cruiser**

- Conditional adoption.
- Circular dependency detection.
- Import boundary enforcement.
- Frontend/server dependency separation.

### 7.4 SCSS and Frontend

**Stylelint**

- SCSS linting.
- CSS quality rules.
- Consistent styling conventions.

**axe-core + Playwright**

- Automated accessibility checks.
- Critical page coverage.
- WCAG 2.2 AA target.

**size-limit**

- Initial JavaScript budget.
- React Islands budgets.
- Route-specific budgets where measurable.

Exact budgets are defined after the first production build.

### 7.5 Database and SQL

**Drizzle ORM + Drizzle Kit**

- TypeScript database schema.
- SQL migration generation.
- Migration history.
- Controlled migration execution.

**Squawk**

- PostgreSQL migration safety.
- Locking and downtime risk detection.
- Release-blocking findings unless explicitly approved.

**sqruff**

- Proposed for Stage 3.
- SQL linting and formatting if handwritten SQL becomes a significant part of the repository.

### 7.6 Security

**Betterleaks**

- Secret detection.
- Pull Request scanning.
- Repository scanning according to security policy.
- No secret values in public CI logs.

**OSV-Scanner**

- Dependency vulnerability scanning.
- Lockfile validation.
- Scheduled vulnerability checks.
- Pull Request integration.

### 7.7 GitHub Actions and Git Workflow

**jactionlint**

- Workflow validation.
- Configuration error detection.
- Version pinning and maintenance review.

**hk**

- Sole Git Hooks manager.
- No Husky or lint-staged.

**commitlint**

- Conventional Commits enforcement.

### 7.8 Documentation Quality

**lychee**

- Broken link detection.

**typos**

- Spelling checks.
- Reviewed domain terminology allowlist.

**rumdl**

- Markdown linting and formatting.

### 7.9 AI Code Review — PR-Agent

**Selected tool:** PR-Agent.

Purpose:

- AI-assisted Pull Request review.
- Potential bug detection.
- Code quality feedback.
- Review of architecture and implementation changes.
- Inline review comments and suggestions.

Integration:

- GitHub Pull Request workflow.
- Automated analysis on PR creation and updates.
- Version-pinned execution.
- Protected LLM and GitHub credentials.
- Review output visible within Pull Requests.

Rules:

1. PR-Agent is introduced in Stage 2.
2. AI review is advisory initially.
3. AI findings do not automatically approve or merge Pull Requests.
4. Mandatory Quality Gates remain authoritative.
5. Review findings must be evaluated by a developer.
6. PR-Agent must not receive production deployment permissions.
7. Secrets and unnecessary personal data must be excluded from prompts and review context.
8. LLM provider configuration must be explicitly documented.
9. The project must account for the data handling and retention policies of the selected LLM provider.

PR-Agent is an additional review layer, not a replacement for deterministic tests, linting, static analysis, or human approval.

### 7.10 Aggregate Commands

Required logical commands:

```bash
mise run lint
mise run format:check
mise run typecheck
mise run astro:check
mise run test
mise run test:e2e
mise run security
mise run db:check
mise run check
mise run fix
mise run build
```

`check` is the aggregate quality command and is mandatory in CI.

`fix` is for safe local autofixes only.

### 7.11 Quality Gate Enforcement

For every adopted gate:

- Run against a clean repository.
- Introduce a controlled violation in a temporary branch.
- Confirm non-zero exit code.
- Revert the violation.
- Confirm GitHub Actions executes the same check.
- Document any infrastructure or credential limitation.

## 8. Build and Artifact Management

### 8.1 Production Build

The production build must:

- Generate Astro static pages.
- Generate Worker-compatible output.
- Include required static assets.
- Exclude development-only dependencies and configuration.
- Preserve localized routes.
- Generate SEO artifacts.
- Validate environment-dependent settings.

### 8.2 Build Reproducibility

- Use committed lockfile.
- Use pinned runtime and package manager.
- Avoid unpinned global tools.
- Record commit SHA.
- Avoid manual edits to generated files.

## 9. Cloudflare Deployment

### 9.1 Deployment Target

**Approved:** Cloudflare Workers + Static Assets.

Cloudflare Pages is not the selected deployment target.

### 9.2 Worker Configuration

Version-controlled configuration must define:

- Worker entry point.
- Static Assets binding.
- Compatibility date.
- Environment-specific bindings.
- Queue producer and consumer configuration.
- R2 bindings where required.
- Database connectivity.
- Environment-specific variables.

Final configuration format will be selected during implementation.

### 9.3 Static Asset Delivery

Verify:

- Correct asset paths.
- Correct cache behavior.
- Correct content types.
- Localized route resolution.
- Missing asset handling.
- Deployment of generated files.

Hashed assets may use long-lived immutable caching.

### 9.4 Worker API

The API must:

- Use same-origin routes under `/api/v1/`.
- Validate incoming requests.
- Apply Turnstile verification.
- Apply rate limiting.
- Persist valid leads.
- Write Outbox records transactionally.
- Publish asynchronous tasks.
- Avoid exposing internal infrastructure details.

### 9.5 Queue Deployment

Deployment verification must include:

- Queue existence.
- Correct producer binding.
- Correct consumer deployment.
- Retry configuration.
- Dead-letter handling.
- Consumer observability.
- Reconciliation process.

Successful Worker deployment does not automatically prove that notification processing is healthy.

## 10. Database Deployment and Migrations

### 10.1 Database Standard

**Approved:**

- Neon PostgreSQL.
- Drizzle ORM.
- Drizzle Kit.
- Squawk for migration safety checks.

### 10.2 Migration Strategy

Migrations must be:

- Version-controlled.
- Reviewed before execution.
- Generated and maintained using Drizzle Kit.
- Tested in staging before production.
- Immutable after application.

Migrations must not run automatically at Worker startup.

### 10.3 Migration Execution

Production migrations are executed through a dedicated CI/CD stage.

Requirements:

- Separate migration credentials.
- Minimum necessary privileges.
- No use of production application credentials for unrestricted schema changes.
- Migration failure blocks release promotion.
- Migration logs and applied versions are traceable.

### 10.4 Expand-and-Contract Strategy

For potentially breaking schema changes:

1. Add compatible schema elements.
2. Deploy code supporting both old and new schema.
3. Backfill data if necessary.
4. Switch application logic.
5. Verify production behavior.
6. Remove obsolete schema in a later release.

### 10.5 Hyperdrive Decision

**Approved:** Hyperdrive is deferred.

MVP uses direct Worker-to-Neon connectivity through a compatible server-side PostgreSQL driver.

Before implementation, validate:

- Neon compatibility.
- Cloudflare Workers runtime compatibility.
- Driver support.
- Transaction behavior.
- Connection management.
- Pooling behavior.

Hyperdrive may be reconsidered if measurable performance or connection-management requirements justify it.

### 10.6 Neon Environment Topology

**Approved:** one Neon Project with separate branches.

```text
Neon Project
    |
    +-- production
    |
    +-- staging
```

Requirements:

- Separate connection strings.
- Separate credentials.
- Isolated test data.
- Independent migration verification.
- No automatic test-data promotion into Production.

A separate Neon branch is not equivalent to full project-level infrastructure isolation. Access policies and resource limits must still be controlled.

## 11. Vercel — Alternative Deployment Platform

### 11.1 Status

**Status:** Approved as an alternative platform, not primary MVP infrastructure.

Cloudflare remains the primary deployment platform.

Vercel is documented as a potential replacement if the project owner decides to migrate.

### 11.2 Migration Objective

The architecture should avoid unnecessary provider-specific coupling where practical.

The application should preserve portable components:

- Astro application.
- TypeScript business logic.
- React Islands.
- SCSS.
- Drizzle ORM.
- Neon PostgreSQL.
- Resend.
- Telegram Bot API.
- Sentry.
- GitHub Actions.

### 11.3 Components Requiring Adaptation

The following Cloudflare-specific components must not be assumed to work unchanged on Vercel:

- Cloudflare Worker API runtime.
- Static Assets bindings.
- Cloudflare Queues.
- R2 bindings.
- Cloudflare-specific environment configuration.
- Turnstile integration.
- Worker-specific database connectivity.
- Wrangler deployment configuration.
- Worker consumer implementation.

A migration would require evaluating:

- Astro deployment adapter.
- API runtime and serverless execution model.
- Background processing alternative to Cloudflare Queues.
- Media storage integration.
- Environment variables and secrets.
- Database driver compatibility.
- CI/CD deployment configuration.
- Observability and operational procedures.

### 11.4 Portability Strategy

Where practical:

- Keep business logic independent of Worker-specific APIs.
- Isolate provider integrations behind service interfaces.
- Keep notification delivery behind adapters.
- Keep lead persistence independent of hosting provider.
- Keep media storage access behind a storage abstraction where justified.
- Avoid unnecessary Cloudflare-specific imports in shared business logic.

Do not introduce excessive abstraction solely to support a hypothetical migration.

### 11.5 Vercel Migration Decision

A move to Vercel requires a separate technical assessment and approval.

It is not a simple hosting switch and must not be represented as a drop-in replacement for Cloudflare.

Neon, Drizzle, Resend, Telegram and Sentry may remain in use if their compatibility and operational requirements are validated.

## 12. Production Approval

### 12.1 GitHub Environment

**Environment:** `production`

Requirements:

- Required reviewer: project owner.
- Deployment branch restriction: `main`.
- Production secrets isolated.
- Manual approval required.
- Self-review prevention enabled where supported by available GitHub settings.

### 12.2 AI Agent Restrictions

AI agents may:

- Create Pull Requests.
- Execute CI checks.
- Prepare release changes.
- Propose migrations.
- Review deployment configuration.

AI agents must not:

- Directly merge unapproved changes.
- Access unrestricted production credentials.
- Approve their own production deployment.
- Execute destructive production operations without explicit authorization.

## 13. Sentry Monitoring

**Approved:** Sentry is included from MVP.

### 13.1 Responsibilities

- Application error tracking.
- Exception reporting.
- Stack trace collection.
- Release tracking.
- Environment tracking.
- Performance diagnostics where configured.

### 13.2 Environments

Sentry must distinguish:

- Development.
- Staging.
- Production.

### 13.3 Release Tracking

Events should include:

- Release identifier.
- Commit SHA where available.
- Environment.
- Relevant application component.
- Error category.

### 13.4 Privacy and Data Filtering

Before sending events to Sentry:

- Remove authentication tokens.
- Remove API keys.
- Remove database credentials.
- Avoid collecting full customer messages.
- Mask phone numbers and email addresses where not necessary.
- Avoid unnecessary personal data in breadcrumbs.
- Review request payload collection and session replay configuration.

Sentry is an application observability layer, not a replacement for Cloudflare infrastructure logs.

## 14. Monitoring and Alerting

### 14.1 Monitoring Architecture

**Cloudflare Observability**

- Worker errors.
- Request volume.
- Latency.
- Queue failures.
- Infrastructure diagnostics.

**Neon Monitoring**

- Database availability.
- Connection errors.
- Compute and storage usage.
- Database resource consumption.

**GitHub Actions**

- Failed builds.
- Failed migrations.
- Deployment errors.
- Quality Gate failures.

**Sentry**

- Application exceptions.
- Error trends.
- Release regressions.
- Application diagnostics.

**Telegram Alerts**

- Critical incidents.
- Operational notifications.
- Selected infrastructure failures.

### 14.2 Critical Events

Alert on:

- Worker unavailable.
- Persistent Lead API failures.
- Database connection failure.
- Lead persistence failure.
- Outbox backlog that cannot recover.
- Queue dead-letter growth.
- Suspected unauthorized access.
- Secret exposure.
- Critical production deployment failure.

### 14.3 Warning Events

Monitor:

- API response time increase.
- Resend failures.
- Telegram failures.
- Queue retry growth.
- Outbox backlog increase.
- Resource consumption growth.

Thresholds are configured after baseline measurements.

## 15. Deployment and Rollback

### 15.1 Deployment Metadata

Every deployment must be traceable through:

- Git commit SHA.
- Build identifier.
- Environment.
- Deployment timestamp.
- Migration version.
- Release identifier.

### 15.2 Application Rollback

If a deployment introduces a critical defect:

1. Stop further promotion.
2. Identify last known-good Worker version.
3. Restore the previous application version using the approved Cloudflare mechanism.
4. Verify public routes.
5. Verify Lead API.
6. Verify Queue consumer.
7. Review logs and Sentry events.
8. Record the incident.

### 15.3 Database Rollback

Database rollback is handled separately.

Requirements:

- Prefer forward-fix migrations where safe.
- Use reversible migrations only when data-safe.
- Verify recovery options before destructive changes.
- Do not assume application rollback restores the previous schema.

### 15.4 Rollback Triggers

Rollback or release suspension is required when:

- Critical pages are unavailable.
- Lead submissions fail.
- Lead records are corrupted or lost.
- Unauthorized data access is detected.
- Critical security vulnerability is introduced.
- Migration makes the application unusable.
- Notification pipeline cannot recover within operational tolerance.

## 16. Backup and Recovery

### 16.1 Recovery Targets

**Approved RPO:** 1 hour.

**Approved RTO:** 4 hours.

These are project recovery objectives, not guarantees supplied automatically by hosting providers.

### 16.2 Neon PostgreSQL

Requirements:

- Use available backup and Point-in-Time Recovery capabilities.
- Verify retention on the selected plan.
- Document recovery permissions.
- Test restoration on an isolated branch.
- Record recovery verification.

A Neon Branch is not considered a complete independent backup.

### 16.3 Cloudflare R2

Requirements:

- Protect important original media.
- Document object deletion protection where available.
- Maintain independent copies of critical source assets.
- Test media recovery.

### 16.4 GitHub

Source code, deployment configuration, and migrations must remain under version control.

Secrets must never be stored in the repository.

### 16.5 Recovery Testing

Recovery procedures must be tested before production launch and periodically thereafter.

A backup that has never been restored is not a verified recovery mechanism.

## 17. Performance Budgets

### 17.1 Core Web Vitals

Approved target values:

- LCP ≤ 2.5 seconds.
- INP ≤ 200 milliseconds.
- CLS ≤ 0.1.

### 17.2 Additional Budgets

The following budgets will be established after the first production build:

- Initial JavaScript bundle.
- React Islands bundle.
- Route-specific JavaScript.
- Image payload size.
- Font payload.
- Critical CSS.
- API response latency.

### 17.3 Performance Tooling

Lighthouse CI is used for critical page checks.

Core Web Vitals must be evaluated using both controlled testing and real-user data when sufficient production traffic is available.

Lighthouse scores are diagnostic indicators and are not substitutes for field Core Web Vitals.

Heavy animations must not compromise initial content rendering, accessibility, or form usability.

## 18. Cost and Resource Management

Monitor:

- Cloudflare Workers usage.
- R2 storage and delivery.
- Neon compute and storage.
- Queue consumption.
- Resend usage.
- Sentry event volume.
- GitHub Actions minutes.
- Build artifact retention.

Avoid introducing always-running infrastructure without demonstrated operational need.

Resource limits must be reviewed against actual traffic and provider pricing.

## 19. Repository Structure

Proposed repository structure:

```text
project-root/
├── .github/
│   └── workflows/
│       ├── ci.yml
│       ├── deploy-staging.yml
│       ├── deploy-production.yml
│       └── scheduled-security.yml
│
├── .config/
├── mise.toml
├── package.json
├── pnpm-lock.yaml
├── astro.config.ts
├── wrangler.jsonc
├── tsconfig.json
├── drizzle.config.ts
│
├── database/
│   ├── schema/
│   └── migrations/
│
├── src/
│   ├── components/
│   ├── content/
│   ├── layouts/
│   ├── pages/
│   ├── styles/
│   └── server/
│
├── tests/
│
├── AGENTS.md
└── README.md
```

The final layout follows the selected Astro integration, Worker architecture, and package tooling.

## 20. AI Agent and Repository Rules

The repository must contain `AGENTS.md`.

Minimum rules:

- Run `mise run check` before completing a change.
- Do not edit generated files manually.
- Do not disable lint rules to bypass CI.
- Keep exceptions narrow and documented.
- Do not commit secrets.
- Do not modify production configuration without approval.
- Do not edit applied database migrations.
- Do not execute destructive production operations without explicit authorization.
- Update tests when business logic changes.
- Keep architecture documentation synchronized with implementation.
- Use separate branches or worktrees for parallel agent work.
- Do not merge without mandatory CI checks.
- Treat PR-Agent comments as review suggestions requiring evaluation.

## 21. Acceptance Criteria

DEPLOY-001 is considered implemented when:

- Development, staging, and production environments are defined.
- mise manages the toolchain.
- pnpm is configured with a committed lockfile.
- Local and CI versions are consistent.
- Aggregate `check` command exists.
- Stage 1 Quality Gates run successfully.
- GitHub Actions CI is configured.
- PR-Agent integration is documented and configured when Stage 2 begins.
- Sentry integration is operational.
- Pull Request checks are mandatory.
- Branch protection is operational.
- Staging deployment is repeatable.
- Production deployment requires manual approval.
- Cloudflare Worker and Static Assets deployment is operational.
- Database migrations are managed through Drizzle Kit.
- Migration safety checks run through Squawk.
- Queue producer and consumer deployment is verified.
- R2 media delivery works.
- Rollback procedures are documented.
- Monitoring and alerting are operational.
- Backup and recovery procedures are verified.
- RPO and RTO targets have been tested.
- Core Web Vitals are measured.
- Deployment metadata is traceable to Git commits.
- Vercel alternative deployment requirements are documented sufficiently to support a future migration assessment.

## 22. Approved Decisions Register

| Decision ID | Topic | Approved Decision |
| --- | --- | --- |
| OD-DEPLOY-001-01 | Package Manager | pnpm |
| OD-DEPLOY-001-02 | Branching Strategy | Trunk-Based Development, one permanent `main` branch and short-lived branches |
| OD-DEPLOY-001-03 | Toolchain | mise with pinned versions |
| OD-DEPLOY-001-04 | Git Hooks | hk, without Husky or lint-staged |
| OD-DEPLOY-001-05 | Database Connectivity | Direct Worker-to-Neon connection; Hyperdrive deferred |
| OD-DEPLOY-001-06 | Database Migrations | Drizzle ORM + Drizzle Kit |
| OD-DEPLOY-001-07 | Staging Database | Separate Neon staging branch in the same project |
| OD-DEPLOY-001-08 | Production Approval | Manual approval through GitHub Environment |
| OD-DEPLOY-001-09 | Monitoring | Sentry, Cloudflare Observability, Neon Monitoring, GitHub Actions, Telegram Alerts; Vercel as alternative deployment platform |
| OD-DEPLOY-001-10 | Recovery Targets | RPO 1 hour; RTO 4 hours |
| OD-DEPLOY-001-11 | Quality Gates | Three-stage adoption, with PR-Agent in Stage 2 |
| OD-DEPLOY-001-12 | Performance Budgets | Core Web Vitals targets; detailed bundle and media budgets after first production build |

## 23. Consolidated Deployment Decisions and Consistency Rules


### 23.1 Document Authority

This revision consolidates the previous DEPLOY-001 v0.2 and DEPLOY-001 v0.3 drafts into one current specification. If an older copy conflicts with this document, this consolidated v0.4 is authoritative.


### 23.2 Deployment Scope

The MVP deployment is a single primary Cloudflare Worker deployment with logical separation for:


- Public API.
- Internal API.
- Lead Management Telegram webhook.
- Incident & Operations Telegram webhook.
- Application Services.
- Outbox Dispatcher.
- Lead Notifications Consumer.
- Operations Events Consumer.
- Scheduled Jobs.
- Health Checks.


Separate Workers per bot are not required for the MVP. Credentials, webhook endpoints, authorization scopes, and operational responsibilities remain isolated.


### 23.3 Two Telegram Bots — Deployment Requirements

**Lead Management Bot** and **Incident & Operations Bot** must be configured independently:

- Separate bot tokens.
- Separate webhook endpoints.
- Separate webhook secrets.
- Separate authorized Telegram User ID allowlists and permission scopes.
- Separate staging and production credentials.
- Separate validation and smoke-test procedures.

Webhook-secret validation authenticates the webhook request. It does not replace authorization of the Telegram user issuing a command.


### 23.4 Webhook Endpoints

The endpoints are:


- Lead Management: `POST /api/v1/webhooks/telegram/leads`
- Incident & Operations: `POST /api/v1/webhooks/telegram/incident`

The deployed endpoint must verify the corresponding Telegram webhook secret before processing an update. Test and production webhook registrations must be separate.


### 23.5 Queues, Consumers, and Reconciliation

The MVP logical queues are:


- `lead-notifications`
- `operations-events`

The deferred `lead-commands` queue is not required for MVP. Staging and production must use isolated Queue resources and configuration.


The target retry policy is five attempts including the initial attempt, with retry delays of 1, 5, 15, and 60 minutes, followed by DLQ handling. These intervals are targets and must be validated against Cloudflare Queues capabilities; unsupported behavior requires explicit application-level implementation rather than assumed platform behavior.


Outbox reconciliation is targeted to run every five minutes. Reconciliation must be safe to repeat and must not create duplicate business actions. Consumers must use idempotency safeguards and delivery state.


### 23.6 Scheduled Jobs

Scheduled responsibilities include:


- Reminder scheduling and processing.
- 2-hour and 24-hour follow-up checks.
- Daily summary.
- Outbox reconciliation.
- Recovery of expired processing leases.
- Detection of stuck tasks.

The exact cron expressions, daily summary time, and timezone remain configuration decisions. Scheduled handlers must not be exposed as public HTTP endpoints.


### 23.7 Correlation and Event Envelope

Correlation IDs must propagate across HTTP or Telegram ingress, application services, database/outbox records, queue messages, consumers, and external provider calls/logs where supported. A correlation ID is for tracing and must not contain personal information.

The minimum event envelope includes:


- `eventId`
- `eventType`
- `eventVersion`
- `correlationId`
- Aggregate type and identifier
- `occurredAt`
- `payload`

`eventId` identifies an event; `correlationId` groups a workflow; `idempotencyKey` prevents duplicate processing. They are not interchangeable. Incoming IDs must be validated and normalized at the boundary.


### 23.8 Outbox and External Delivery

Database persistence and Outbox creation must occur within the same database transaction. Queue dispatch is asynchronous and is not assumed to be atomic with the database transaction. Reconciliation closes the gap between Outbox state and Queue delivery.

External provider calls (Telegram, Resend) must occur outside database transactions. Delivery state and idempotency controls must prevent retries from repeating already completed business actions where the provider and operation permit.


### 23.9 Direct Alerts and AI Analysis

Direct operational alerts must remain independent of optional AI analysis. An AI analysis failure, timeout, or unavailable provider must not prevent the underlying alert from being generated, delivered, acknowledged, or resolved.

AI analysis is optional and must receive only sanitized diagnostic context. It must not have direct database access, secrets, unrestricted production credentials, or authority to perform destructive production operations.


### 23.10 Database and Environment Boundaries

Neon PostgreSQL is the source of truth. The deployment document defines migration execution and environment isolation; DATA-001 remains authoritative for database entities, fields, relationships, constraints, indexes, and schema-level decisions.

A single Neon Project with separate staging and production branches is the approved initial topology. Separate credentials are required. Branches alone do not provide full project-level security isolation; access controls and resource limits must be verified.


### 23.11 Deployment Order and Compatibility

The normal release order is:


1. Validate the migration and application compatibility.
2. Apply migration to staging.
3. Verify staging database state.
4. Deploy staging Worker.
5. Verify public routes and API.
6. Verify Queue producers, consumers, and processing.
7. Verify Telegram webhooks and both bots.
8. Verify Scheduled Jobs and reminders.
9. Run staging smoke/E2E checks.
10. Obtain production approval.
11. Apply approved production migration.
12. Deploy production Worker.
13. Verify health, API, static assets, Queue processing, both bots, alerts, and tracing.
14. Record deployment outcome and release metadata.

For breaking schema changes, use expand-and-contract. Do not deploy code that requires a schema not yet present, and do not remove compatibility fields until all consumers have moved.


### 23.12 Operational Readiness

The release readiness checklist must include explicit verification of:


- Both bot tokens and webhook secrets.
- Webhook registration and secret validation.
- Telegram User ID allowlist and permission scopes.
- Queue producers, consumers, retry settings, DLQ, and monitoring.
- Outbox reconciliation.
- Scheduled triggers and timezone configuration.
- Correlation ID propagation.
- Direct alert delivery independent of AI analysis.
- Staging and production separation.
- Production approval.
- Rollback and recovery procedures.


### 23.13 Scope and Status

This document specifies the target deployment architecture and procedures. It does not assert that infrastructure, pipelines, migrations, bots, monitoring, backup policies, or restore procedures have already been implemented or tested. Implementation status must be verified in the repository and provider configuration.


## 24. Document Boundaries

- **DEPLOY-001** is the source of truth for environments, toolchain, CI/CD, deployment, migrations execution, release controls, rollback, monitoring deployment, and recovery procedures.
- **API-001** is the source of truth for API endpoints, request/response contracts, webhook behavior, authorization rules, and application-level event/notification contracts.
- **DATA-001** is the source of truth for database schema, entities, fields, indexes, constraints, and relationships.
- **OPS-001** is the source of truth for operational runbooks, incident response, alert handling, and routine maintenance procedures.
- **TEST-001** is the source of truth for test strategy, test scope, and quality validation approach.


## 25. Related Documents

- BRD-001 — Business Requirements.
- SRS-001 — Software Requirements Specification.
- SAD-001 — System Architecture.
- DATA-001 — Data Model.
- API-001 — API Specification.
- ADR-001 — Architecture Decision Records.
- IA-001 — Information Architecture.
- UX-001 — UX Specification.
- BRAND-001 — Brand Specification.
- CONTENT-001 — Content Specification.
- SEO-001 — SEO/GEO Specification.
- TECH-001 — Telegram Bots Technical Architecture.
- OPS-001 — Operations & Monitoring.
- TEST-001 — Testing Strategy.
- IMPLEMENTATION-001 — Implementation Plan.


## Change Log

| Version | Date | Changes |
| --- | --- | --- |
| 0.1 | 2026-11-17 | Initial deployment architecture. |
| 0.2 | 2026-11-18 | Toolchain, Trunk-Based Development, hk, Drizzle, staged Quality Gates, Sentry, backup, recovery targets, Cloudflare primary, Vercel alternative, and detailed CI/CD controls. |
| 0.3 | 2026-11-19 | Added two Telegram bots, isolated webhook credentials, Queue resources, Scheduled Worker, Correlation ID, Direct Alert, deployment order, production validation, and readiness checklist. |
| 0.4 | 2026-11-20 | Consolidated v0.2 and v0.3 into one expanded specification; clarified deployment boundaries, webhook authorization, Outbox/Queue reconciliation, idempotency, AI-independent alerts, environment isolation, and implementation status. |


**Current version:** 0.4 — Consolidated Specification  
**Status:** Draft — requires implementation verification and project-owner/technical-lead approval.

