# Plan 1 — Foundation and Staff IAM Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** A runnable monorepo with the NestJS API, PostgreSQL via Prisma, Redis, CI, an append-only audit log, and back-office staff authentication with mandatory TOTP MFA and role-based permissions.

**Architecture:** Modular monolith (design §6). This plan builds the shared kernel (config, database, Redis, request context, domain errors), the `audit` module and the `iam` module, each with `domain/ application/ infrastructure/ http/` folders. Every later module plugs into these.

**Tech Stack:** Node.js 24.14.1, pnpm workspaces, TypeScript strict, NestJS 12.5.1 (CommonJS project consuming ESM packages via `require(esm)`), Prisma 7.10.1 with `@prisma/adapter-pg`, PostgreSQL 17, Redis 7, ioredis, class-validator, Zod (env validation), `@node-rs/argon2`, otplib 13, helmet, cookie-parser, Jest 30 + ts-jest, Supertest, GitHub Actions.

**Spec:** `docs/superpowers/specs/2026-11-18-cuba-store-platform-design.md` (sections 3.1, 6, 7.2 store settings excluded, 7.9, 8, 9, 10, 11).

**Plan series:** 1 Foundation + staff IAM (this) · 2 Settings, reference data, pricing · 3 Catalog, media, inventory · 4 Cart, checkout, orders, payments, outbox/worker · 5 Shipping, notifications, reports, import · 6 Admin frontend · 7 Store frontend · 8 Production infra, backups, hardening.

## Global Constraints

- Node.js `>=24.9` (`engines`), pinned `24.14.1` in `.nvmrc` and CI. Jest cannot load ESM-only NestJS 12 below 24.9 (`ERR_REQUIRE_ASYNC_MODULE`).
- Prisma pinned to stable `7.10.1` (npm `latest` tag is `8.0.0-rc.34`, a release candidate — do not use).
- All dependency versions pinned exactly (no `^`/`~`). No dependency beyond the Tech Stack line without a written justification in the commit body.
- TypeScript `strict: true`, `noUncheckedIndexedAccess: true`.
- Code, identifiers, comments, API messages: English.
- Money is never handled in this plan; IDs are UUID v7 (`@default(uuid(7))`).
- Every table: `created_at`, `updated_at`. Append-only tables get no UPDATE/DELETE grant for the runtime role.
- Two database roles: `ecommerce_owner` (migrations, `MIGRATION_DATABASE_URL`) and `ecommerce_app` (runtime, `DATABASE_URL`).
- Nothing is built or pulled from Cuba: CI builds everything (design §3.2).
- Conventional Commits. No AI attribution lines in commits.
- Work on branch `feat/foundation-staff-iam`, never on `main`.
- Prerequisite on the dev machine: Docker Desktop (not installed at plan time) and `corepack enable` for pnpm `10.x`.

## Review Focus

1. **Email typed with different case or surrounding spaces** (`" Admin@Store.com "`) must log into the account created as `admin@store.com` → Task 6 test `normalizes email on login`.
2. **Phone clock a little off** (common on phones in Cuba): a TOTP code from the previous 30-second step must be accepted; one from two steps back must not; the same code must not work twice → Task 7 tests.
3. **Deactivated employee with an open session** must be rejected on the very next request → Task 8 test `deactivated user session is rejected`.
4. **Locking the business out**: removing or deactivating the last active superadministrator, or yourself, must fail with 409 → Task 8 tests.
5. **Flaky connection retries**: a retried logout or a request with an expired session must return a clean 401, never a 500 → Task 6 test `expired session returns 401`.

---

## File Structure

```
package.json, pnpm-workspace.yaml, tsconfig.base.json, eslint.config.mjs, .prettierrc.json,
.editorconfig, .gitignore, .nvmrc, .env.example
infra/docker/docker-compose.yml
infra/docker/postgres/init/01-roles-and-databases.sql
packages/config/                          shared tsconfig/eslint presets
apps/api/
  package.json, nest-cli.json, tsconfig.json, tsconfig.build.json, jest.config.ts, prisma.config.ts
  prisma/schema.prisma, prisma/migrations/**, prisma/seed.ts
  src/main.ts, src/app.module.ts, src/bootstrap.ts           (bootstrap shared by main and tests)
  src/config/env.schema.ts, src/config/app-config.module.ts
  src/shared/domain/errors.ts
  src/shared/http/domain-error.filter.ts
  src/shared/context/request-context.ts                       AsyncLocalStorage
  src/shared/database/prisma.service.ts, src/shared/database/tx.ts
  src/shared/redis/redis.module.ts, src/shared/redis/rate-limiter.ts
  src/modules/health/health.controller.ts, health.module.ts
  src/modules/audit/{domain/audit-entry.ts, application/audit.service.ts,
                     infrastructure/prisma-audit.repository.ts, http/audit.controller.ts, audit.module.ts}
  src/modules/iam/domain/{permissions.ts, roles.ts, password-policy.ts, user.ts}
  src/modules/iam/application/{auth.service.ts, mfa.service.ts, users.service.ts, ports.ts}
  src/modules/iam/infrastructure/{argon2-password-hasher.ts, otplib-totp.ts, aes-gcm-secret-box.ts,
                                  prisma-user.repository.ts, prisma-session.repository.ts}
  src/modules/iam/http/{auth.controller.ts, users.controller.ts, dto/*.ts,
                        session.guard.ts, permissions.guard.ts, recent-mfa.guard.ts, csrf.guard.ts, decorators.ts}
  src/modules/iam/iam.module.ts
  src/cli/create-superadmin.ts
  test/integration/{global-setup.ts, helpers/app.ts, helpers/db.ts, helpers/login.ts, **/*.int-spec.ts}
.github/workflows/ci.yml
```

---

### Task 1: Monorepo, tooling and local infrastructure

**Files:**
- Create: root files listed above, `packages/config/*`, `infra/docker/docker-compose.yml`, `infra/docker/postgres/init/01-roles-and-databases.sql`

**Interfaces:**
- Produces: `pnpm lint`, `pnpm typecheck`, `pnpm test` at root (delegating with `pnpm -r`); Compose services `postgres` (port 5432) and `redis` (port 6379).

- [ ] **Step 1: Create the branch** — `git switch -c feat/foundation-staff-iam`.
- [ ] **Step 2: Root workspace** — `package.json` with `"private": true`, `"packageManager": "pnpm@10.x"` (exact latest 10.x at execution), `"engines": {"node": ">=24.9"}`; `pnpm-workspace.yaml` with `apps/*`, `packages/*`; `tsconfig.base.json` with the Global Constraints compiler flags; ESLint flat config with `typescript-eslint` strict-type-checked; Prettier (`singleQuote`, `printWidth: 100`).
- [ ] **Step 3: Compose** — `postgres:17-alpine` (env `POSTGRES_USER=postgres`, password from `.env`), `redis:7-alpine` with `--appendonly yes`; mount the init SQL.
- [ ] **Step 4: Init SQL** creates roles `ecommerce_owner` (LOGIN) and `ecommerce_app` (LOGIN), databases `ecommerce` and `ecommerce_test` owned by `ecommerce_owner`, and in each database:
  ```sql
  ALTER DEFAULT PRIVILEGES FOR ROLE ecommerce_owner IN SCHEMA public
    GRANT SELECT, INSERT, UPDATE, DELETE ON TABLES TO ecommerce_app;
  ALTER DEFAULT PRIVILEGES FOR ROLE ecommerce_owner IN SCHEMA public
    GRANT USAGE, SELECT ON SEQUENCES TO ecommerce_app;
  GRANT USAGE ON SCHEMA public TO ecommerce_app;
  ```
- [ ] **Step 5: `.env.example`** with every variable of Task 2's schema and safe local values.
- [ ] **Step 6: Verify** — `docker compose -f infra/docker/docker-compose.yml up -d` then `docker compose ... exec postgres psql -U postgres -c "\du"` lists both roles. `pnpm install` succeeds.
- [ ] **Step 7: Commit** — `chore: scaffold pnpm monorepo and local postgres/redis`.

### Task 2: API skeleton, config validation, errors, request context, health

**Files:**
- Create: `apps/api/{package.json,nest-cli.json,tsconfig*.json,jest.config.ts}`, `src/main.ts`, `src/bootstrap.ts`, `src/app.module.ts`, `src/config/*`, `src/shared/domain/errors.ts`, `src/shared/http/domain-error.filter.ts`, `src/shared/context/request-context.ts`, `src/shared/redis/redis.module.ts`, `src/modules/health/*`
- Test: `src/config/env.schema.spec.ts`, `src/shared/http/domain-error.filter.spec.ts`, `test/integration/health.int-spec.ts`

**Interfaces:**
- Produces:
  - `parseEnv(raw: NodeJS.ProcessEnv): AppEnv` with fields `NODE_ENV ('development'|'test'|'production')`, `PORT (number, default 3001)`, `DATABASE_URL`, `MIGRATION_DATABASE_URL`, `REDIS_URL`, `MFA_ENCRYPTION_KEY` (base64, decodes to exactly 32 bytes), `CORS_ORIGINS` (comma list → `string[]`), `COOKIE_SECURE` (`'true'|'false'` → boolean), `SESSION_IDLE_MINUTES` (default 30), `SESSION_ABSOLUTE_HOURS` (default 12). Injected as `APP_ENV` token.
  - `abstract class DomainError extends Error { abstract readonly code: string }` with subclasses `ValidationError`(400), `UnauthorizedError`(401), `ForbiddenError`(403), `NotFoundError`(404), `ConflictError`(409), `TooManyRequestsError`(429). Filter body: `{ error: { code, message, requestId } }`. Unknown errors → 500 with code `internal_error`, message never leaks the original.
  - `RequestContext.get(): { requestId: string; ip: string; userAgent: string; actor?: { type: 'USER'|'SYSTEM'; id: string } }`, `RequestContext.setActor(actor)`. `requestId` taken from `X-Request-Id` if it is a UUID, else generated; echoed in the response header.
  - `REDIS` injection token → `ioredis` client.
  - `createApp(): Promise<INestApplication>` in `bootstrap.ts` — applies helmet, cookie-parser, CORS (`CORS_ORIGINS`, `credentials: true`), global `ValidationPipe({ whitelist: true, forbidNonWhitelisted: true, transform: true })`, the filter, `ConsoleLogger` with `json: true`, Swagger at `/docs` unless production. Used by `main.ts` and integration tests.

- [ ] **Step 1: Write failing unit tests**
  - `env.schema.spec.ts`: `rejects MFA key that is not 32 bytes` → throws mentioning `MFA_ENCRYPTION_KEY`; `parses CORS_ORIGINS into array` → `['https://a.test','https://b.test']`; `applies defaults` → `PORT === 3001`, `SESSION_IDLE_MINUTES === 30`.
  - `domain-error.filter.spec.ts`: `maps ConflictError to 409 with code`; `hides unknown error message` → status 500, body message `'Internal server error'`.
- [ ] **Step 2: Run** `pnpm --filter @ecommerce/api test:unit` → FAIL (modules missing).
- [ ] **Step 3: Implement** the files above. Jest config has two projects: `unit` (`src/**/*.spec.ts`) and `integration` (`test/integration/**/*.int-spec.ts`, `globalSetup`, `--runInBand`). Scripts: `test:unit`, `test:int`, `test`.
- [ ] **Step 4: Health** — `GET /health` → 200 `{ status: 'ok', checks: { database: 'ok', redis: 'ok' } }`; any failed check → 503 with that check `'down'`. Database check is added in Task 3 (`SELECT 1`); here only Redis `PING`.
- [ ] **Step 5: Integration test** `health.int-spec.ts`: `GET /health returns 200 and echoes X-Request-Id`.
- [ ] **Step 6: Run** `pnpm --filter @ecommerce/api test` → PASS.
- [ ] **Step 7: Commit** — `feat(api): add nest skeleton with env validation, error mapping and health`.

### Task 3: Prisma, migrations, transactions and test database

**Files:**
- Create: `apps/api/prisma.config.ts`, `prisma/schema.prisma`, `src/shared/database/prisma.service.ts`, `src/shared/database/tx.ts`, `test/integration/global-setup.ts`, `test/integration/helpers/db.ts`
- Modify: `src/modules/health/*` (database check)

**Interfaces:**
- Produces:
  - Generator: `generator client { provider = "prisma-client" output = "../src/generated/prisma" moduleFormat = "cjs" }` (generated folder gitignored; `postinstall` runs `prisma generate`). Confirm `moduleFormat` against Prisma 7.10.1 docs at execution.
  - `prisma.config.ts` uses `MIGRATION_DATABASE_URL` (migrations run as owner).
  - `PrismaService extends PrismaClient` built with `new PrismaPg({ connectionString: env.DATABASE_URL })`, connects on module init, disconnects on shutdown.
  - `type Tx = Prisma.TransactionClient`; `PrismaService.inTransaction<T>(fn: (tx: Tx) => Promise<T>): Promise<T>` with isolation `ReadCommitted`, timeout 10 s.
  - Test helpers: `resetDatabase(): Promise<void>` truncates every table except `_prisma_migrations` (`TRUNCATE ... RESTART IDENTITY CASCADE` as owner); global setup runs `prisma migrate deploy` against `ecommerce_test`.
  - Scripts: `db:migrate` (`prisma migrate dev`), `db:deploy`, `db:seed`.

- [ ] **Step 1: Failing integration test** `health.int-spec.ts` add `reports database ok`.
- [ ] **Step 2: Run** `test:int` → FAIL.
- [ ] **Step 3: Implement** Prisma setup with an empty initial migration `0000_init` (no models yet) and the database health check.
- [ ] **Step 4: Run** `pnpm --filter @ecommerce/api db:deploy && pnpm --filter @ecommerce/api test` → PASS.
- [ ] **Step 5: Commit** — `feat(api): wire prisma 7 with pg adapter, migrations and test database`.

### Task 4: Audit module (append-only)

**Files:**
- Create: `src/modules/audit/**`, migration `0001_audit_log`
- Test: `src/modules/audit/domain/audit-entry.spec.ts`, `test/integration/audit/audit.int-spec.ts`

**Interfaces:**
- Produces:
  - Table `audit_log`: `id uuid pk`, `occurred_at timestamptz default now()`, `actor_type text` (`USER|SYSTEM|CUSTOMER`), `actor_id uuid null`, `action text` (dotted, e.g. `iam.user.created`), `entity_type text`, `entity_id text`, `before jsonb null`, `after jsonb null`, `request_id uuid null`, `ip text null`, `user_agent text null`. Indexes: `(entity_type, entity_id, occurred_at desc)`, `(actor_id, occurred_at desc)`, `(occurred_at desc)`. Migration ends with `REVOKE UPDATE, DELETE, TRUNCATE ON audit_log FROM ecommerce_app;`.
  - `redact(value: unknown): unknown` — replaces values of keys matching `/password|secret|token|totp/i` with `'[REDACTED]'`, recursively.
  - `AuditService.record(tx: Tx, entry: { action: string; entityType: string; entityId: string; before?: unknown; after?: unknown }): Promise<void>` — fills actor, request id, ip, user agent from `RequestContext`; actor defaults to `SYSTEM`.
  - `GET /admin/audit?entityType&entityId&actorId&from&to&cursor&limit` (limit ≤ 100, default 50), cursor pagination by `(occurred_at, id)`. Guarded by `@RequirePermissions('audit.read')` — the guard arrives in Task 8; until then the controller is registered in Task 8, not here.

- [ ] **Step 1: Failing tests**
  - Unit: `redact hides nested passwordHash and totpSecret but keeps email`.
  - Integration: `record writes actor and request id from context`; `runtime role cannot update audit_log` → `UPDATE audit_log SET action='x'` through `PrismaService` rejects with a permission error (Postgres code `42501`).
- [ ] **Step 2: Run** → FAIL.
- [ ] **Step 3: Implement** module, migration, repository.
- [ ] **Step 4: Run** `test` → PASS.
- [ ] **Step 5: Commit** — `feat(audit): add append-only audit log with redaction`.

### Task 5: IAM data model, permission catalog, system roles, seed

**Files:**
- Create: `src/modules/iam/domain/{permissions.ts,roles.ts,password-policy.ts}`, migration `0002_iam`, `prisma/seed.ts`
- Test: `src/modules/iam/domain/roles.spec.ts`, `password-policy.spec.ts`, `test/integration/iam/seed.int-spec.ts`

**Interfaces:**
- Produces:
  - `PERMISSIONS` (readonly tuple) and `type Permission`: `iam.manage, audit.read, settings.read, settings.write, settings.base_currency, pricing.read, pricing.write, pricing.rates.write, catalog.read, catalog.write, inventory.read, inventory.receive, inventory.adjust, orders.read, orders.manage, orders.cancel_paid, payments.read, payments.confirm, payments.refund, shipping.read, shipping.manage, shipping.deliver, customers.read, reports.read`.
  - `SYSTEM_ROLES: Record<RoleCode, readonly Permission[]>` with `RoleCode = 'SUPERADMIN'|'ADMIN'|'CATALOG_MANAGER'|'INVENTORY_MANAGER'|'ORDERS_MANAGER'|'FINANCE_MANAGER'|'DELIVERY'`:
    - `SUPERADMIN`: all.
    - `ADMIN`: all except `iam.manage`, `settings.base_currency`.
    - `CATALOG_MANAGER`: `catalog.read, catalog.write, pricing.read, pricing.write, inventory.read`.
    - `INVENTORY_MANAGER`: `inventory.read, inventory.receive, inventory.adjust, catalog.read`.
    - `ORDERS_MANAGER`: `orders.read, orders.manage, shipping.read, shipping.manage, customers.read, payments.read, inventory.read`.
    - `FINANCE_MANAGER`: `payments.read, payments.confirm, payments.refund, orders.read, orders.cancel_paid, pricing.read, pricing.rates.write, reports.read`.
    - `DELIVERY`: `shipping.deliver`.
  - Tables: `staff_user` (`id`, `email` unique, `full_name`, `password_hash`, `status` `ACTIVE|DISABLED`, `must_change_password bool`, `mfa_secret_enc bytea null`, `mfa_pending_secret_enc bytea null`, `mfa_enabled_at null`, `mfa_last_step bigint null`, `failed_login_count int`, `locked_until null`, timestamps, `version int`), `role` (`code` unique, `name`, `is_system`), `permission` (`code` pk), `role_permission`, `user_role` (composite pks, FKs `ON DELETE RESTRICT`), `staff_session` (`id`, `user_id`, `token_hash` unique, `csrf_token_hash`, `mfa_verified_at null`, `created_at`, `last_seen_at`, `expires_at`, `revoked_at null`, `ip`, `user_agent`).
  - `validatePassword(password: string, email: string): void` throws `ValidationError` unless 12–128 chars and not containing the email local part (case-insensitive).
  - `normalizeEmail(email: string): string` → trimmed, lower-cased.
  - Seed is idempotent: upserts every permission and system role with exactly its permission set.

- [ ] **Step 1: Failing tests**
  - Unit: `ADMIN lacks iam.manage and settings.base_currency`; `every role permission exists in PERMISSIONS`; `password shorter than 12 is rejected`; `password containing email local part is rejected`.
  - Integration: `seed twice yields 7 roles and 24 permissions`.
- [ ] **Step 2: Run** → FAIL.
- [ ] **Step 3: Implement** domain files, Prisma models, migration, seed.
- [ ] **Step 4: Run** → PASS.
- [ ] **Step 5: Commit** — `feat(iam): add staff user model, permission catalog and system roles`.

### Task 6: Password login, server-side sessions, logout, rate limiting

**Files:**
- Create: `src/modules/iam/application/{ports.ts,auth.service.ts}`, `infrastructure/{argon2-password-hasher.ts,prisma-user.repository.ts,prisma-session.repository.ts}`, `http/{auth.controller.ts,session.guard.ts,csrf.guard.ts,decorators.ts,dto/login.dto.ts}`, `src/shared/redis/rate-limiter.ts`, `src/cli/create-superadmin.ts`, `test/integration/helpers/{app.ts,login.ts}`
- Test: `src/shared/redis/rate-limiter.spec.ts` (with a fake Redis), `test/integration/iam/login.int-spec.ts`

**Interfaces:**
- Consumes: Task 5 tables, `normalizeEmail`, `AuditService.record`.
- Produces:
  - Ports: `PasswordHasher { hash(p: string): Promise<string>; verify(hash: string, p: string): Promise<boolean> }` (argon2id via `@node-rs/argon2`).
  - `RateLimiter.hit(key: string, limit: number, windowSeconds: number): Promise<{ allowed: boolean; retryAfterSeconds: number }>` (Redis `INCR` + `EXPIRE NX`).
  - `AuthService.login(email, password, meta) → { sessionToken: string; csrfToken: string; mfa: 'ENROLL_REQUIRED'|'VERIFY_REQUIRED' }`. Tokens: 32 random bytes base64url; DB stores SHA-256 hashes only.
  - Limits: 20 login attempts per IP per minute; 5 failures per account → `locked_until = now + 15 min` (429 while locked); failure response is identical for unknown email and wrong password (`invalid_credentials`, 401), and the hasher runs for unknown emails too (timing).
  - Cookies: `adm_sid` (HttpOnly, SameSite=Strict, Secure per `COOKIE_SECURE`, Path=/), `adm_csrf` (not HttpOnly, same flags). Mutating requests (POST/PUT/PATCH/DELETE) with a session require header `X-CSRF-Token` matching the session's CSRF hash (`CsrfGuard`), except `POST /admin/auth/login`.
  - `SessionGuard` (global for `/admin/**` except routes marked `@Public()`): loads the session by token hash; rejects revoked, expired (absolute `SESSION_ABSOLUTE_HOURS`), idle (`last_seen_at` older than `SESSION_IDLE_MINUTES`) or `DISABLED` users with 401 `session_invalid`; rejects sessions without `mfa_verified_at` with 401 `mfa_required` unless the route has `@AllowPendingMfa()`; updates `last_seen_at` at most once per minute; sets `RequestContext` actor; attaches `req.staff = { userId, sessionId, permissions: Set<Permission>, mfaVerifiedAt }`.
  - Endpoints: `POST /admin/auth/login` (`@Public`), `POST /admin/auth/logout` (`@AllowPendingMfa`, idempotent: always 204 and clears cookies), `GET /admin/auth/me` → `{ id, email, fullName, roles, permissions, mustChangePassword }`.
  - CLI: `pnpm --filter @ecommerce/api cli:create-superadmin --email <e> --name <n>` reads the password from env `INITIAL_PASSWORD`, validates policy, creates an `ACTIVE` user with role `SUPERADMIN` and `must_change_password = true`, audited as `SYSTEM`. Fails if any SUPERADMIN exists.
  - Audit actions: `iam.auth.login_succeeded`, `iam.auth.login_failed`, `iam.auth.locked`, `iam.auth.logout`.

- [ ] **Step 1: Failing tests**
  - Unit: `rate limiter blocks the 21st hit within window and reports retryAfter`.
  - Integration: `valid credentials set adm_sid and adm_csrf and require MFA`; `normalizes email on login`; `wrong password and unknown email return identical 401 bodies`; `sixth failure locks account with 429`; `me without MFA verified returns 401 mfa_required`; `expired session returns 401` (set `expires_at` in the past); `logout twice returns 204 both times`; `POST without X-CSRF-Token returns 403`.
- [ ] **Step 2: Run** → FAIL.
- [ ] **Step 3: Implement** the files above.
- [ ] **Step 4: Run** → PASS.
- [ ] **Step 5: Commit** — `feat(iam): add staff login with server-side sessions, csrf and rate limiting`.

### Task 7: Mandatory TOTP MFA, enrollment and re-authentication

**Files:**
- Create: `src/modules/iam/application/mfa.service.ts`, `infrastructure/{otplib-totp.ts,aes-gcm-secret-box.ts}`, `http/recent-mfa.guard.ts`, `http/dto/mfa-code.dto.ts`
- Modify: `auth.controller.ts`
- Test: `infrastructure/aes-gcm-secret-box.spec.ts`, `infrastructure/otplib-totp.spec.ts`, `test/integration/iam/mfa.int-spec.ts`

**Interfaces:**
- Produces:
  - `SecretBox { seal(plain: Buffer): Buffer; open(sealed: Buffer): Buffer }` — AES-256-GCM, key `MFA_ENCRYPTION_KEY`, layout `iv(12) | tag(16) | ciphertext`.
  - `Totp { generateSecret(): string; keyUri(email: string, secret: string): string; verify(code: string, secret: string, now: Date): { valid: boolean; step: number } }` — 6 digits, 30 s, accepts current and previous step only. Issuer `"Store Admin"` (renamed when the store name exists in settings, Plan 2). Use otplib 13's API; check its README at execution.
  - `POST /admin/auth/mfa/enroll` (`@AllowPendingMfa`, only if user has no MFA) → `{ otpauthUri, secret }`, stores sealed pending secret.
  - `POST /admin/auth/mfa/verify { code }` (`@AllowPendingMfa`) → confirms enrollment when pending, sets `session.mfa_verified_at = now`, rotates session and CSRF tokens (new cookies), stores `mfa_last_step`; rejects `step <= mfa_last_step` (replay) with 401 `mfa_invalid`. 5 failed codes per session → session revoked.
  - `POST /admin/auth/reauth { code }` → refreshes `mfa_verified_at`.
  - `@RequireRecentMfa(minutes = 5)` + `RecentMfaGuard` → 401 `reauth_required` when `mfa_verified_at` is older.
  - Audit: `iam.mfa.enrolled`, `iam.mfa.verified`, `iam.mfa.failed`.

- [ ] **Step 1: Failing tests**
  - Unit: `secret box round-trips and rejects tampered ciphertext`; `totp accepts previous step`; `totp rejects code two steps back`.
  - Integration: `enroll then verify grants access to me`; `same code cannot be used twice`; `enroll is refused when MFA already enabled` (409); `verify rotates adm_sid`; `fifth wrong code revokes session`; `reauth refreshes recent-MFA window` (route under test: a test-only controller registered in the integration app helper).
- [ ] **Step 2: Run** → FAIL.
- [ ] **Step 3: Implement.**
- [ ] **Step 4: Run** → PASS.
- [ ] **Step 5: Commit** — `feat(iam): require totp mfa for staff with enrollment, replay protection and reauth`.

### Task 8: RBAC guard, user management, password change, audit endpoint

**Files:**
- Create: `src/modules/iam/application/users.service.ts`, `http/{users.controller.ts,permissions.guard.ts,dto/*.ts}`, `src/modules/audit/http/audit.controller.ts`
- Modify: `iam.module.ts`, `audit.module.ts`, `auth.controller.ts`
- Test: `test/integration/iam/users.int-spec.ts`, `test/integration/iam/rbac.int-spec.ts`, `test/integration/audit/audit-endpoint.int-spec.ts`

**Interfaces:**
- Produces:
  - `@RequirePermissions(...perms: Permission[])` + `PermissionsGuard` (all listed required) → 403 `forbidden`.
  - `GET /admin/users` (`iam.manage`), `POST /admin/users { email, fullName, roleCodes, initialPassword }` (`iam.manage`, recent MFA) → user with `must_change_password = true`.
  - `PUT /admin/users/:id/roles { roleCodes }` (`iam.manage`, recent MFA): the actor must hold every permission of every target role (else 403 `privilege_escalation`); removing `SUPERADMIN` from the last active superadmin → 409 `last_superadmin`; changing your own roles → 409 `self_modification`.
  - `POST /admin/users/:id/disable` (`iam.manage`, recent MFA): revokes all sessions of that user; self or last active superadmin → 409. `POST /admin/users/:id/enable`.
  - `POST /admin/auth/password { currentPassword, newPassword }` (`@AllowPendingMfa` not allowed — needs full session): validates policy, clears `must_change_password`, revokes the user's other sessions. While `must_change_password` is true every route except `me`, `password`, `logout` returns 403 `password_change_required`.
  - `GET /admin/audit` from Task 4 registered with `audit.read`.
  - All mutations audited with before/after (`iam.user.created`, `iam.user.roles_changed`, `iam.user.disabled`, `iam.user.enabled`, `iam.user.password_changed`).

- [ ] **Step 1: Failing tests**
  - `rbac.int-spec.ts`: `catalog manager gets 403 on GET /admin/users`; `finance manager gets 403 on GET /admin/audit`; `admin cannot grant SUPERADMIN (privilege_escalation)`.
  - `users.int-spec.ts`: `superadmin creates user with roles and it is audited`; `deactivated user session is rejected`; `cannot disable last superadmin`; `cannot change own roles`; `must change password blocks other routes`; `password change revokes other sessions`; `role change without recent MFA returns reauth_required`.
  - `audit-endpoint.int-spec.ts`: `filters by entity and paginates with cursor`.
- [ ] **Step 2: Run** → FAIL.
- [ ] **Step 3: Implement.**
- [ ] **Step 4: Run** → PASS.
- [ ] **Step 5: Commit** — `feat(iam): add rbac guard, staff user management and audit query endpoint`.

### Task 9: Continuous integration

**Files:**
- Create: `.github/workflows/ci.yml`

**Interfaces:**
- Produces: workflow on `push` and `pull_request` with Node `24.14.1`, `corepack enable`, `pnpm install --frozen-lockfile`, services `postgres:17-alpine` and `redis:7-alpine`, a step running `01-roles-and-databases.sql` with `psql`, then `pnpm lint`, `pnpm typecheck`, `pnpm --filter @ecommerce/api db:deploy` (test DB), `pnpm test`.

- [ ] **Step 1: Write the workflow.**
- [ ] **Step 2: Verify locally** that the same command sequence passes from a clean clone in a scratch directory (`git clone . "$(mktemp -d)"`), with Compose running.
- [ ] **Step 3: Commit** — `ci: run lint, typecheck, unit and integration tests on github actions`.
- [ ] **Step 4:** Push and opening a pull request are the owner's decision; report the branch is ready.

---

## Spec coverage (this plan)

| Design section | Covered by |
|---|---|
| §6.1 hexagonal modules, domain errors | Tasks 2, 4–8 |
| §6.2 repo layout (api part, infra/docker) | Tasks 1–3 |
| §7.9 staff separate from customers, roles incl. DELIVERY, MFA, server-side sessions, no self-escalation, audit | Tasks 4–8 |
| §8 UUID v7, timestamps, version, RESTRICT, append-only grants | Tasks 3–5 |
| §9 CSRF, rate limiting, CORS, secrets redacted, authorization tests | Tasks 2, 4, 6–8 |
| §10 GitHub Actions CI (deploy comes in Plan 8) | Task 9 |
| §11 unit + integration on real PostgreSQL/Redis; unauthorized admin access | Tasks 2–8 |

Deferred to later plans: OpenTelemetry (Plan 8), worker/outbox (Plan 4), store settings and reference data (Plan 2), customers auth (Plan 4), frontends (Plans 6–7).
