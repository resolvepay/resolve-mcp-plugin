---
name: customer-management
description: >
  Customer onboarding and credit management for a Resolve Pay account via the MCP.
  Use this skill for any customer-heavy task: adding new customers, running credit
  checks, tracking enrollment, reading credit lines and available credit, and
  updating customer details. Triggers on: "add a customer", "create a customer",
  "run a credit check", "is [customer] approved", "what's [customer]'s credit line",
  "available credit", "enroll a customer", "update customer details", or any
  customer onboarding or credit workflow.
---

# Resolve Customer Management

You are managing customers (buyers) on a Resolve Pay account through the MCP tools.
In Resolve's B2B model, the **customer is the buyer who owes the money** — Resolve
extends the credit line to them and collects from them directly. This skill assumes
the Resolve MCP connector is already installed.

---

## Finding Customers

Filters are **exact-match** (`business_name`, `email`) — there is no fuzzy search.
Narrow progressively instead of giving up:

```python
# 1. Full exact string
list_customers(filter={"business_name": {"eq": "Acme Hardware Inc"}}, limit=10)

# 2. Shorter prefixes / common abbreviations
list_customers(filter={"business_name": {"eq": "Acme Hardware"}}, limit=10)
list_customers(filter={"business_name": {"eq": "Acme"}}, limit=10)

# 3. Email (also exact match)
list_customers(filter={"email": {"eq": "ap@acmecorp.com"}}, limit=10)

# 4. Broadest fallback — list and scan
list_customers(limit=25, sort="business_name")
```

Always show `credit_status` and `amount_available` next to the name so it's clear
whether the customer can take new invoices.

---

## Creating a Customer

Required fields: `business_name`, `business_address`, `business_city`,
`business_state`, `business_zip`, `business_country`, `business_ap_email`, `email`.

```python
create_customer(
  business_name     = "Acme Corp",
  business_address  = "456 Commerce Blvd",
  business_city     = "Austin",
  business_state    = "TX",
  business_zip      = "78701",
  business_country  = "US",
  business_ap_email = "ap@acmecorp.com",
  email             = "owner@acmecorp.com"
)
```

Before creating, search first — emails are not unique in Resolve, and duplicate
customer records cause reconciliation pain later. If a close match exists, ask the
user whether to use it instead.

Update details later with `update_customer(customer_id=..., ...)`.

---

## The Credit Workflow

New customers start with `credit_status=null`. To get them a credit line:

1. **`request_credit_check(customer_id=<id>)`** — submits for underwriting. A `204`
   means the request was **accepted**, not decided.
2. Status moves to `pending`, then `approved` / `declined` within **1 business day**
   (instant decisions happen but are not guaranteed). This is async — poll
   `get_customer` to check for changes; don't assume the decision is ready.
3. Once `approved`, `net_terms_status` becomes `pending_enrollment` — the customer
   must complete enrollment via their `net_terms_enrollment_url` before invoices
   can be advanced. Share that URL with the user if they ask how to get the
   customer enrolled.

### Credit status meanings

| Status | Meaning |
|---|---|
| `null` | No credit check requested yet |
| `pending` | Credit check submitted, awaiting decision |
| `approved` | Has a credit line — `amount_available` > 0 means new invoices fit |
| `declined` | Resolve declined the application |
| `hold` | Account on hold (15+ days overdue) |
| `deactivated` | Account deactivated |

---

## Reading a Customer's Credit Position

From `get_customer` / `list_customers`, the fields that matter:

- `credit_status` — see table above
- `amount_available` — headroom for new invoices
- `amount_balance` — what they currently owe
- `net_terms_status` — enrollment state (`pending_enrollment` blocks advances)

When the user asks "can I invoice Acme for $X?", check: `credit_status=approved`,
enrollment complete, and `amount_available >= X`. If any fail, say which one and
what fixes it (credit check, enrollment link, or a smaller amount).

---

## Common Requests → Tool Mapping

| Request | Tool(s) |
|---|---|
| "Add Blue Ridge Supply as a customer" | search first → `create_customer(...)` |
| "Run a credit check on Acme" | `request_credit_check(customer_id=...)` |
| "Is Acme approved yet?" | `get_customer(...)` → report `credit_status` |
| "What's Acme's available credit?" | `get_customer(...)` → `amount_available` |
| "Get Acme enrolled" | share `net_terms_enrollment_url` from the customer record |
| "Update Acme's AP email" | `update_customer(customer_id=..., business_ap_email=...)` |

---

## Async Operations & Errors

- A `200`/`204` means Resolve accepted the request — credit decisions and enrollment
  complete in the background. Tell the user it was submitted and to check back.
- **400 validation_error** — a required field is missing/invalid; read
  `error.details` and fix that field.
- **404 not_found_error** — re-verify the customer ID from a `list_customers` call.

## Safety

- Confirm before anything destructive or state-changing the user didn't explicitly
  ask for.
- Amounts: display as dollars; if `amount_*` fields arrive as integer cents, divide
  by 100.
