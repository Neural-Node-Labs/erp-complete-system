# ERP Platform

A modular ERP system built to a consistent enterprise layering and tech stack
across every module:

- **Frontend:** Vue 3 + Vuetify (Material Design), routed multi-page SPA
- **Backend:** Java 21, Spring WebFlux (reactive), layered
  `entity / model / repository / service / controller / common`
- **Auth:** a **custom OAuth2/OIDC Authorization Server** (Spring Authorization
  Server), its own container, backed by Postgres
- **Database:** PostgreSQL 16, two logical databases (`erp_auth`, `erp_core`)

**Modules implemented:** Accounting/Finance, Inventory, Sales, Procurement,
HR/Payroll, CRM/Support, and Manufacturing — see **[`features.md`](./features.md)**
for a full catalog of what each one does, and **[`usage.md`](./usage.md)** for
worked examples (curl walkthroughs) of every module's end-to-end workflow.

- **Cross-cutting foundation:** a shared `Party` model (customers/vendors/
  employees/other, one table for every module) and an atomic
  `DocumentNumberingService` (`SO-2026-000042` style sequences, safe under
  concurrency) that every document-producing module reuses
- **Audit:** every create/update/deactivate/post/void/terminate across every
  module is written to an append-only `audit_log` table, viewable only by ADMIN

Everything runs in Docker containers via one `docker-compose.yml`.

## Project layout

```
erp-system/
├── docker-compose.yml
├── features.md                           # what's implemented, module by module
├── usage.md                              # how to run it + curl walkthroughs
├── infra/postgres/init-multi-db.sh       # creates erp_auth + erp_core on first boot
├── auth-server/                          # OAuth2/OIDC Authorization Server (port 9000)
│   └── src/main/java/com/erp/authserver/
│       ├── config/    AuthorizationServerConfig, ClientSeeder, DefaultSecurityConfig, PersistentJwkSourceConfig
│       ├── entity/    AppUser
│       ├── repository/AppUserRepository
│       └── service/   JpaUserDetailsService
├── backend/                              # Reactive ERP core API (port 8080)
│   └── src/main/java/com/erp/backend/
│       ├── common/    ApiResponse, exceptions, GlobalExceptionHandler, CurrentUser
│       ├── config/    SecurityConfig (JWT validation, CORS), R2dbcConfig
│       ├── entity/    account/, journal/, audit/, party/, numbering/, inventory/,
│       │              sales/, procurement/, hr/, crm/, manufacturing/   (one package per module)
│       ├── model/     same module breakdown as entity/ (request/response DTOs)
│       ├── repository/ one Spring Data R2DBC repository per table
│       ├── service/    one Service(+Impl) per aggregate root
│       └── controller/ one controller per aggregate root
└── frontend/                             # Vue 3 + Vuetify SPA (port 5173)
    └── src/
        ├── router/      route guards (auth required; /admin/* requires ADMIN)
        ├── stores/       Pinia auth store (role getters per module, e.g. isSales/isHr)
        ├── services/     authService (OIDC/PKCE), api (axios + JWT header)
        ├── components/layout/ AppLayout (nav, gated per-module by role, top bar)
        └── pages/         Login, Callback, Dashboard, core/PartiesPage,
                            accounting/*, inventory/*, sales/*, procurement/*,
                            hr/*, crm/*, manufacturing/*, admin/AuditLogPage
```

Every backend module follows the same shape: `entity/<module>`, `model/<module>`,
one repository per table, one service per aggregate, one controller — and every
mutating service method calls `auditLogService.log*()` the same way Accounting
does.

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
- **Roles:** `ADMIN`, `ACCOUNTANT`, `INVENTORY_CLERK`

Because every module's `@PreAuthorize` check includes `ADMIN`, this one account
can exercise every module in the system today, even Sales/Procurement/HR/CRM/
Manufacturing whose dedicated roles (`SALES`/`PROCUREMENT`/`HR`/`CRM`/
`MANUFACTURING`) aren't separately seeded to any user yet (see "Known gaps"
below).

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
   JWT's signature against the auth-server's published JWKs, reads the `roles`
   claim baked into the token by `AuthorizationServerConfig`'s token
   customizer, and enforces `@PreAuthorize` rules per endpoint — it never
   talks to Postgres to check who you are.

## Auth-server persistence (and two bugs already found and fixed here)

Everything the Authorization Server needs to survive a restart (or run as
more than one replica) is in Postgres, not memory:

- **Registered clients** — `JdbcRegisteredClientRepository`. `ClientSeeder`
  inserts `erp-frontend` (public, PKCE) and `erp-backend-service`
  (client-credentials) once, on first boot only.
- **Issued authorizations/tokens and consents** — `JdbcOAuth2AuthorizationService`
  and `JdbcOAuth2AuthorizationConsentService`.
- **JWT signing key** — `PersistentJwkSourceConfig` generates one RSA key pair
  on the very first boot and reuses it on every boot after, so a restart never
  silently invalidates every previously issued token.

Two real bugs were hit and fixed building this, both worth knowing about since
a new environment (staging, a teammate's machine, CI) could reintroduce either:

1. **Postgres column-type mismatch.** The OAuth2 schema was hand-translated
   from Spring Authorization Server's generic/H2 schema (`BLOB` columns) to
   Postgres by mapping `BLOB → BYTEA`. That's wrong — `JdbcOAuth2AuthorizationService`
   serializes `attributes`/`*_value`/`*_metadata` as JSON **strings**, not
   binary, so every authorization-code/token insert failed with
   `column "attributes" is of type bytea but expression is of type character varying`.
   Fixed via `V3__fix_oauth2_authorization_column_types.sql` (additive — the
   original migration was left alone since Flyway had already validated it).
2. **Issuer/discovery hostname split across the Docker network.** The
   auth-server's issuer is `http://localhost:9000` (correct — it's what the
   browser and every already-issued token's `iss` claim uses), but the backend
   reaches it internally as `http://auth-server:9000`. Spring's `issuer-uri`
   autoconfiguration requires the discovery document's `issuer` to equal the
   exact URL it was fetched from, so it threw `IllegalStateException` on every
   request needing JWT validation. Fixed by decoupling: the backend now fetches
   signing keys from `OAUTH2_JWK_SET_URI` (internal address) and validates the
   `iss` claim separately against `OAUTH2_ISSUER` (external/canonical address)
   — two properties instead of one that had to serve both purposes.

**Lesson for any future infrastructure:** whenever something crosses the
"browser-facing hostname" vs. "container-network hostname" boundary, assume it
needs to be configurable independently rather than sharing one property.

## Testing

See **[`features.md`](./features.md#testing)** for the full breakdown of what's
covered per module. In short: every module has unit tests (mocked repositories)
covering its state machine and business rules, and a handful of repositories/
services additionally have Testcontainers integration tests against a real
Postgres, proving the actual SQL/Flyway migrations/R2DBC mapping work together.

Run them with `cd backend && mvn test`.

**Known gap:** the Testcontainers integration-test pattern exists but hasn't
been extended to Sales/Procurement/HR/CRM/Manufacturing yet — only Accounting/Inventory/Audit/
Numbering have it. Controller-layer tests (`@WebFluxTest`) and any frontend
tests also don't exist yet.

## Extending with the next module

The established pattern, followed by every module so far:

1. **Migration:** a new additive `V{n}__<module>_schema.sql` — never edit an
   already-applied one.
2. **Backend:** `entity/<module>/*`, `model/<module>/*`, one repository per
   table, one `Service(+Impl)` per aggregate root (call `auditLogService.log*()`
   in every write; `documentNumberingService.nextNumber(DocumentType.X)` for
   numbering; reuse `Party`/`InventoryItemRepository`/`StockMovementService`/
   `JournalEntryService` rather than reinventing customer/vendor/employee data,
   stock movement, or GL posting), one controller per aggregate root with
   `@PreAuthorize("hasAnyRole('<MODULE_ROLE>', 'ADMIN')")`.
3. **Frontend:** add `pages/<module>/*.vue`, register routes in
   `router/index.js`, add a nav entry in `AppLayout.vue`.
4. **Auth:** add the new role to `@PreAuthorize` checks (already enforced) and,
   eventually, to whatever grants roles to real users (see "Known gaps").

**All modules from the original scope are now built.** Sales, Procurement,
HR/Payroll, CRM/Support, and Manufacturing all follow the pattern above; see
`features.md` for what each one does and `git log`/this README's history for
the design notes behind each (e.g. Manufacturing's single-level-BOM scoping
decision, HR's attendance-driven proration). Future modules follow the same
four steps.

## Known gaps

- **No role-administration UI or API.** `SALES`/`PROCUREMENT`/`HR`/`CRM`/
  `MANUFACTURING` are enforced roles with no seeded user and no way to grant
  them to a real user except inserting directly into `app_user_role` in the
  `erp_auth` database. This is the most pressing gap before any of those five
  modules could be used by anyone other than an admin-role tester.
- **No period close / closing entries** in Accounting — the Balance Sheet shows
  unclosed current-period net income as its own line rather than assuming it's
  been swept into Retained Earnings.
- **No multi-currency, no recurring/templated journal entries, no bank
  reconciliation.**
- **No MFA, no password-reset flow, no account-lockout policy, no refresh-token
  rotation UI.**
- **Secrets are hardcoded dev defaults** (DB passwords, client secret) in
  `application.yml`/`docker-compose.yml` — fine for local dev, needs
  externalized secrets management before any non-local deployment.
- **No CI/CD pipeline, no OpenAPI/Swagger docs, no structured logging/metrics/
  tracing, no documented backup/DR strategy.**
