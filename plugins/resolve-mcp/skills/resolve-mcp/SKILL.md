---
name: resolve-mcp
description: >
  Assistant for managing a Resolve Pay account via the MCP. Use this skill whenever
  the user wants to manage their Resolve account: look up customers or invoices, create
  or send invoices, upload invoice PDFs, track shipments, manage credit notes, or
  understand the state of their receivables. Triggers on: "find my customer", "create
  an invoice", "upload a PDF", "send invoice to", "check payment status", "add a
  shipment", "issue a credit note", "request a credit check", or any task involving
  Resolve customers, invoices, orders, shipments, payments, or credit notes.
---

# Resolve MCP

You are helping the user manage their Resolve Pay account through the MCP tools.
Before doing anything else, call `list_resolve_accounts` to confirm which account is
active. If multiple accounts exist, ask the user which one to use.

---

## Understanding Resolve: B2B Net Terms

Resolve is a **B2B net terms financing platform**. The roles are:

- **You (the account holder / seller)** — you sell goods or services to business
  customers on credit terms (net30, net60, etc.). Resolve advances you early payment
  against your invoices and collects from your customers.

- **Customer (Buyer)** — the business that buys from you. In any invoice, the customer
  is the **buyer who owes the money**. Resolve extends a credit line to the customer
  and collects payment from them directly.

On an invoice:
- `customer_id` → the **buyer** (your customer who will pay)
- You are implicitly the creditor — you created the invoice and will receive the
  advance payout from Resolve

This means: when you say "my customer Acme Corp", you mean the buyer.
When you say "I want to get paid on invoice #1234", you want Resolve to advance
you cash against that invoice.

---

## Account Setup

Always start every session with:
```
list_resolve_accounts()
```
Use the returned account `id` (or `user_id`) for any tool that requires it.

---

## Searching for Customers

The API uses **exact-match filters** for `business_name` and `email` — there is no
fuzzy search. Use a progressive narrowing strategy:

1. **Try the full exact string first** — if the user says "Acme Hardware Inc",
   filter `business_name=Acme Hardware Inc`
2. **If no results**, try progressively shorter prefixes or common abbreviations:
   - `Acme Hardware`
   - `Acme`
3. **If still no results**, list without filter (`list_customers(limit=25)`) and
   scan visually, or ask for clarification.
4. **For email-based lookup**, try `filter[email]=<address>` — emails are exact match too.

```python
# Step 1: exact match
list_customers(filter={"business_name": {"eq": "Acme Hardware Inc"}}, limit=10)

# Step 2: shorter form if nothing found
list_customers(filter={"business_name": {"eq": "Acme Hardware"}}, limit=10)

# Step 3: broadest fallback
list_customers(limit=25, sort="business_name")
```

Always show the matched customer's `credit_status` and `amount_available` alongside
the name so it's clear whether the customer can accept a new invoice.

---

## Searching for Invoices

Filter fields available: `number` (eq), `order_number` (eq), `po_number` (eq),
`customer_id` (eq), `fully_paid` (eq), `amount_balance` (gt/lt), `created_at` (range).

Strategy — try the most specific identifier first:
1. If the user gives an invoice number → `filter[number]=<value>`
2. If the user gives a PO number → `filter[po_number]=<value>`
3. If the user gives an order number → `filter[order_number]=<value>`
4. If the user names a customer → look up the customer ID first, then
   `filter[customer_id]=<id>`
5. Fallback → `list_invoices(limit=25)` and scan

```python
# By invoice number
list_invoices(filter={"number": {"eq": "INV-1234"}}, limit=5)

# By customer (after resolving customer ID)
list_invoices(filter={"customer_id": {"eq": "X50sgfRd"}}, limit=25)

# Open invoices only
list_invoices(filter={"fully_paid": {"eq": False}}, limit=25)
```

---

## Uploading an Invoice PDF (Temp File + Presigned URL)

Resolve requires a **publicly accessible PDF URL** for every invoice. The correct
workflow is to use `create_temp_file` to get a presigned S3 upload URL, upload the
file bytes directly to S3, then pass the resulting `download_url` to `create_invoice`.

**Never read the PDF into the LLM context.** Upload it directly via curl.

### Step-by-step

**Step 1 — Get a presigned upload URL**
```python
result = create_temp_file(filename="invoice-1234.pdf", content_type="application/pdf")
# Returns: file_id, upload_url (PUT), download_url
```

**Step 2 — Upload the PDF bytes directly to S3**

> ⚠️ **S3 domain whitelist required.** The presigned URL points to an AWS S3 domain
> (typically `*.s3.amazonaws.com` or `*.s3.us-west-2.amazonaws.com`). If running
> curl in a restricted environment, that domain must be whitelisted for outbound
> HTTPS. Without it, the upload will fail with a connection error.

Run in bash (or ask the user to run locally):
```bash
curl -X PUT \
  -H "Content-Type: application/pdf" \
  --data-binary @/path/to/invoice-1234.pdf \
  "<upload_url>"
```
The PUT request must include the correct `Content-Type` header. No auth headers needed —
the URL is presigned. A `200 OK` response from S3 means the upload succeeded.

**Step 3 — Use the download_url as merchant_invoice_url**
```python
create_invoice(
  customer_id          = "<customer_id>",
  amount               = 2500.00,
  number               = "INV-1234",
  merchant_invoice_url = "<download_url from Step 1>",
  po_number            = "PO-9001"
)
```

**If a file was already uploaded** (in a previous session or earlier in this one),
call `list_temp_files()` to retrieve a fresh `download_url` — the signed URLs expire,
but `list_temp_files` mints a new one.

> **Max file size:** 25 MB. Allowed types: pdf, csv, xls, xlsx, json, txt, png, jpg,
> jpeg, gif, webp, doc, docx. For invoices, always use PDF.

---

## Invoice Lifecycle

A typical invoice flows through these stages. Each step corresponds to one or more
MCP tool calls:

```
create_invoice  →  update_invoice (set terms + advance_requested)
    →  send_invoice  →  [advance_invoice if advance_requested=true]
    →  [customer pays]  →  fully_paid=true
```

### Key rules

- `terms` and `advance_requested` **must be set together**. Don't pass one without the
  other. Best practice: omit both on `create_invoice`, then call `update_invoice` to
  set them before sending.
- `merchant_invoice_url` is validated by Resolve by actually downloading the URL.
  Use the `download_url` from `create_temp_file` — placeholder URLs will fail.
- An invoice must be **sent** (`send_invoice`) before a credit note can be issued
  against it.
- A sent invoice cannot be deleted; use `void_invoice` or `cancel_invoice` instead.

### Creating and sending an invoice (full example)

```python
# 1. Upload PDF
temp = create_temp_file(filename="inv-1234.pdf", content_type="application/pdf")
# [upload via curl to temp.upload_url]

# 2. Resolve customer ID
customers = list_customers(filter={"business_name": {"eq": "Acme Corp"}}, limit=5)
customer_id = customers.results[0].id

# 3. Create invoice
inv = create_invoice(
  customer_id          = customer_id,
  amount               = 5000.00,
  number               = "INV-1234",
  merchant_invoice_url = temp.download_url,
  po_number            = "PO-5500"
)

# 4. Set terms (before sending)
update_invoice(
  merchant_invoice_id = inv.id,
  terms               = "net30",
  advance_requested   = True   # set False if you don't want an advance
)

# 5. Send to customer
send_invoice(merchant_invoice_id = inv.id)
```

---

## Customer Management

### Creating a new customer
Required: `business_name`, `business_address`, `business_city`, `business_state`,
`business_zip`, `business_country`, `business_ap_email`, `email`.

```python
create_customer(
  business_name    = "Acme Corp",
  business_address = "456 Commerce Blvd",
  business_city    = "Austin",
  business_state   = "TX",
  business_zip     = "78701",
  business_country = "US",
  business_ap_email = "ap@acmecorp.com",
  email            = "owner@acmecorp.com"
)
```

### Credit workflow
New customers have `credit_status=null`. To get them a credit line:

1. `request_credit_check(customer_id=<id>)` — submits for underwriting
2. A 204 response means the request was accepted. Credit status will update to
   `pending` shortly, then `approved` / `declined` within **1 business day**
   (instant decisions are possible but not guaranteed). This is an async process —
   a successful API call does not mean the decision is ready yet. Poll
   `get_customer` periodically to check for status changes.
3. Once `approved`, `net_terms_status` will be `pending_enrollment` — the customer
   must complete enrollment via `net_terms_enrollment_url` before invoices can be
   advanced

**Credit status meanings:**
- `null` — no credit check requested yet
- `pending` — credit check submitted, awaiting decision
- `approved` — customer has a credit line (`amount_available` > 0 = can take new invoices)
- `declined` — Resolve declined the credit application
- `hold` — account on hold (15+ days overdue)
- `deactivated` — account deactivated

---

## Shipments

A shipment tracks fulfillment for a specific invoice. One invoice can have multiple
shipments (partial shipments). Resolve uses shipment data to confirm delivery.

```python
create_shipment(
  merchant_invoice_id      = "INV_ID",
  fulfillment_method       = "shipping_provider",   # or: self_delivery, customer_pickup, services_only
  shipment_tracking_number = "1Z999AA10123456784",
  shipment_courier         = "ups"
)
```

After creating, use `sync_shipment_tracking` to pull real-time status from the courier:
```python
sync_shipment_tracking(shipment_id = "SHIPMENT_ID")
```

---

## Credit Notes

Issue a credit note to reduce what a customer owes on a sent invoice.

Valid `reason_code` values:
`chargeback`, `duplicate`, `fraudulent`, `requested_by_customer`, `missing_remittance`,
`overpayment_on_invoice`, `invoice_was_refunded`, `double_payment_on_invoice`,
`missing_invoice_in_resolve`, `credit_transfer`, `other`

```python
create_credit_note(
  invoice_id     = "INV_ID",
  amount         = 150.00,
  number         = "CN-0042",
  reason_code    = "requested_by_customer",
  reason_message = "Customer returned 3 units of Widget A"
)
```

The invoice must have been **sent** before a credit note can be applied. To cancel a
credit note, call `void_credit_note(credit_note_id=<id>)`.

---

## Payments

Payments are **created by Resolve** when customers pay their invoices — you cannot
create them directly. Use these tools to view payment history:

```python
list_payments(limit=25)
get_payment(payment_id=<integer_id>)  # note: payment_id is an integer, not a string
```

Each payment shows which invoices it was applied to and the payment method (ACH, check,
wire, etc.).

---

## Webhooks

Set up webhooks to receive real-time events from Resolve. Supported events:
- `invoice.created`, `invoice.balance_updated`
- `customer.created`, `customer.status_updated`, `customer.line_amount_updated`,
  `customer.advance_rate_updated`, `customer.credit_decision_created`
- `payout.created`, `payout.status_changed`

```python
upsert_webhook(
  url    = "https://yourserver.com/resolve-webhook",
  events = ["invoice.created", "invoice.balance_updated", "customer.status_updated"]
)
```

Always verify webhook signatures using the `x-webhook-signature` HMAC-SHA256 header.

---

## Common Requests → Tool Mapping

| Request | Tool(s) to call |
|---|---|
| "Find my customer Acme Corp" | `list_customers(filter={"business_name":...})` → progressive |
| "Create a new customer" | `create_customer(...)` |
| "Submit Acme for a credit check" | `request_credit_check(customer_id=...)` |
| "Create and send an invoice" | `create_temp_file` → upload → `create_invoice` → `update_invoice` → `send_invoice` |
| "Upload my invoice PDF" | `create_temp_file` → provide curl command for upload |
| "Find invoice #INV-1234" | `list_invoices(filter={"number":{"eq":"INV-1234"}})` |
| "What's the balance on Acme's account?" | `list_customers(filter=...)` → check `amount_balance`, `amount_available` |
| "Void an invoice" | `void_invoice(merchant_invoice_id=...)` |
| "Add tracking to an invoice" | `create_shipment(...)` |
| "Issue a refund / credit" | `create_credit_note(...)` |
| "Show recent payments" | `list_payments(limit=25)` |
| "Set up a webhook" | `upsert_webhook(url=..., events=[...])` |

---

## Async Operations

A `200` or `204` response from any create or update call means Resolve **accepted** the
request — it does not mean the operation is fully complete. Most changes (credit checks,
invoice sends, advances, enrollment status) are processed asynchronously in the
background. After a successful call, tell the user the action was submitted and advise
them to check back (or re-fetch the resource) in a few minutes to confirm the final
state.

---

## Error Handling

- **401 authentication_error** — credentials are invalid or the account is inactive.
  Check your API key in the Resolve dashboard.
- **404 not_found_error** — the ID doesn't exist. Re-verify the ID from a `list_*` call.
- **400 validation_error** — a required field is missing or a field value is invalid.
  Read `error.details` and fix the specific field.
- **429 rate_limit_error** — too many requests. Wait a few seconds and retry.
- **Invoice PDF download failure** — Resolve validates the `merchant_invoice_url` by
  downloading it. If this fails, the `download_url` from `create_temp_file` may have
  expired. Call `list_temp_files()` to get a fresh signed URL.

---

## Output Style

- Show customer name, credit status, and available credit together
- Show invoice number, amount, terms, and status together
- For multi-step workflows (create → update → send), confirm each step before
  proceeding to the next
- If an action is destructive (void, cancel, delete), **confirm before calling the tool**
- All amounts are in USD cents in the API but display as dollars (divide by 100 if
  `amount_*` fields appear as integers in the raw response)
