# Cuba Store Platform — Design

- **Status:** Draft for owner approval
- **Date:** 2026-11-18
- **Scope:** Architecture and domain design for v1. No code exists yet.

## 1. Intent

A single business located in Cuba (the owner and their employees) sells its own products online.
Any person, inside or outside Cuba, can browse the store and create purchase orders. Orders are
delivered inside Cuba. The owner controls the catalog, the prices, the exchange rates, and how
every step of the store behaves, from a private back office. Employees can operate the business
only within the permissions their role grants.

This is a commercial product meant to run a real business, not a demo. Correctness of stock,
money, and order history matters more than visual polish. Visual quality is still a requirement.

## 2. Goals, non-goals, anti-goals

### Goals

1. The owner publishes products with variants, images, and per-variant stock.
2. Buyers inside and outside Cuba create orders. A buyer abroad names a recipient in Cuba.
3. The owner sets prices in a base currency and maintains exchange rates to other currencies.
   The owner can also fix an explicit price per currency for any variant.
4. The store behavior is configurable through reference data ("nomencladores"): currencies,
   exchange rates, payment methods, delivery zones, delivery modalities, reasons, and more.
5. Stock is never oversold. Two buyers can never both get the last unit.
6. Every change to stock, money, order state, permissions, and configuration is traceable.
7. Orders can be paid and confirmed **without** any international payment gateway (see 5.1).
8. The back office works on slow and unstable connections in Cuba.

### Non-goals (v1)

- A marketplace with multiple independent sellers. Rejected by the owner.
- Automatic exchange rates pulled from an external feed.
- Courier API integrations. Delivery is managed manually; interfaces are prepared.
- Microservices. The system is a modular monolith.
- Native mobile apps.

### Anti-goals (would be a failure even if it technically works)

- Selling at a loss because a price was converted with a stale or wrong rate without the owner
  knowing which rate was used.
- Stock numbers that cannot be explained by the movement history.
- An employee confirming a payment, changing stock, or granting permissions without the right
  role and without leaving a trace.
- Confirming a payment from the browser or from a return URL.
- Using third-party accounts, borrowed identities, or anything that evades geographic or
  financial restrictions.
- A back office that is unusable on a Cuban mobile connection.

## 3. Constraints

### 3.1 Mandated stack (from the owner's master brief)

Next.js (App Router), React, TypeScript strict, Tailwind CSS, shadcn/ui, React Hook Form, Zod,
TanStack Query; NestJS REST with OpenAPI; PostgreSQL with Prisma and versioned migrations;
Redis with BullMQ; Docker and Docker Compose; Nginx; Jest, Supertest, Playwright, ESLint,
Prettier; OpenTelemetry; Sentry (pending, see 3.2). No technology is added without a written
justification.

### 3.2 External constraints and verification status

The business owner is a resident of Cuba. That determines most external constraints.

| Topic | Finding | Status | Consequence |
|---|---|---|---|
| Stripe | Restricted-business policy bars parties located or resident in Cuba | Third-party summary, confirm on stripe.com | No Stripe. Assume no US-based processor until proven otherwise |
| GitHub | Cloud services, free and paid, generally available to developers in Cuba; SDN and government parties excluded; Copilot restricted | Verified on docs.github.com (2026-11-18) | Source control and CI/CD live on GitHub Actions |
| Google | Blocks Google Cloud, Developers, Play from Cuba | Reports from 2021-2022 | No runtime dependency on Google (Fonts CDN, reCAPTCHA, Maps). `next/font` self-hosts at build time, which runs in CI |
| Docker Hub | Reportedly blocks Cuba | Indirect source, unverified | Images are built in CI, pushed to GHCR, pulled by the server. Nothing is built from Cuba |
| Cloudflare | Some free and paid services authorized for Cuba, government parties excluded | Cloudflare blog | CDN, DNS, and R2 are possible; must be confirmed for an account held by a Cuban resident |
| Hosting (EU VPS) | Unknown whether providers accept a Cuban resident as customer | **Unverified — Fase 0 blocker** | Must be confirmed in writing before contracting. The account must be in the owner's own name |
| Sentry, transactional email | US companies, likely same sanctions exposure | Unverified | Both sit behind ports. Fallback: OpenTelemetry to a self-hosted backend; SMTP to a provider confirmed to serve the owner |
| Cuba MINCIN Res. 93/2023 | Merchants must accept Transfermóvil and EnZona | Independent press; check Gaceta Oficial | Both are first-class payment methods in v1 (manual confirmation) |
| Cuba Res. 35/2020 (MINCOM) | Regulates virtual store design, including payment method selection | Official PDF found, not yet reviewed in full | Legal review item |
| Transfermóvil / EnZona merchant APIs | Unknown whether a merchant API exists and is available to this business | Unverified | v1 does not depend on them; adapters can be added later |
| Tropipay | No Banco de España authorization record found | Unverified | Not used |

### 3.3 Connectivity

Admins and many recipients use Cuban mobile networks: high latency, low bandwidth, frequent
drops, data cost. Back office pages must be server-rendered, low-JavaScript, paginated, and
tolerant of retries (idempotent mutations). Images are compressed in the browser before upload.

## 4. Approaches

| Approach | Summary | Trade-off | Decision |
|---|---|---|---|
| **A. Order decoupled from payment, manual confirmation, adapters later** | Order reserves stock; buyer pays by a configured method and submits a reference and proof; a finance employee confirms. Gateways plug in later as adapters | Sellable now with zero external payment dependency; needs a human to confirm payments | **Chosen** |
| B. Wait for an international gateway | Launch only when a legal gateway for a Cuban-resident merchant exists | Fully automatic, but blocked by legal and commercial factors outside our control | Rejected: blocks launch indefinitely |
| C. Marketplace | Multiple sellers, commissions, payouts | Several times larger; different legal exposure | Rejected by the owner |

## 5. Owner decisions (as decided)

1. Single business: the owner's store and employees.
2. Buyers anywhere can create orders.
3. Order and payment are decoupled; manual confirmation is the v1 payment path.
4. The owner defines prices and exchange rates. Prices can be shown and paid in different
   currencies.
5. The store's steps are configured through reference data (nomencladores) in the back office.

### 5.1 Payment model

- A payment method is a configurable record (see 7.2) with instructions, accepted currencies,
  whether proof is required, and the provider that handles it.
- v1 ships one provider, `manual`. Later providers (`transfermovil`, `enzona`, an international
  gateway) implement the same port.
- A payment method has a `timing`: `PREPAID` (default) or `ON_DELIVERY` (cash on delivery).
  - `PREPAID`: the order must reach `PAID` before preparation.
  - `ON_DELIVERY`: staff may move the order from `PENDING_PAYMENT` straight to `PREPARING`
    (stock moves from reserved to committed at that moment, so the reservation no longer
    expires). The courier, a staff user with the Delivery role, records the cash on delivery;
    finance can also record it afterwards. `SHIPPED → DELIVERED` is only allowed once that
    payment is recorded, and both happen in the same transaction when the courier confirms.
- **Who pays from abroad (owner decision):** buyers outside Cuba pay by transfer to an account
  abroad dedicated to the business. This is a `PREPAID` manual method (e.g. `foreign_transfer`)
  whose accepted currencies match that account (e.g. USD, EUR) and whose instructions (holder,
  account details, reference format "order number in the transfer concept") are configured in
  the back office, never hard-coded. The buyer submits the transfer reference and proof; finance
  confirms after seeing the funds in the account statement. The daily reconciliation report
  (7.6) lists confirmed payments for this method so finance can match them against the statement.
  The account must belong to the owner or the owner's business (see 14.6, legal review).
- The design also supports a payer different from the buyer: anyone holding the order's signed
  tracking link can submit a payment reference and proof (e.g. a relative in Cuba paying by
  Transfermóvil).

## 6. Architecture

### 6.1 Style

Modular monolith. Each module follows a hexagonal layout:

```
modules/<name>/
  domain/          entities, value objects, domain services, domain errors — no framework imports
  application/     use cases (commands/queries), ports (interfaces)
  infrastructure/  Prisma repositories, provider adapters, BullMQ processors
  http/            NestJS controllers, DTOs (class-validator), OpenAPI decorators
```

Rules:
- A module never reads or writes another module's tables. It calls the other module's
  application service through an exported interface, or reacts to a domain event.
- Cross-module consistency that must be atomic (e.g. order creation + stock reservation) runs
  inside one database transaction coordinated by the use case that owns the flow, through
  repository methods that accept a transaction handle.
- Domain events are written to an `outbox` table in the same transaction and published to BullMQ
  by a relay. Redis is never the source of truth.

### 6.2 Repository layout

pnpm workspaces. No additional build orchestrator until build times justify one.

```
apps/
  api/        NestJS REST API (public + admin endpoints, separate route prefixes and guards)
  worker/     NestJS application context running BullMQ processors and the outbox relay
  store/      Next.js public store (buyers)
  admin/      Next.js back office (employees), separate domain, separate cookies, stricter CSP
packages/
  contracts/  Zod schemas and TypeScript types shared by API and frontends
  config/     shared ESLint, Prettier, TypeScript configs
infra/
  docker/     Dockerfiles, docker-compose for dev and test
  nginx/      reverse proxy config
  backup/     backup and restore scripts
docs/
  architecture/, adr/, runbooks/, manual-admin/
```

`api` and `worker` share the same module code; they are two entry points of one NestJS codebase
(`apps/api/src/main.ts`, `apps/api/src/worker.ts`) packaged as one image with two commands.
The admin is a separate Next.js app so its bundle stays small for Cuban connections and its
session cookies are isolated from the public store.

### 6.3 Modules

`catalog`, `pricing` (currencies, exchange rates, price resolution), `inventory`, `cart`,
`checkout`, `orders`, `payments`, `shipping`, `customers`, `iam` (staff users, roles,
permissions, MFA), `audit`, `settings` (store settings and generic reference data),
`notifications`, `media`.

## 7. Domain design

### 7.1 Pricing and currencies

- **Currency** (reference data): `code` (ISO 4217 where it exists; custom codes allowed, e.g.
  `MLC`), name, symbol, `decimals`, `rounding_step` (e.g. `1` for CUP, `0.01` for USD),
  `rounding_mode` (half-up, up), `active`, `accepted_for_checkout`.
- **Base currency:** one, set in store settings. Changing it requires the `settings.base_currency`
  permission and is blocked while there are orders in non-terminal states.
- **ExchangeRate:** `from` is always the base currency; `to_currency`, `rate NUMERIC(20,10)`,
  `effective_from`, `created_by`, `reason`. Append-only: a new rate is a new row. The current
  rate is the latest row with `effective_from <= now()`. Future-dated rates are allowed.
- **VariantPrice:** `variant_id`, `currency`, `amount NUMERIC(19,4)`, `kind` = `BASE` | `OVERRIDE`.
  Every sellable variant has exactly one `BASE` price in the base currency
  (`UNIQUE(variant_id, currency)`).
- **Price resolution** (one domain service, used by cart, checkout, and display):
  1. If an `OVERRIDE` exists for the requested currency, use it.
  2. Otherwise `base_amount × current_rate`, rounded with the currency's rule.
  3. If no rate exists for that currency, the variant is not sellable in that currency.
- **Freezing:** at order creation the server recomputes every line and stores, per order:
  currency, `exchange_rate_id` used (null when all lines were overrides or base), unit price per
  line, totals, and the base-currency equivalent of the total for reporting. After creation the
  order's amounts never change; payments must match the order currency and amount.
- Shipping rates and coupon fixed amounts follow the same rules (base amount + optional
  per-currency override).

### 7.2 Reference data (nomencladores)

All reference data shares these rules: immutable `code`, editable display fields, `active` flag
(deactivate, never delete when referenced; FKs use `ON DELETE RESTRICT`), every change written
to the audit log, permission-gated per type.

| Reference data | Key fields | Used by |
|---|---|---|
| Currencies, exchange rates | see 7.1 | pricing |
| Payment methods | code, name, instructions (rich text per locale), accepted currencies, provider (`manual`, later adapters), proof required, confirmation window (hours), display order | checkout, payments |
| Provinces, municipalities | Cuba political-administrative division, seeded | shipping, addresses |
| Delivery zones | name, set of municipalities | shipping |
| Delivery modalities | code (`home`, `pickup`), name, requires address | shipping |
| Delivery rates | zone × modality × currency, amount, estimated days | checkout |
| Pickup points | name, municipality, address, hours | shipping |
| Warehouses | code, name, municipality, priority, active | inventory |
| Categories | tree, slug, SEO fields | catalog |
| Attribute types and values | e.g. size → S/M/L, color → red | catalog variants |
| Units of measure | code, name | catalog |
| Reasons | type (cancellation, stock adjustment, return, payment rejection, delivery incident), code, name | orders, inventory, payments, shipping |
| Identity document types | e.g. carné de identidad, passport | recipients |
| Countries | ISO 3166, seeded | buyers |

**Store settings** (single record, versioned in audit): base currency, reservation TTL for
unpaid orders, guest checkout on/off, required buyer fields, terms and policies version,
low-stock default threshold, four-eyes payment confirmation on/off, store contact data.

**What is not configurable:** the order state machine and inventory rules. They are code
invariants. Configuration decides which methods, zones, and options are offered, not whether
an order can skip payment or stock can go negative. Display labels of states are translatable.

### 7.3 Inventory

- **InventoryBalance** per (variant, warehouse): `on_hand`, `reserved`, `committed`, `min_stock`.
  `available = on_hand − reserved − committed`. DB `CHECK` constraints: all ≥ 0 and
  `reserved + committed <= on_hand`.
- **Reserve** (single atomic statement inside the order-creation transaction):

  ```sql
  UPDATE inventory_balance
     SET reserved = reserved + $qty, version = version + 1
   WHERE variant_id = $v AND warehouse_id = $w
     AND on_hand - reserved - committed >= $qty;
  ```

  Zero rows updated means not enough stock: the whole order transaction rolls back.
- **Warehouse allocation (v1):** for each line, the active warehouse with the highest priority
  that can cover the full quantity. Lines are not split across warehouses in v1.
- **StockReservation:** order line, warehouse, qty, `expires_at`, status
  (`ACTIVE`, `COMMITTED`, `RELEASED`).
- **Paid:** `reserved −= qty`, `committed += qty`, reservation `COMMITTED`.
- **Shipped:** `committed −= qty`, `on_hand −= qty`, ledger movement `SALE_OUT`.
- **Expiry:** a BullMQ delayed job at `expires_at`, plus a periodic sweeper (every minute) that
  releases any `ACTIVE` reservation past `expires_at`. Both are idempotent.
- **While payment proof is under review** (`PAYMENT_PROCESSING`), the reservation does not
  expire. An alert fires if review exceeds the payment method's confirmation window.
- **InventoryMovement** is append-only (no UPDATE/DELETE grants for the app role): types
  `RECEIPT`, `ADJUSTMENT_IN`, `ADJUSTMENT_OUT`, `SALE_OUT`, `RETURN_IN`, `TRANSFER_OUT`,
  `TRANSFER_IN`. Each has qty, reason, actor, reference (order, receipt, transfer). Corrections
  are new movements. A nightly job checks that `on_hand` equals the sum of movements and alerts
  on any mismatch.
- Manual adjustments require `inventory.adjust`, a reason from reference data, and are audited.

### 7.4 Cart and checkout

- Cart is persistent: by customer account, or by an anonymous cart token cookie for guests.
  Cart lines hold variant and quantity only. Prices are never stored from the client.
- Buyer picks a display/checkout currency among `accepted_for_checkout` currencies.
- Checkout collects: buyer (name, email, phone, country of residence), recipient in Cuba (name,
  phone, identity document type and number if the owner requires it, municipality, address or
  pickup point), delivery modality, payment method (filtered by currency), coupon, acceptance of
  terms (version stored).
- **Place order** (one transaction): re-resolve prices, validate coupon, compute shipping,
  reserve stock, create order `PENDING_PAYMENT`, write outbox event. Idempotency key header
  required so a retried request on a bad connection never creates two orders.

**Coupons (v1):** percentage or fixed amount (fixed amounts follow the pricing rules of 7.1),
validity dates, minimum order amount, maximum total uses, maximum uses per customer or email,
optional restriction to categories. One coupon per order, no stacking. The discount is prorated
across the eligible lines and stored per line, so partial refunds are exact. Usage is counted
inside the order transaction (conditional update, same technique as stock). Automatic promotions
(without a code) are deferred to a later phase; the `Promotion` entity is not built in v1.

**Taxes:** prices are tax-inclusive by default. Order lines and orders still store
`tax_amount` and `tax_rate_code` columns (zero by default) so a later legal requirement does not
leave historical orders without that data. Invoice data is pending the legal review (14.6).

**Search:** PostgreSQL full-text search (with `unaccent`) over product name, description, SKU,
and category, plus filters by category, attributes, price range, and availability.

### 7.5 Orders

States: `DRAFT`, `PENDING_PAYMENT`, `PAYMENT_PROCESSING`, `PAID`, `PREPARING`, `READY`,
`SHIPPED`, `DELIVERED`, `CANCELLED`, `REFUNDED`, `PAYMENT_REJECTED`.

| From | To | Trigger |
|---|---|---|
| DRAFT | PENDING_PAYMENT | staff finalizes an assisted order (stock reserved here) |
| DRAFT | CANCELLED | staff discards |
| PENDING_PAYMENT | PAYMENT_PROCESSING | buyer submits payment reference/proof, or gateway reports pending |
| PENDING_PAYMENT | PAID | verified gateway notification, or staff confirmation |
| PENDING_PAYMENT | CANCELLED | reservation expired, buyer cancels, or staff cancels |
| PENDING_PAYMENT | PREPARING | staff, only when the payment method's timing is `ON_DELIVERY` (stock committed) |
| PAYMENT_PROCESSING | PAID | staff confirms, or verified gateway notification |
| PAYMENT_PROCESSING | PAYMENT_REJECTED | staff rejects with reason, or gateway reports failure |
| PAYMENT_REJECTED | PAYMENT_PROCESSING | buyer submits a new proof within the window |
| PAYMENT_REJECTED | CANCELLED | window elapsed or staff cancels (reservation released) |
| PAID | PREPARING | staff starts preparation |
| PREPARING | READY | staff |
| READY | SHIPPED | staff assigns delivery and dispatches |
| SHIPPED | DELIVERED | delivery confirmed; requires a confirmed payment covering the order total |
| SHIPPED | CANCELLED | delivery failed and goods returned to the store (shipment `RETURNED`); stock re-enters through a `RETURN_IN` movement when received; refund flow if paid |
| PAID, PREPARING, READY | CANCELLED | staff with `orders.cancel_paid`; triggers refund flow and stock return |
| CANCELLED (after paid), DELIVERED | REFUNDED | refunds recorded for the full paid amount |

**Returns and partial refunds (v1):** after `DELIVERED`, staff can register a `Return` with
specific lines and quantities. Each returned quantity creates a `RETURN_IN` movement into a chosen
warehouse (or an `ADJUSTMENT_OUT` reason if the goods are damaged). Refunds can be partial; the
sum of refunds can never exceed the confirmed payments. The order moves to `REFUNDED` only when
refunds cover the full paid amount; partial refunds leave it `DELIVERED` and show in its history.

`DRAFT` exists for assisted orders created by employees (e.g. taken by phone). Online orders
start at `PENDING_PAYMENT`.

Every transition writes `OrderStatusHistory`: from, to, actor (user or system), reason code,
note, operation reference, timestamp. Invalid transitions throw a domain error and are covered
by tests that enumerate the whole matrix.

**Late payment on an expired order:** if a payment arrives for a `CANCELLED` (expired) order,
the payment is recorded as `UNMATCHED`, finance is alerted, and finance either reinstates the
order (only if stock can be reserved again) or records a refund.

### 7.6 Payments

Port (`application/ports`):

```ts
interface PaymentProvider {
  readonly key: string;
  createIntent(order: OrderPaymentView): Promise<PaymentIntentResult>;   // instructions or redirect
  parseNotification(raw: RawNotification): Promise<VerifiedPaymentEvent>; // throws on bad signature
  refund(payment: PaymentView, amount: Money, reason: string): Promise<RefundResult>;
  fetchStatus(externalId: string): Promise<ProviderPaymentStatus>;       // reconciliation
}
```

- **Payment:** order, method, provider, amount, currency, status
  (`PENDING`, `SUBMITTED`, `CONFIRMED`, `REJECTED`, `UNMATCHED`, `REFUNDED`), external
  reference, proof file, confirmed_by, confirmed_at.
- `UNIQUE(payment_method_id, external_reference)`: the same Transfermóvil or bank transaction
  reference can never confirm two orders.
- **Manual confirmation:** permission `payments.confirm`; recent re-authentication with MFA
  required; amount and currency entered by the confirmer must equal the order's; optional
  four-eyes (a second user approves) controlled by settings.
- **Gateway notifications (future adapters):** raw body kept for signature verification;
  `PaymentEvent` stored with `UNIQUE(provider, external_event_id)` before processing (replay and
  duplicate protection); amount, currency, and order reference validated; state change applied in
  one transaction; the browser return URL is never treated as proof.
- **Refunds:** `PaymentRefund` with amount, reason, actor, external reference. Manual refunds are
  recorded by finance.
- **Reconciliation:** daily report of confirmed payments per method and currency vs. orders,
  for finance to match against Transfermóvil/EnZona/bank statements. Adapter-based automatic
  reconciliation later.
- No card data is ever handled by this system.

### 7.7 Shipping

- **Shipment:** order, modality, zone, recipient snapshot, assigned courier (staff user or
  free-text external courier), estimated date, status (`PENDING`, `ASSIGNED`, `IN_TRANSIT`,
  `DELIVERED`, `INCIDENT`, `RETURNED`), proof of delivery (photo or signature name).
- **ShipmentEvent:** append-only timeline with actor and note.
- `CourierProvider` port prepared; v1 uses manual updates only.
- Buyer tracking page shows the shipment timeline; guests access it with a signed tracking link.

### 7.8 Customers

- Accounts with email + password (argon2id), email verification, password reset with
  single-use, short-lived, hashed tokens. Guest checkout allowed when enabled in settings.
- Saved recipients (people in Cuba) and addresses.
- Order history and tracking. Consent records (terms version, marketing opt-in) with timestamps.

### 7.9 IAM, RBAC, audit

- Staff users are separate from customers (`User` vs `Customer`), with separate login endpoints
  and session cookies.
- Roles: Superadministrator, Administrator, Catalog manager, Inventory manager, Orders manager,
  Finance manager, Delivery (sees only the shipments assigned to them; updates shipment status;
  records cash for `ON_DELIVERY` orders). Delivery is added to the brief's roles because cash on
  delivery and manual shipping need a courier who is not finance. Permissions are fine-grained codes (`catalog.write`, `pricing.rates.write`,
  `inventory.adjust`, `payments.confirm`, `payments.refund`, `orders.cancel_paid`,
  `iam.manage`, `settings.write`, ...). Roles map to permission sets; only
  `iam.manage` can change them; nobody can grant a permission they don't hold.
- MFA (TOTP) mandatory for all staff. Server-side sessions stored in the database (revocable),
  `HttpOnly`, `Secure`, `SameSite=Strict` cookies on the admin domain.
- **AuditLog:** actor, action, entity, entity id, before/after JSON diff, IP, user agent,
  timestamp. Append-only. Written in the same transaction as the change.

### 7.10 Notifications

`NotificationChannel` port. v1 ships email only (provider pending, 3.2) plus the order tracking
page, which is the reliable channel for recipients in Cuba. Events: order created (buyer), payment
confirmed or rejected (buyer), shipment dispatched and delivered (buyer), payment review overdue
and low stock (staff). Each notification is stored with status and retried by the worker; a failed
email never blocks an order. SMS or WhatsApp providers are a later adapter, subject to the same
eligibility check as email.

### 7.11 Data import

Phase 3 includes a CSV import in the admin for products/variants with base prices and for opening
stock per warehouse (opening stock becomes `RECEIPT` movements with reason "opening balance").
Imports are validated in full before anything is written, and run as one transaction per file.

### 7.12 Media

Images uploaded from the admin are resized and compressed in the browser (canvas, no extra
library), sent to the API, validated (type sniffing, size limits), processed with `sharp` into
responsive sizes, stored in S3-compatible storage (provider pending, see 3.2), and served through
the CDN or Nginx.

## 8. Data model notes

Entities: User, Role, Permission, RolePermission, UserRole, Session, Customer, Recipient,
Address, Country, Province, Municipality, Product, ProductVariant, VariantAttributeValue,
AttributeType, AttributeValue, Category, ProductImage, UnitOfMeasure, Currency, ExchangeRate,
VariantPrice, Warehouse, InventoryBalance, InventoryMovement, StockReservation, Cart, CartItem,
Order, OrderItem, OrderStatusHistory, Payment, PaymentMethod, PaymentMethodCurrency,
PaymentEvent, PaymentRefund, PaymentReconciliation, Coupon, Promotion, DeliveryZone,
DeliveryZoneMunicipality, DeliveryModality, DeliveryRate, PickupPoint, Shipment, ShipmentEvent,
Reason, IdentityDocumentType, StoreSettings, AuditLog, Outbox, IdempotencyKey.

- Primary keys: UUID v7 (time-ordered).
- Money: `NUMERIC(19,4)` plus currency code. Rates: `NUMERIC(20,10)`. Quantities: `INTEGER`.
  Never floating point.
- All tables: `created_at`, `updated_at`; mutable business tables also `created_by`,
  `updated_by`, and `version` for optimistic locking on admin edits.
- SKU unique per variant. Product slug unique.
- `ON DELETE RESTRICT` by default; `CASCADE` only for pure child rows (cart items of a cart).
- Order and order items store snapshots (product name, SKU, unit price, currency, rate id) so
  later catalog edits never change history.
- Append-only tables (`inventory_movement`, `order_status_history`, `payment_event`,
  `audit_log`, `shipment_event`, `exchange_rate`) have no UPDATE/DELETE grants for the
  application database role.
- Raw SQL (via Prisma `$executeRaw`) is allowed only for the atomic stock statements and CHECK
  constraints that Prisma cannot express; these live in migrations and repository code with tests.

## 9. Security

OWASP ASVS level 2 as the verification reference. HTTPS only (HSTS); security headers and CSP
via Nginx and Next.js; CSRF protection on cookie-authenticated mutations (same-site cookies plus
a double-submit token); input validation with DTOs and Zod; Prisma parameterized queries;
rate limiting on auth, checkout, and proof upload (Redis-backed); restrictive CORS (store and
admin origins only); resource-level authorization checks (a customer can only read their own
orders); secrets in environment files outside the repo, never logged; structured logs with a
redaction list (passwords, tokens, document numbers); encrypted off-site backups; authorization
and data-isolation tests for every admin endpoint.

## 10. Infrastructure

```
[Cloudflare DNS/CDN — if confirmed for the owner]
            |
     EU VPS (Docker Compose)
     Nginx ── store (Next.js)
           ── admin (Next.js)
           ── api (NestJS)
     worker (NestJS + BullMQ)
     PostgreSQL 17 ── WAL archiving + nightly base backup → off-site S3-compatible (encrypted)
     Redis (AOF on)

GitHub Actions: lint → test → build images → push GHCR → deploy over SSH (pull + migrate + restart)
```

Environments: local (Compose), staging (same VPS or a smaller one, separate database), production.
Restores are tested monthly by restoring into a scratch database and running integrity checks.
Certificates via Let's Encrypt (or Cloudflare origin certificates if Cloudflare is used).
OpenTelemetry traces and metrics; error tracking provider pending (3.2).

## 11. Testing strategy

- **Unit (Jest):** domain rules: price resolution and rounding, state machine matrix, reservation
  math, permission checks.
- **Integration (Jest + Supertest + real PostgreSQL and Redis in Docker Compose):** every use case
  through the HTTP API.
- **Concurrency:** N parallel orders for the last unit → exactly one succeeds, stock stays
  consistent.
- **Critical cases from the brief:** last unit; simultaneous buyers; confirmed, rejected, and
  pending payment; duplicate notification; invalid signature (adapter test with the simulated
  provider); amount mismatch; wrong currency; reservation expiry; cancellation; refund;
  merchandise return; PostgreSQL outage and retry; unauthorized admin access; unauthorized stock
  change; backup restore.
- **Pricing cases:** rate change between cart and order; override vs converted price; missing rate
  makes the variant unsellable in that currency; order amounts unchanged after a later rate change.
- **E2E (Playwright):** buyer abroad orders in USD for a recipient in Cuba and pays by foreign
  transfer; finance confirms payment;
  staff prepares and delivers; owner changes a rate and sees new prices.
- A module is done only when its functional and security tests pass, not when it compiles.

## 12. Phases

| Phase | Content | Blocked by |
|---|---|---|
| 0 | Viability matrix with evidence; hosting, storage, email, error tracking confirmed for the owner; Cuban legal review | Owner and providers |
| 1 | Architecture docs, ADRs, ER model, OpenAPI contract skeleton | — |
| 2 | Monorepo, Docker, CI/CD, database, migrations, staff auth with MFA, RBAC, audit, settings and reference data | Hosting for deploy (local work can start) |
| 3 | Catalog, pricing and currencies, inventory, cart, checkout, orders, admin screens | — |
| 4 | Payments: manual provider in production; simulated gateway adapter for tests | Real gateways: legal eligibility |
| 5 | Shipping, dashboard, reports, reconciliation | — |
| 6 | Load, security, concurrency, restore, and connectivity tests from Cuban networks; runbooks; admin manual | Access to Cuban networks for testing |

## 13. Left to the builder

- Exact Prisma schema naming, indexes beyond those listed, and migration ordering.
- Admin UI layout details within the stated constraints (server-rendered, low JS, tables with
  filters).
- Visual identity of the public store (own identity, no copied brands).
- BullMQ queue names, retry counts, and backoff values.
- Choice of image sizes and caching headers.

## 14. Open items (need the owner or external confirmation)

1. Hosting provider that accepts the owner (Cuban resident) as a customer, in writing.
2. Object storage and CDN provider for the owner (Cloudflare R2 or alternative).
3. Transactional email provider that serves the owner.
4. Error tracking: Sentry eligibility or a self-hosted alternative.
5. Whether Transfermóvil and EnZona offer merchant APIs available to this business.
6. Legal review in Cuba (MINCIN/MINCOM rules for online stores, invoicing, data protection) and
   for foreign buyers' payments to a Cuban-resident business.
7. Whether the recipient's identity document is required for delivery.
8. Resolved: buyers abroad pay by transfer to the business's dedicated account abroad (5.1).
   Pending only the legal confirmation that this collection path is compliant in both
   jurisdictions (14.6).

## 15. Calls made by the designer (owner may override)

- Cash on delivery as an `ON_DELIVERY` payment timing, with a Delivery role (5.1, 7.9).
- Failed deliveries cancel the order and return stock; partial returns and partial refunds in v1 (7.5).
- Coupons only, no stacking; automatic promotions deferred (7.4).
- Tax-inclusive prices with zeroed tax columns stored (7.4).
- Email plus tracking page as v1 notifications; SMS/WhatsApp later (7.10).
- CSV import for catalog and opening stock (7.11).
- PostgreSQL full-text search (7.4).
- Phases run sequentially for a small team; four-eyes approval, multi-warehouse allocation, and
  reconciliation reports can be deferred if an earlier launch is needed.
