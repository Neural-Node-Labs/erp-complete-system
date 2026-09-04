# Usage Guide

How to actually run this system and drive each module through a full workflow.
See `features.md` for what exists and `README.md` for architecture/setup details.

---

## 1. Start everything

```bash
cd erp-system
docker compose up --build
```

| Service | URL |
|---|---|
| Frontend | http://localhost:5173 |
| Backend API | http://localhost:8080 |
| Auth Server | http://localhost:9000 |

## 2. Log in

Open http://localhost:5173 and sign in with the seeded admin account:

- **Username:** `admin`
- **Password:** `Admin@12345`

This account has `ADMIN`, `ACCOUNTANT`, and `INVENTORY_CLERK` roles — and since
every module's `@PreAuthorize` check includes `ADMIN`, this one login can
exercise **every** module below, including Sales/Procurement/HR/CRM even though
those specific roles aren't separately seeded to any user yet.

For API calls directly (curl/Postman) rather than through the frontend, get a
token via the Authorization Code + PKCE flow (same as the frontend does), or
inspect the frontend's network tab for a live `Authorization: Bearer ...` header
to copy for quick manual testing.

Every module below now has both a UI (linked in the left nav once your account
has the matching role, or always for `admin`) and the API endpoints shown here
— the curl examples are for scripting/testing, but everything is clickable too.

---

## 3. Accounting

**Set up accounts, then post a balanced journal entry.**

```bash
# Create an account (or use the seeded starter chart of accounts - Cash 1000, AR 1100, etc.)
curl -X POST http://localhost:8080/api/accounting/accounts \
  -H "Authorization: Bearer $TOKEN" -H "Content-Type: application/json" \
  -d '{"code":"1050","name":"Petty Cash","accountType":"ASSET"}'

# Create a draft journal entry (debits must equal credits)
curl -X POST http://localhost:8080/api/accounting/journal-entries \
  -H "Authorization: Bearer $TOKEN" -H "Content-Type: application/json" \
  -d '{
    "entryDate":"2026-09-01","description":"Owner contribution",
    "lines":[
      {"accountId":"<cash-account-id>","debit":1000.00,"credit":0},
      {"accountId":"<equity-account-id>","debit":0,"credit":1000.00}
    ]}'

# Post it (this is what actually updates account balances)
curl -X POST http://localhost:8080/api/accounting/journal-entries/<id>/post \
  -H "Authorization: Bearer $TOKEN"

# Pull a report
curl "http://localhost:8080/api/accounting/reports/trial-balance?asOf=2026-09-01" \
  -H "Authorization: Bearer $TOKEN"
```

---

## 4. Inventory

**Create an item, then receive stock.**

```bash
curl -X POST http://localhost:8080/api/inventory/items \
  -H "Authorization: Bearer $TOKEN" -H "Content-Type: application/json" \
  -d '{"sku":"WIDGET-001","name":"Standard Widget","cost":5.00,"salePrice":12.00,"reorderPoint":20}'

curl -X POST http://localhost:8080/api/inventory/movements/receipts \
  -H "Authorization: Bearer $TOKEN" -H "Content-Type: application/json" \
  -d '{"itemId":"<item-id>","quantity":100,"reference":"Initial stock load"}'
```

---

## 5. Sales — the full order-to-cash cycle

**Prerequisite:** a `Party` with `partyType: CUSTOMER` and an inventory item with
enough stock.

```bash
# 1. Create a customer
curl -X POST http://localhost:8080/api/parties \
  -H "Authorization: Bearer $TOKEN" -H "Content-Type: application/json" \
  -d '{"partyType":"CUSTOMER","name":"Acme Retail","email":"ap@acme.example"}'

# 2. Create a draft sales order
curl -X POST http://localhost:8080/api/sales/orders \
  -H "Authorization: Bearer $TOKEN" -H "Content-Type: application/json" \
  -d '{
    "customerId":"<customer-id>","orderDate":"2026-09-01",
    "lines":[{"itemId":"<item-id>","quantity":10,"unitPrice":12.00,"taxRate":0.08}]
  }'

# 3. Confirm it (soft stock-availability check, no inventory movement yet)
curl -X POST http://localhost:8080/api/sales/orders/<order-id>/confirm -H "Authorization: Bearer $TOKEN"

# 4. Fulfill it (this is what actually issues stock)
curl -X POST http://localhost:8080/api/sales/orders/<order-id>/fulfill -H "Authorization: Bearer $TOKEN"

# 5. Create an invoice from the fulfilled order (lines copied at this moment)
curl -X POST http://localhost:8080/api/sales/invoices \
  -H "Authorization: Bearer $TOKEN" -H "Content-Type: application/json" \
  -d '{"salesOrderId":"<order-id>","invoiceDate":"2026-09-01","dueDate":"2026-09-30"}'

# 6. Post the invoice (this is what hits the general ledger: Dr AR / Cr Revenue [+ Cr Tax])
curl -X POST http://localhost:8080/api/sales/invoices/<invoice-id>/post -H "Authorization: Bearer $TOKEN"

# 7. Apply a payment (auto-marks the invoice PAID once the balance reaches zero)
curl -X POST http://localhost:8080/api/sales/payments \
  -H "Authorization: Bearer $TOKEN" -H "Content-Type: application/json" \
  -d '{"invoiceId":"<invoice-id>","amount":129.60,"paymentDate":"2026-09-05","method":"BANK_TRANSFER"}'
```

**Common rejections you'll hit on purpose:**
- Confirming with insufficient stock → `400` "Insufficient stock to confirm order: ..."
- Fulfilling a `DRAFT` order (must be `CONFIRMED` first) → `400` "Only CONFIRMED orders can be fulfilled..."
- Cancelling a `FULFILLED` order → `400` telling you to void the resulting invoice instead
- Overpaying an invoice → `400` "Payment of X exceeds the balance due of Y..."

---

## 6. Procurement — the mirror of Sales

```bash
# 1. Create a vendor
curl -X POST http://localhost:8080/api/parties \
  -H "Authorization: Bearer $TOKEN" -H "Content-Type: application/json" \
  -d '{"partyType":"VENDOR","name":"Widget Supply Co","email":"orders@widgetsupply.example"}'

# 2. Draft -> approve -> receive
curl -X POST http://localhost:8080/api/procurement/orders \
  -H "Authorization: Bearer $TOKEN" -H "Content-Type: application/json" \
  -d '{"vendorId":"<vendor-id>","orderDate":"2026-09-01",
       "lines":[{"itemId":"<item-id>","quantity":200,"unitCost":5.00}]}'

curl -X POST http://localhost:8080/api/procurement/orders/<order-id>/approve -H "Authorization: Bearer $TOKEN"
curl -X POST http://localhost:8080/api/procurement/orders/<order-id>/receive -H "Authorization: Bearer $TOKEN"

# 3. Bill it (Dr Inventory / Cr Accounts Payable on post), then pay it
curl -X POST http://localhost:8080/api/procurement/bills \
  -H "Authorization: Bearer $TOKEN" -H "Content-Type: application/json" \
  -d '{"purchaseOrderId":"<order-id>","billDate":"2026-09-01","dueDate":"2026-09-30"}'
curl -X POST http://localhost:8080/api/procurement/bills/<bill-id>/post -H "Authorization: Bearer $TOKEN"
curl -X POST http://localhost:8080/api/procurement/payments \
  -H "Authorization: Bearer $TOKEN" -H "Content-Type: application/json" \
  -d '{"billId":"<bill-id>","amount":1000.00,"paymentDate":"2026-09-10","method":"BANK_TRANSFER"}'
```

---

## 7. HR / Payroll

**Prerequisite:** a `Party` with `partyType: EMPLOYEE`.

```bash
# 1. Create the employee's party record, then their employment profile
curl -X POST http://localhost:8080/api/parties \
  -H "Authorization: Bearer $TOKEN" -H "Content-Type: application/json" \
  -d '{"partyType":"EMPLOYEE","name":"Jane Doe","email":"jane@erp.local"}'

curl -X POST http://localhost:8080/api/hr/employees \
  -H "Authorization: Bearer $TOKEN" -H "Content-Type: application/json" \
  -d '{"partyId":"<party-id>","employeeNumber":"EMP-001","jobTitle":"Engineer",
       "department":"Engineering","hireDate":"2026-01-01","baseSalary":6000.00}'

# 2. Log attendance for the pay period (recording the same employee+date twice upserts)
curl -X POST http://localhost:8080/api/hr/attendance \
  -H "Authorization: Bearer $TOKEN" -H "Content-Type: application/json" \
  -d '{"employeeId":"<employee-id>","workDate":"2026-09-01","status":"PRESENT","hoursWorked":8}'
# ...repeat for each work day in the period...

# 3. Create a payroll run, then calculate it (re-runnable any time before APPROVED)
curl -X POST http://localhost:8080/api/hr/payroll-runs \
  -H "Authorization: Bearer $TOKEN" -H "Content-Type: application/json" \
  -d '{"periodStart":"2026-09-01","periodEnd":"2026-09-07","payDate":"2026-09-10"}'

curl -X POST http://localhost:8080/api/hr/payroll-runs/<run-id>/calculate -H "Authorization: Bearer $TOKEN"
# -> gross/net pay per employee, prorated by daysPresent / businessDaysInPeriod

# 4. Approve, then pay (this is what posts Dr Salaries and Wages / Cr Cash)
curl -X POST http://localhost:8080/api/hr/payroll-runs/<run-id>/approve -H "Authorization: Bearer $TOKEN"
curl -X POST http://localhost:8080/api/hr/payroll-runs/<run-id>/pay -H "Authorization: Bearer $TOKEN"
```

---

## 8. CRM / Support

No general-ledger impact — pure record-keeping, so there's less ceremony.

```bash
# Opportunity pipeline
curl -X POST http://localhost:8080/api/crm/opportunities \
  -H "Authorization: Bearer $TOKEN" -H "Content-Type: application/json" \
  -d '{"partyId":"<party-id>","name":"Q4 renewal","estimatedValue":25000,"expectedCloseDate":"2026-12-01"}'

curl -X POST http://localhost:8080/api/crm/opportunities/<opp-id>/stage \
  -H "Authorization: Bearer $TOKEN" -H "Content-Type: application/json" -d '{"stage":"QUALIFIED"}'

# Close it out - won or lost, from wherever it currently sits
curl -X POST http://localhost:8080/api/crm/opportunities/<opp-id>/won -H "Authorization: Bearer $TOKEN"
# or:
curl -X POST http://localhost:8080/api/crm/opportunities/<opp-id>/lost \
  -H "Authorization: Bearer $TOKEN" -H "Content-Type: application/json" -d '{"reason":"Chose a competitor"}'

# Support ticket
curl -X POST http://localhost:8080/api/crm/tickets \
  -H "Authorization: Bearer $TOKEN" -H "Content-Type: application/json" \
  -d '{"partyId":"<party-id>","subject":"Login broken","priority":"HIGH"}'

curl -X POST http://localhost:8080/api/crm/tickets/<ticket-id>/assign \
  -H "Authorization: Bearer $TOKEN" -H "Content-Type: application/json" -d '{"assignedTo":"agent.smith"}'

curl -X POST http://localhost:8080/api/crm/tickets/<ticket-id>/status \
  -H "Authorization: Bearer $TOKEN" -H "Content-Type: application/json" \
  -d '{"status":"RESOLVED","resolutionNotes":"Restarted the auth service"}'

# Log a call/email/meeting/note against a party, optionally tied to the opportunity/ticket above
curl -X POST http://localhost:8080/api/crm/activities \
  -H "Authorization: Bearer $TOKEN" -H "Content-Type: application/json" \
  -d '{"partyId":"<party-id>","opportunityId":"<opp-id>","activityType":"CALL",
       "subject":"Renewal discussion","activityDate":"2026-09-01T15:00:00Z"}'
```

---

## 9. Manufacturing

**Prerequisite:** two inventory items — one to be the finished good, one (or
more) to be components.

```bash
# 1. Define the recipe: 1 unit of the finished good needs 3 of the component
curl -X POST http://localhost:8080/api/manufacturing/boms \
  -H "Authorization: Bearer $TOKEN" -H "Content-Type: application/json" \
  -d '{
    "itemId":"<finished-good-item-id>","name":"v1",
    "lines":[{"componentItemId":"<component-item-id>","quantityPerUnit":3}]
  }'

# 2. Create a work order to produce 10 units (snapshots the BOM: needs 30 of the component)
curl -X POST http://localhost:8080/api/manufacturing/work-orders \
  -H "Authorization: Bearer $TOKEN" -H "Content-Type: application/json" \
  -d '{"itemId":"<finished-good-item-id>","bomId":"<bom-id>","quantityToProduce":10}'

# 3. Release it (soft check: is there enough component stock to do this?)
curl -X POST http://localhost:8080/api/manufacturing/work-orders/<wo-id>/release -H "Authorization: Bearer $TOKEN"

# 4. Start it (this is what actually issues every component from stock)
curl -X POST http://localhost:8080/api/manufacturing/work-orders/<wo-id>/start -H "Authorization: Bearer $TOKEN"

# 5. Complete it (this is what receives the 10 produced units into inventory)
curl -X POST http://localhost:8080/api/manufacturing/work-orders/<wo-id>/complete -H "Authorization: Bearer $TOKEN"
```

**Common rejections you'll hit on purpose:**
- A BOM listing an item as its own component → `400` "An item cannot be a component of its own BOM"
- Releasing with insufficient component stock → `400` "Insufficient stock to release work order: ..."
- Cancelling an `IN_PROGRESS` work order → `400` "...no un-consume path in this version"

---

## 10. Viewing the audit trail (ADMIN only)

```bash
curl http://localhost:8080/api/audit-logs -H "Authorization: Bearer $ADMIN_TOKEN"
curl http://localhost:8080/api/audit-logs/entity/SalesOrder/<order-id> -H "Authorization: Bearer $ADMIN_TOKEN"
```

---

## Troubleshooting

These are real issues hit and fixed during this project's development —
included here in case you see the same symptom on a fresh environment.

| Symptom | Cause | Fix |
|---|---|---|
| Auth-server 500s with `column "attributes" is of type bytea but expression is of type character varying` | OAuth2 schema had the wrong Postgres column types | Already fixed via `V3__fix_oauth2_authorization_column_types.sql` — just make sure Flyway has actually run (check the startup log for "Successfully applied N migrations") |
| Backend 500s with `The Issuer "..." provided in the configuration did not match the requested issuer "..."` | Backend was fetching OIDC discovery from a different hostname than the token's `iss` claim | Already fixed — backend uses separate `OAUTH2_JWK_SET_URI` (internal Docker address) and `OAUTH2_ISSUER` (canonical/external address) env vars instead of one `issuer-uri` |
| `403 Forbidden` on a Sales/Procurement/HR/CRM endpoint even though you're logged in | Your user doesn't have that module's role, and isn't `ADMIN` | Log in as `admin`, or add the role manually to `app_user_role` in the `erp_auth` database (no admin UI for this yet) |
| A `POST` to create a Sales Invoice/Vendor Bill fails with "Required account with code ... does not exist" | The relevant Flyway migration (`V5`/`V6`/`V7`) hasn't run yet | Restart the backend container; Flyway runs on every boot and is safe to re-run |
