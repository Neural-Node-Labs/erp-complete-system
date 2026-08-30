# ERP Platform — Accounting/Finance Module

Foundation for a modular ERP system, built to the enterprise layering and
tech-stack you specified:

- **Frontend:** Vue 3 + Vuetify (Material Design), routed multi-page SPA
- **Backend:** Java 21, Spring WebFlux (reactive), layered
  `entity / model / repository / service / controller / common`
- **Auth:** a **custom OAuth2/OIDC Authorization Server** (Spring Authorization
  Server), its own container, backed by Postgres
- **Database:** PostgreSQL 16, two logical databases (`erp_auth`, `erp_core`)
- **First module:** Accounting/Finance — Chart of Accounts, double-entry
  Journal Entries (draft → post → void), and live financial reports (Trial
  Balance, Income Statement, Balance Sheet) computed directly from posted
  ledger activity
- **Cross-cutting foundation:** a shared `Party` model (customers/vendors/
  employees, one table for every future module) and an atomic
  `DocumentNumberingService` (JE-2026-000123 style sequences, safe under
  concurrency) that Journal Entries already use and Sales/Procurement will
  reuse
- **Second module: Inventory** — items with SKU/cost/sale price/reorder
  point, and stock movements (Receipt / Issue / Adjustment) that keep
  quantity-on-hand in sync the same way Journal Entries keep account
  balances in sync, with the same audit trail and auto-numbering
- **Audit:** every create/update/deactivate/post/void across every module is
  written to an append-only `audit_log` table, viewable only by ADMIN

Everything runs in Docker containers via one `docker-compose.yml`.

## Project layout

```
erp-system/
├── docker-compose.yml
├── infra/postgres/init-multi-db.sh       # creates erp_auth + erp_core on first boot
├── auth-server/                          # OAuth2/OIDC Authorization Server (port 9000)
│   └── src/main/java/com/erp/authserver/
│       ├── config/    AuthorizationServerConfig, DefaultSecurityConfig
│       ├── entity/    AppUser
│       ├── repository/AppUserRepository
│       └── service/   JpaUserDetailsService
├── backend/                              # Reactive ERP core API (port 8080)
│   └── src/main/java/com/erp/backend/
│       ├── common/    ApiResponse, exceptions, GlobalExceptionHandler, CurrentUser
│       ├── config/    SecurityConfig (JWT validation, CORS), R2dbcConfig
│       ├── entity/    account/, journal/, audit/, party/, numbering/, inventory/  (per-module packages)
│       ├── model/     account/, journal/, audit/, party/, inventory/  (request/response DTOs)
│       ├── repository/AccountRepository, JournalEntryRepository, JournalLineRepository, AuditLogRepository, PartyRepository, InventoryItemRepository, StockMovementRepository
│       ├── service/    AccountService(+Impl), JournalEntryService(+Impl), ReportService(+Impl), AuditLogService(+Impl), PartyService(+Impl), DocumentNumberingService(+Impl), InventoryItemService(+Impl), StockMovementService(+Impl)
│       └── controller/AccountController, JournalEntryController, ReportController, AuditLogController, PartyController, InventoryItemController, StockMovementController
└── frontend/                             # Vue 3 + Vuetify SPA (port 5173)
    └── src/
        ├── router/      route guards (auth required; /admin/* requires ADMIN)
        ├── stores/       Pinia auth store
        ├── services/     authService (OIDC/PKCE), api (axios + JWT header)
        ├── components/layout/ AppLayout (nav, top bar)
        └── pages/         Login, Callback, Dashboard, core/PartiesPage, accounting/* (Accounts, Journal Entries, Reports), inventory/* (Items, Movements), admin/AuditLogPage
```

Every module going forward (Inventory, Sales, HR, Procurement, ...) follows the
same shape: a new `entity/<module>`, `model/<module>`, one repository per
table, one service per aggregate, one controller — and every mutating service
method calls `auditLogService.log*()` the same way Accounting does.

## Running it

```bash
cd erp-system
docker compose up --build
```

This starts, in order: Postgres → Authorization Server → Backend → Frontend.

| Service        | URL                          |
|----------------|-------------------------------|
| Frontend       | http://localhost:5173         |
| Backend API    | http://localhost:8080         |
| Auth Server    | http://localhost:9000          |
| OIDC discovery | http://localhost:9000/.well-known/openid-configuration |
| Postgres       | localhost:5432 (db `erp_auth`, `erp_core`, user `erp` / `erp_password`) |

### Default login

A seed admin user is created on first boot (Flyway migration in `auth-server`):

- **Username:** `admin`
- **Password:** `Admin@12345`
- **Roles:** `ADMIN`, `ACCOUNTANT`

Change this password (or delete/replace the seed row) before any real
deployment — it's a local/dev convenience only.

## How authentication works end-to-end

1. Frontend redirects the browser to `auth-server`'s `/oauth2/authorize`
   (Authorization Code + PKCE — the frontend is a public client, no secret).
2. User logs in on the auth-server's own login page (Postgres-backed).
3. Auth-server redirects back to `http://localhost:5173/callback` with a code;
   the frontend exchanges it for tokens directly with the auth-server.
4. The frontend attaches the resulting JWT access token as
   `Authorization: Bearer ...` on every backend API call.
5. The backend (`erp-backend`) is a pure **resource server**: it validates the
   JWT's signature against the auth-server's published JWKs
   (`issuer-uri`), reads the `roles` claim baked into the token by
   `AuthorizationServerConfig.jwtTokenCustomizer()`, and enforces
   `@PreAuthorize` rules per endpoint — it never talks to Postgres to check
   who you are.

## Audit logging

Every mutating operation (create, update, deactivate, post, void) in every
service calls into `AuditLogService`, which writes an immutable row to
`audit_log` containing: module, entity type/id, action, who did it, and a
JSON snapshot of the record before and after the change. Reads of this log
are exposed at `GET /api/audit-logs` and `GET /api/audit-logs/entity/{type}/{id}`,
both restricted to the `ADMIN` role at two layers (path-level in
`SecurityConfig` and `@PreAuthorize` on the controller) so it fails closed
even if one layer is misconfigured.

## Cross-cutting foundation: Party and Document Numbering

Before adding another module, two pieces went in that every future module
will share rather than reinvent:

- **`Party`** (`party` table) — one shared record for anyone the ERP deals
  with, differentiated by `partyType` (`CUSTOMER`, `VENDOR`, `EMPLOYEE`,
  `OTHER`). Sales will filter this table for customers, Procurement for
  vendors, HR for employees — same audit trail, same address/contact
  fields, no drift between three near-identical tables.
- **`DocumentNumberingService`** — allocates human-readable document numbers
  (`JE-2026-000123`, and later `SO-2026-...`, `PO-2026-...`, `INV-2026-...`)
  via a single atomic `INSERT ... ON CONFLICT ... DO UPDATE ... RETURNING`
  against a `document_sequence` table. This is safe across concurrent
  requests and multiple backend replicas because Postgres itself serializes
  the increment — no in-process counter that breaks the moment you scale
  horizontally. Journal Entries already use it: leave "reference" blank
  when creating one and it's auto-numbered.

## Testing

Unit tests exist for the highest-risk business logic — the code where a
bug would mean silently wrong money or silently wrong stock:

- `AccountServiceImplTest` — duplicate code rejection, unknown parent rejection
- `JournalEntryServiceImplTest` — unbalanced entries rejected, a line can't
  have both a debit and a credit, posting a balanced entry updates both
  accounts' balances correctly (asset up on debit, revenue up on credit),
  double-posting is rejected
- `StockMovementServiceImplTest` — issuing more than is on hand is rejected,
  a non-positive issue quantity is rejected, an adjustment that would drive
  quantity negative is rejected

Run them with `cd backend && mvn test`.

**Not covered yet** (the honest gap): controller-layer tests (`@WebFluxTest`),
repository/integration tests against a real Postgres (e.g. via Testcontainers),
security/`@PreAuthorize` tests confirming role enforcement actually blocks
what it should, auth-server tests, and any frontend tests at all. Given how
much of this system's correctness lives in "does the ADMIN-only audit log
really reject a non-admin" and "does a restart really preserve OAuth2
clients," those are the next tests worth writing before this goes anywhere
near production traffic.

## Auth-server persistence

Everything the Authorization Server needs to survive a restart (or run as
more than one replica) is now in Postgres, not memory:

- **Registered clients** — `JdbcRegisteredClientRepository` against the
  standard Spring Authorization Server schema (`V2__oauth2_persistence_schema.sql`,
  adapted for Postgres). `ClientSeeder` inserts `erp-frontend` and
  `erp-backend-service` once, on first boot only — it checks
  `findByClientId` before inserting, so it's safe to restart repeatedly.
- **Issued authorizations/tokens and consents** — `JdbcOAuth2AuthorizationService`
  and `JdbcOAuth2AuthorizationConsentService`, same schema file.
- **JWT signing key** — `PersistentJwkSourceConfig` generates one RSA
  key pair on the very first boot, stores it in the `signing_key` table,
  and loads that same key on every boot after. Before this, every restart
  generated a brand-new random key, which silently invalidated every
  access token already issued (the resource server validates signatures
  against this server's published JWKs, which would have rotated to an
  unrelated key).

## Financial reports

`GET /api/accounting/reports/trial-balance?asOf=YYYY-MM-DD`,
`/income-statement?from=...&to=...`, and `/balance-sheet?asOf=...` are all
computed live from `journal_line` rows on POSTED entries - not from the
denormalized `Account.balance` column - so a report for any past date is
always accurate. There are no period-close/closing entries yet, so the
Balance Sheet shows current-period net income as its own unclosed line
rather than assuming it has already been swept into Retained Earnings.

## Inventory module

Mirrors the Accounting module's shape on purpose: `InventoryItem.quantityOnHand`
is a denormalized running total (like `Account.balance`), and every change
to it goes through `StockMovementService`, which writes an immutable
`stock_movement` row and updates the item's quantity in the same transaction
(like posting a `JournalEntry` updates account balances) - so at any point,
quantity-on-hand is provably just the sum of that item's movement history.

Three movement types, each its own endpoint under `/api/inventory/movements`:
- **`POST /receipts`** — stock in (auto-numbers `GR-2026-######` if no
  reference given); optionally tags a vendor `Party`
- **`POST /issues`** — stock out; rejected with a `BusinessException` if it
  would take quantity negative
- **`POST /adjustments`** — signed correction (e.g. after a physical count),
  also rejected if it would go negative

`GET /api/inventory/items/low-stock` returns active items at or below their
reorder point - the query the Sales module will eventually use to block
selling out-of-stock items, and Procurement will use to auto-suggest
purchase orders.

## Extending with the next module (Sales or Procurement)

1. **Backend:** add `entity/sales/*` (or `procurement`), `model/sales/*`,
   repositories, `service/SalesOrderService(+Impl)` (call `auditLogService.log*()`
   in every write, `documentNumberingService.nextNumber(DocumentType.SALES_ORDER)`
   for numbering, reference `Party` for the customer/vendor and
   `InventoryItemRepository`/`StockMovementService` to check and decrement
   stock on fulfillment), `controller/SalesOrderController`. Add a new
   Flyway migration `V5__sales_schema.sql`.
2. **Frontend:** add `pages/inventory/*.vue`, register routes in
   `router/index.js`, add a nav entry in `AppLayout.vue`.
3. **Auth:** add any new scopes/roles needed (e.g. `INVENTORY_MANAGER`) to the
   auth-server's seed data and to `@PreAuthorize` checks on the new endpoints.

## Known scaffold limitations (intentional, to keep this reviewable)

- No refresh-token rotation UI, no MFA, no password-reset flow yet.
- The Accounting module covers Chart of Accounts + Journal Entries + reports
  only — no period-close / closing entries yet, though the ledger data
  model supports building that directly from `journal_line`.
- No automated tests yet (unit/integration) — see "Testing" below for what
  exists so far and what's still missing.
