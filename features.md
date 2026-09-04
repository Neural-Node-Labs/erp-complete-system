# Features

Everything currently implemented in this ERP system, organized by module. This is
the "what exists" reference; see `usage.md` for "how to actually use it" with
worked examples, and `README.md` for architecture/setup.

---

## Platform / cross-cutting

| Feature | Notes |
|---|---|
| **OAuth2/OIDC Authorization Server** | Own Spring Authorization Server container. Authorization Code + PKCE for the frontend (public client, no secret), client-credentials for service-to-service. Registered clients, issued tokens, and consents are all persisted in Postgres (survive a restart or running multiple replicas). |
| **JWT resource-server security** | The backend never touches passwords — it validates the JWT's signature against the auth-server's published JWKs and reads a `roles` claim baked into the token at issuance. |
| **Role-based access control** | Every write endpoint is `@PreAuthorize`-gated per module role (see the Roles table below). `/api/audit-logs/**` is additionally restricted at the security-filter level, not just `@PreAuthorize`, so it fails closed even if one layer is misconfigured. |
| **`Party`** | One shared master-data table for every person/company the ERP deals with (`CUSTOMER`, `VENDOR`, `EMPLOYEE`, `OTHER`), used by Accounting, Sales, Procurement, HR, and CRM instead of five near-duplicate tables. |
| **`DocumentNumberingService`** | Atomic, concurrency-safe document numbering (`SO-2026-000042` style) via a single `INSERT ... ON CONFLICT ... RETURNING` — no in-process counter that breaks under horizontal scaling. Used by every module that creates a numbered document. |
| **Audit logging** | Every create/update/deactivate/post/void/terminate across every module writes an immutable row to `audit_log` (module, entity type/id, action, who, before/after JSON snapshot). Readable only by `ADMIN` via `GET /api/audit-logs` and `GET /api/audit-logs/entity/{type}/{id}`. |
| **Uniform API envelope** | Every endpoint returns `ApiResponse<T>` (`success`, `data`, `message`, `timestamp`) regardless of module. |

### Roles

| Role | Grants access to |
|---|---|
| `ADMIN` | Everything, including `/api/audit-logs` |
| `ACCOUNTANT` | Accounting (Accounts, Journal Entries, Reports), Parties |
| `INVENTORY_CLERK` | Inventory (Items, Movements) |
| `SALES` | Sales (Orders, Invoices, Payments) |
| `PROCUREMENT` | Procurement (Orders, Bills, Payments) |
| `HR` | HR (Employees, Attendance, Payroll Runs) |
| `CRM` | CRM (Opportunities, Tickets, Activities) |
| `MANUFACTURING` | Manufacturing (Bills of Materials, Work Orders) |

Only `ADMIN`, `ACCOUNTANT`, and `INVENTORY_CLERK` are actually seeded to a user today
(the built-in `admin` account has all three). `SALES`/`PROCUREMENT`/`HR`/`CRM`/
`MANUFACTURING` exist as enforced roles but have no seeded user or admin UI to
grant them yet — see "Known gaps" in `README.md`. In the meantime, `admin` can
exercise every module in the system because every `@PreAuthorize` check
includes `ADMIN`.

---

## Accounting

Chart of Accounts + double-entry Journal Entries + live financial reports.

- **Chart of Accounts** — `ASSET`/`LIABILITY`/`EQUITY`/`REVENUE`/`EXPENSE` accounts,
  optional parent for hierarchy, denormalized running `balance` kept in sync by
  every posted Journal Entry.
- **Journal Entries** — `DRAFT → POSTED`, or `VOID` (only from `POSTED`, reverses
  the balance impact). A draft must balance (total debits = total credits) and no
  single line may have both a debit and a credit.
- **Reports**, computed live from `journal_line` on `POSTED` entries only (never
  from the denormalized `balance` column, so any past date is always accurate):
  - Trial Balance (`asOf` date)
  - Income Statement (`from`/`to` range)
  - Balance Sheet (`asOf` date) — shows unclosed current-period net income as its
    own line since there's no period-close/closing-entries feature yet.

**Endpoints:** `/api/accounting/accounts`, `/api/accounting/journal-entries`,
`/api/accounting/reports/{trial-balance,income-statement,balance-sheet}`

---

## Inventory

- **Items** — SKU, cost, sale price, reorder point, denormalized `quantityOnHand`.
- **Stock Movements** — `RECEIPT` / `ISSUE` / `ADJUSTMENT`, each an immutable row
  that updates `quantityOnHand` atomically in the same transaction (same pattern
  as posting a Journal Entry updates account balances). Issues/adjustments that
  would drive quantity negative are rejected.
- **Low-stock query** (`GET /api/inventory/items/low-stock`) — active items at or
  below their reorder point; this is what Sales' stock-availability check and a
  future Procurement auto-suggest feature both build on.

**Endpoints:** `/api/inventory/items`, `/api/inventory/movements/{receipts,issues,adjustments}`

---

## Sales

Quote-to-cash: order → fulfillment → invoice → payment, closing the books' AR
subledger as a side effect of real invoices rather than a bolted-on module.

- **Sales Orders** — `DRAFT → CONFIRMED → FULFILLED → INVOICED`, or `CANCELLED`
  (only from `DRAFT`/`CONFIRMED` — once fulfilled, void the resulting invoice
  instead so inventory/GL reversal happens correctly).
  - `confirm()` is a soft stock-availability check (no inventory movement yet).
  - `fulfill()` is what actually issues stock, one `StockMovementService.recordIssue()`
    call per line, all-or-nothing (no partial fulfillment in v1).
- **Sales Invoices** — created from a `FULFILLED` order (lines copied at that
  moment, so a later item price change never alters a historical invoice).
  `DRAFT → POSTED → PAID`, or `VOID` (blocked once `PAID` — reverse the payment
  first). Posting books **Dr Accounts Receivable (1100) / Cr Sales Revenue (4000)**
  + **Cr Sales Tax Payable (2150)** if the invoice has tax, via the existing
  Journal Entry service — this *is* the AR subledger, not a separate one to
  reconcile.
- **Customer Payments** — applied against a `POSTED` invoice; rejected if the
  amount exceeds the balance due; auto-marks the invoice `PAID` once the balance
  reaches zero.

**Endpoints:** `/api/sales/orders`, `/api/sales/invoices`, `/api/sales/payments`
**Frontend:** `/sales/orders` (full order form with lines, confirm/fulfill/cancel),
`/sales/invoices` (create from a fulfilled order, post, void, embedded payment dialog)

---

## Procurement

The structural mirror of Sales — same shape, opposite direction (buy instead of
sell, receive instead of issue, AP instead of AR).

- **Purchase Orders** — `DRAFT → APPROVED → RECEIVED → BILLED`, or `CANCELLED`
  (only from `DRAFT`/`APPROVED`). `approve()` is a pure status gate (no
  availability check needed — there's nothing to run out of when receiving).
  `receive()` issues one `StockMovementService.recordReceipt()` call per line.
- **Vendor Bills** — created from a `RECEIVED` order (lines copied at that
  moment). `DRAFT → POSTED → PAID`, or `VOID` (blocked once `PAID`). Posting
  books **Dr Inventory (1200) / Cr Accounts Payable (2000)** for the full total
  — purchase tax is folded into the inventory debit rather than a separate
  payable line, since it's generally part of landed cost rather than a
  liability collected on someone else's behalf the way sales tax is.
- **Vendor Payments** — mirror of Customer Payments.

**Endpoints:** `/api/procurement/orders`, `/api/procurement/bills`, `/api/procurement/payments`
**Frontend:** `/procurement/orders`, `/procurement/bills` (mirrors the Sales pages exactly)

---

## HR / Payroll

The first module with no ledger-mirroring shape — attendance-driven payroll
calculation instead of a document-with-lines workflow.

- **Employee Profiles** — a 1:1 extension of `Party` (`party_type = EMPLOYEE`)
  holding only employment-specific fields (employee number, job title,
  department, hire date, base salary, `ACTIVE`/`TERMINATED` status) that `Party`
  itself doesn't have. Name/email/phone still live on the `Party` row.
- **Attendance** — one record per employee per work day (`PRESENT`/`ABSENT`/
  `LEAVE`/`HOLIDAY` + hours worked). Recording the same employee+date twice
  upserts rather than duplicating.
- **Payroll Runs** — `DRAFT → CALCULATED → APPROVED → PAID`, or `CANCELLED` (any
  state before `PAID`, which can never be cancelled since it's already posted).
  - `calculate()` is a **real computation**, not a stub: for every `ACTIVE`
    employee hired on or before the period end, it counts their `PRESENT`
    attendance days in the period, divides by actual business days (Mon–Fri) in
    that range, and prorates `baseSalary` by that ratio. It's idempotent and
    re-runnable any number of times while `DRAFT`/`CALCULATED` (e.g. after
    correcting an attendance record) — unlike Sales' `confirm()`, there's no
    external availability to check, so recalculating is always safe.
  - `pay()` is the only step touching the GL: **Dr Salaries and Wages (6100) /
    Cr Cash (1000)** for the total net pay.

**Endpoints:** `/api/hr/employees`, `/api/hr/attendance`, `/api/hr/payroll-runs`
**Frontend:** `/hr/employees`, `/hr/attendance` (per-employee date-range view + daily
recording), `/hr/payroll-runs` (create → calculate → approve → pay, with a
per-employee line breakdown in the view dialog)

---

## CRM / Support

The first module with **no general-ledger or inventory interaction at all** —
pure record-keeping. Also the only module with no restriction on which `Party`
type it applies to (Sales requires `CUSTOMER`, Procurement requires `VENDOR`;
CRM accepts any, since a prospect who isn't a customer *yet* is exactly the
point of a CRM).

- **Opportunities** — sales pipeline: `NEW → QUALIFIED → PROPOSAL → NEGOTIATION`,
  with `markWon()`/`markLost()` reachable from **any** non-terminal stage rather
  than requiring a specific predecessor (real deals close from wherever they
  happen to be). `WON`/`LOST` are terminal; `changeStage()` refuses to set either
  directly — use the dedicated endpoints.
- **Support Tickets** — deliberately more permissive than a document workflow:
  any status can move to any other status (agents reopen/reprioritize
  constantly), with exactly one enforced rule — moving to `RESOLVED` requires
  non-blank resolution notes. Priority: `LOW`/`MEDIUM`/`HIGH`/`URGENT`.
- **Activities** — a free-standing interaction log (`CALL`/`EMAIL`/`MEETING`/
  `NOTE`) against a party, optionally tied to one opportunity or ticket (or
  neither, for a standalone note). No state machine at all — logged once, never
  changes.

**Endpoints:** `/api/crm/opportunities`, `/api/crm/tickets`, `/api/crm/activities`
**Frontend:** `/crm/opportunities` (inline stage dropdown + won/lost buttons),
`/crm/tickets` (inline status dropdown + assign), `/crm/activities` (filterable
by party, or auto-filtered when reached via the "Activities" link on an
opportunity/ticket row)

---

## Manufacturing

The first module where a single document consumes **multiple** inventory items
(every component) to produce **one** inventory item (the finished good) —
every other module's stock movements are always one item per line. Like HR,
this one also touches no general ledger at all - it's an inventory-only
workflow.

**Scope decision:** single-level BOMs only. A component that's itself a
manufactured item just gets its own separate Work Order upstream, rather than
this module doing recursive multi-level BOM explosion. This is a common
simplification even in mid-size MRP systems and keeps this a genuinely
working v1 rather than an open-ended engine.

- **Bills of Materials** — a produced item (`itemId`) + a named/versioned
  recipe (`name`, e.g. `"v1"`) + component lines (`componentItemId`,
  `quantityPerUnit`). **Immutable once created** — create + deactivate only,
  no in-place line editing. If the recipe changes, deactivate this version and
  create a new one; this sidesteps ever having to reconcile an edit against
  Work Orders that already snapshotted lines from an earlier version. An item
  cannot be listed as its own component (validated on create).
- **Work Orders** — `DRAFT → RELEASED → IN_PROGRESS → COMPLETED`, or
  `CANCELLED` (only from `DRAFT`/`RELEASED` — once components have been
  issued, there is no un-consume path in this version).
  - Creating one **snapshots** the BOM's lines into `WorkOrderComponent` rows
    (`quantityRequired = bomLine.quantityPerUnit * quantityToProduce`) at that
    moment — same "copy at creation, never re-derive" reasoning as Sales
    Invoice lines, so a later BOM edit (a new version) never retroactively
    changes what an already-created work order requires.
  - `release()` is a soft availability check across every component (mirrors
    `SalesOrder.confirm()`) — no inventory movement yet.
  - `start()` is what actually issues stock: one
    `StockMovementService.recordIssue()` call per component, all in the same
    transaction — the genuinely different inventory interaction this module
    exists for (N items consumed, not 1).
  - `complete()` receives the single finished-good output via one
    `StockMovementService.recordReceipt()` call for `quantityToProduce` — no
    yield variance modeled in v1 (you get exactly what you planned to
    produce, or the work order isn't complete yet).

**Endpoints:** `/api/manufacturing/boms`, `/api/manufacturing/work-orders`
**Frontend:** `/manufacturing/boms` (component-line builder, deactivate),
`/manufacturing/work-orders` (BOM picker scoped to the chosen item, release →
start → complete lifecycle buttons, a view dialog showing exactly which
components will be/were consumed)

---

## Testing

| Module | Unit tests (mocked repos) | Notes |
|---|---|---|
| Accounting | `AccountServiceImplTest`, `JournalEntryServiceImplTest` | Unbalanced-entry rejection, debit+credit-on-one-line rejection, balance updates on post, double-post rejection |
| Inventory | `StockMovementServiceImplTest` | Over-issue rejection, non-positive quantity rejection, negative-adjustment rejection |
| Sales | `SalesOrderServiceImplTest`, `SalesInvoiceServiceImplTest`, `CustomerPaymentServiceImplTest` | Full state-machine coverage including the GL posting lines |
| Procurement | `PurchaseOrderServiceImplTest`, `VendorBillServiceImplTest`, `VendorPaymentServiceImplTest` | Mirror of the Sales tests |
| HR/Payroll | `EmployeeProfileServiceImplTest`, `AttendanceServiceImplTest`, `PayrollRunServiceImplTest` | Includes a proration-math test (4/5 working days present on a $1000 salary → $800.00 gross) |
| CRM | `OpportunityServiceImplTest`, `SupportTicketServiceImplTest`, `ActivityServiceImplTest` | Stage/status transition rules |
| Manufacturing | `BillOfMaterialsServiceImplTest`, `WorkOrderServiceImplTest` | Self-as-own-component rejection, component snapshot-quantity math, insufficient-stock rejection on release, multi-component issue on start |

Integration tests against a real Postgres exist for a handful of repositories
(`AccountRepositoryIntegrationTest`, `InventoryItemRepositoryIntegrationTest`,
`JournalLineReportingQueriesIntegrationTest`) and services
(`AuditLogServiceImplIntegrationTest`, `DocumentNumberingServiceIntegrationTest`)
via Testcontainers, proving the actual SQL/Flyway migrations/R2DBC mapping work
together — not just that service logic is right when the repository is mocked.
**Not yet extended to Sales/Procurement/HR/CRM/Manufacturing** — see `README.md`'s known gaps.

Run everything with `cd backend && mvn test`.
