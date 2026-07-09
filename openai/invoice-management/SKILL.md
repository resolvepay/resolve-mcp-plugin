---
name: invoice-management-chatgpt
description: >
  Deep-dive invoice management for a Resolve Pay account via the MCP in ChatGPT and
  Codex. Use this skill for any invoice-heavy task: creating and sending invoices,
  attaching invoice PDFs,
  setting terms and requesting advances, voiding or cancelling invoices, issuing
  credit notes, and attaching shipments. Triggers on: "create an invoice", "send
  invoice", "upload invoice PDF", "set terms", "request an advance", "void invoice",
  "cancel invoice", "issue a credit note", "add tracking / shipment", or any
  multi-invoice workflow (batch creation, reconciliation, chasing unpaid invoices).
---

# Resolve Invoice Management (ChatGPT / Codex)

You are managing invoices on a Resolve Pay account through the MCP tools. This skill
covers the full invoice lifecycle in depth. For account basics (account selection,
customer search strategy), the base `resolve-mcp` skill applies; this skill assumes
the Resolve MCP connector is already installed.

> **File handling in this client:** files are attached directly to the conversation
> and passed straight into tool parameters. There are no `create_temp_file` /
> `list_temp_files` tools and no upload step.

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

## Attaching an Invoice PDF (File Attachments)

Resolve requires a PDF for every invoice. Any tool parameter that expects a file
URL (such as `merchant_invoice_url` on `create_invoice`) accepts a **file attached
to the conversation** — the MCP server resolves the attachment to what the Resolve
API expects. You never construct or pass URLs yourself.

**Step 1 — Ask the user to attach the file** to the conversation if they haven't
already (paperclip / drag-and-drop).

**Step 2 — Pass the attachment as the file parameter** when calling the write tool.

**Do NOT read the PDF's contents into the conversation** — pass the attachment
straight through to the tool parameter.

> **Max file size:** 25 MB. For invoices, always use PDF.

---

## Creating and Sending an Invoice (full example)

```python
# 1. Make sure the invoice PDF is attached to the conversation
#    (ask the user to attach it if it isn't)

# 2. Resolve the customer ID (exact-match filters; narrow progressively)
customers = list_customers(filter={"business_name": {"eq": "Acme Corp"}}, limit=5)
customer_id = customers.results[0].id

# 3. Create the invoice
inv = create_invoice(
  customer_id          = customer_id,
  amount               = 5000.00,
  number               = "INV-1234",
  merchant_invoice_url = <attached PDF>,
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
- **Invoice file validation failure** — Resolve could not download the file. Ask
  the user to re-attach it to the conversation and retry with the fresh attachment.
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
