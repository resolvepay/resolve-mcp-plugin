---
name: invoice-management
description: >
  Deep-dive invoice management for a Resolve Pay account via the MCP. Use this skill
  for any invoice-heavy task: creating and sending invoices, uploading invoice PDFs,
  setting terms and requesting advances, voiding or cancelling invoices, issuing
  credit notes, and attaching shipments. Triggers on: "create an invoice", "send
  invoice", "upload invoice PDF", "set terms", "request an advance", "void invoice",
  "cancel invoice", "issue a credit note", "add tracking / shipment", or any
  multi-invoice workflow (batch creation, reconciliation, chasing unpaid invoices).
---

# Resolve Invoice Management

You are managing invoices on a Resolve Pay account through the MCP tools. This skill
covers the full invoice lifecycle in depth. For account basics (account selection,
customer search strategy), the base `resolve-mcp` skill applies; this skill assumes
the Resolve MCP connector is already installed.

---

## The Invoice Lifecycle

```
create_invoice  →  update_invoice (set terms + advance_requested)
    →  send_invoice  →  [advance_invoice if advance_requested=true]
    →  [customer pays]  →  fully_paid=true
```

State-changing rules — these are load-bearing, violating them causes API errors:

- **`terms` and `advance_requested` must be set together.** Never pass one without
  the other. Best practice: omit both on `create_invoice`, then set them with a
  single `update_invoice` call before sending.
- **An invoice must be sent before a credit note can be issued against it.**
- **A sent invoice cannot be deleted.** Use `void_invoice` or `cancel_invoice`.
  `delete_invoice` only works on drafts (never sent).
- **`merchant_invoice_url` is validated by download.** Resolve fetches the URL when
  the invoice is created/updated — a placeholder or expired URL fails validation.

---

## Uploading an Invoice PDF (Temp File + Presigned URL)

Resolve requires a PDF for every invoice. Use the temp-file flow — **never read the
PDF into the LLM context**:

**Step 1 — Mint a presigned upload URL**
```python
temp = create_temp_file(filename="invoice-1234.pdf", content_type="application/pdf")
# Returns: file_id, upload_url (PUT), download_url
```

**Step 2 — Upload the bytes directly to S3 via curl**
```bash
curl -X PUT \
  -H "Content-Type: application/pdf" \
  --data-binary @/path/to/invoice-1234.pdf \
  "<upload_url>"
```
No auth headers — the URL is presigned. `200 OK` means success. If curl runs in a
restricted environment, the S3 domain (`*.s3.amazonaws.com` /
`*.s3.us-west-2.amazonaws.com`) must be allowed for outbound HTTPS.

**Step 3 — Pass `download_url` as `merchant_invoice_url`**

If a file was uploaded earlier (this session or a previous one), call
`list_temp_files()` to mint a fresh `download_url` — signed URLs expire.

> **Max file size:** 25 MB. For invoices, always use PDF.

---

## Creating and Sending an Invoice (full example)

```python
# 1. Upload PDF (see above)
temp = create_temp_file(filename="inv-1234.pdf", content_type="application/pdf")
# [upload via curl to temp.upload_url]

# 2. Resolve the customer ID (exact-match filters; narrow progressively)
customers = list_customers(filter={"business_name": {"eq": "Acme Corp"}}, limit=5)
customer_id = customers.results[0].id

# 3. Create the invoice
inv = create_invoice(
  customer_id          = customer_id,
  amount               = 5000.00,
  number               = "INV-1234",
  merchant_invoice_url = temp.download_url,
  po_number            = "PO-5500"
)

# 4. Set terms + advance together, before sending
update_invoice(
  merchant_invoice_id = inv.id,
  terms               = "net30",
  advance_requested   = True   # False if no advance wanted
)

# 5. Send to the customer
send_invoice(merchant_invoice_id = inv.id)
```

Before creating, check the customer's `credit_status` is `approved` and
`amount_available` covers the invoice amount — otherwise the invoice cannot be
advanced and may be rejected.

---

## Finding Invoices

Filters (exact match unless noted): `number`, `order_number`, `po_number`,
`customer_id`, `fully_paid`, `amount_balance` (gt/lt), `created_at` (range).

```python
list_invoices(filter={"number": {"eq": "INV-1234"}}, limit=5)          # by number
list_invoices(filter={"customer_id": {"eq": "X50sgfRd"}}, limit=25)    # by customer
list_invoices(filter={"fully_paid": {"eq": False}}, limit=25)          # open invoices
```

For "unpaid over $X" style questions, combine `fully_paid=false` with
`amount_balance` gt filters, then present number, customer, amount, balance, and
due date together.

---

## Undoing Things

| Situation | Tool |
|---|---|
| Draft invoice, never sent | `delete_invoice` |
| Sent invoice, wrong details | `void_invoice` |
| Sent invoice, order fell through | `cancel_invoice` |
| Sent invoice, partial adjustment (returns, discounts, overpayment) | `create_credit_note` |

### Credit notes

Valid `reason_code` values: `chargeback`, `duplicate`, `fraudulent`,
`requested_by_customer`, `missing_remittance`, `overpayment_on_invoice`,
`invoice_was_refunded`, `double_payment_on_invoice`, `missing_invoice_in_resolve`,
`credit_transfer`, `other`.

```python
create_credit_note(
  invoice_id     = "INV_ID",
  amount         = 150.00,
  number         = "CN-0042",
  reason_code    = "requested_by_customer",
  reason_message = "Customer returned 3 units of Widget A"
)
```

Undo a credit note with `void_credit_note(credit_note_id=<id>)`.

---

## Shipments on Invoices

Resolve uses shipment data to confirm delivery. One invoice can have multiple
(partial) shipments.

```python
create_shipment(
  merchant_invoice_id      = "INV_ID",
  fulfillment_method       = "shipping_provider",  # or: self_delivery, customer_pickup, services_only
  shipment_tracking_number = "1Z999AA10123456784",
  shipment_courier         = "ups"
)
sync_shipment_tracking(shipment_id = "SHIPMENT_ID")   # pull live courier status
```

---

## Common Failure Modes

- **400 on `update_invoice` with terms** — you passed `terms` without
  `advance_requested` (or vice versa). Always set both.
- **Invoice URL validation failure** — the `download_url` expired. Call
  `list_temp_files()` for a fresh signed URL and retry.
- **Credit note rejected** — the invoice hasn't been sent yet. Send it first.
- **`delete_invoice` fails** — the invoice was sent. Use void or cancel instead.
- **Advance not happening after send** — check `advance_requested=true` was set
  before sending, and the customer is enrolled (`net_terms_status`).

A `200`/`204` response means Resolve *accepted* the request; sends, advances, and
status changes complete asynchronously. Re-fetch the invoice to confirm final state.

---

## Safety

- Confirm with the user before any destructive action: void, cancel, delete.
- For multi-step flows (create → update → send), confirm each step's success before
  the next.
- Amounts: display as dollars; if `amount_*` fields arrive as integer cents, divide
  by 100.
