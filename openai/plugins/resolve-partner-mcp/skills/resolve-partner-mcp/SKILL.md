---
name: resolve-partner-mcp-chatgpt
description: >
  Assistant for managing a Resolve Pay **partner** account via the MCP in ChatGPT and Codex. Use this skill
  when the user's Resolve account is a partner (platform or marketplace) with
  sub-merchants underneath it: listing or onboarding sub-merchants, creating and sending
  invoices on a sub-merchant's behalf, managing buyers and credit checks across
  sub-merchants, and reading payouts. Triggers on: "list my sub-merchants", "onboard a
  merchant", "create an invoice for <sub-merchant>", "which of my merchants", "partner
  account", or any task using the `partner_*` tools.
---

# Resolve Partner MCP (ChatGPT / Codex)

You are helping the user manage a Resolve Pay **partner** account through the MCP.

**Every Resolve tool you use here is prefixed `partner_`.** A partner account does not
have the merchant tools (`create_invoice`, `list_customers`, …) — they are not exposed
to it, and they would be wrong if they were: the merchant endpoints scope to a single
merchant record, while the `partner_*` tools scope across all sub-merchants beneath the
partner. The general tools (`list_resolve_accounts` and `search`) are shared and have no
prefix.

> **This is the ChatGPT / Codex variant of the skill.** Files are attached directly to
> the conversation and passed straight into tool parameters. There are no
> `create_temp_file` / `list_temp_files` tools and no upload step.

---

## First: identify the account and the partner type

```
list_resolve_accounts()
```

The returned account includes `partner_type`, which is **load-bearing** — it changes
who owns buyers and therefore how `partner_create_customer` is called:

| `partner_type` | Buyers belong to | Credit lines |
|---|---|---|
| `marketplace` | the **partner** | one line, shared across all sub-merchants |
| `single_merchant` | each **sub-merchant** | separate line per sub-merchant |

If `partner_type` is missing from the response, ask the user which model they are on
rather than guessing — the wrong choice produces a validation error on customer creation.

---

## Understanding the partner model

- **You (the partner)** — a platform or marketplace with **sub-merchants** underneath
  you. You do not sell directly; your sub-merchants do.
- **Sub-merchant** — a business that sells to buyers through your platform. Invoices
  always belong to a specific sub-merchant.
- **Buyer (customer)** — the business that owes the money. Resolve extends it credit and
  collects from it.

Invoices are **always** tied to a sub-merchant via `merchant_id`. Buyers are tied to the
partner (marketplace) or to a sub-merchant (single-merchant), per the table above.

---

## Sub-merchants

```python
partner_list_merchants(limit=25)                       # your roster
partner_get_merchant(merchant_id="<id>")               # one sub-merchant
partner_create_merchant(...)                           # onboard a new one
partner_update_merchant(merchant_id="<id>", ...)       # update details
```

Resolve `merchant_id` from this roster before creating anything against a sub-merchant.
When the user names a sub-merchant in prose ("create an invoice for Northwind"), look it
up here first and confirm the match before writing.

---

## Creating an invoice

`merchant_id` is **required** — it names the sub-merchant the invoice belongs to. This
is the main difference from the merchant API, where there is no such field.

Ask the user to attach the invoice PDF to the conversation, then pass the attachment
straight through as `merchant_invoice_url`. **Do not read the PDF's contents into the
conversation.**

```python
# 1. Find the sub-merchant
merchants = partner_list_merchants(limit=25)

# 2. Find the buyer
customers = partner_list_customers(filter={"business_name": {"eq": "Acme Corp"}}, limit=5)

# 3. Create against that sub-merchant
inv = partner_create_invoice(
  merchant_id          = "<sub_merchant_id>",       # required
  customer_id          = "<buyer_id>",
  amount               = 5000.00,                   # decimal dollars, not cents
  number               = "INV-1234",
  merchant_invoice_url = <attached PDF>,            # the conversation attachment
  po_number            = "PO-5500"
)

# 5. Set terms together, then send
partner_update_invoice(merchant_invoice_id=inv.id, terms="net30", advance_requested=True)
partner_send_invoice(merchant_invoice_id=inv.id)
```

### Lifecycle rules (same as the merchant API)

- `terms` and `advance_requested` **must be set together** — omit both on create, then
  set them with one `partner_update_invoice` before sending.
- `merchant_invoice_url` is validated by Resolve **downloading it**. Pass the
  conversation attachment; placeholder URLs fail.
- An invoice must be **sent** before a credit note can be issued against it.
- A sent invoice cannot be deleted — use `partner_void_invoice` or
  `partner_cancel_invoice`. `partner_delete_invoice` is for drafts only.
- `partner_delete_invoices` deletes several drafts in one call.

There is **no** `partner_advance_invoice` tool. Request the advance by setting
`advance_requested=True` before sending.

---

## Buyers (customers)

```python
partner_list_customers(filter={"business_name": {"eq": "Acme Corp"}}, limit=10)
partner_get_customer(merchant_customer_id="<id>")
```

Filters are **exact match** — there is no fuzzy search. Narrow progressively: full
string → shorter form → unfiltered `partner_list_customers(limit=25)` and scan.

### Creating a buyer — `merchant_id` depends on partner type

```python
# marketplace partner — the buyer belongs to you; merchant_id must be OMITTED
partner_create_customer(
  business_name="Acme Corp", business_address="456 Commerce Blvd",
  business_city="Austin", business_state="TX", business_zip="78701",
  business_country="US", business_ap_email="ap@acmecorp.com", email="owner@acmecorp.com"
)

# single-merchant partner — the buyer belongs to a sub-merchant; merchant_id REQUIRED
partner_create_customer(
  merchant_id="<sub_merchant_id>",
  business_name="Acme Corp", ...
)
```

Passing `merchant_id` as a marketplace partner is rejected, and omitting it as a
single-merchant partner is rejected. If you are unsure of the type, ask.

### Credit workflow

```python
partner_request_credit_check(merchant_customer_id="<id>")   # submit for underwriting
partner_enroll_customer(merchant_customer_id="<id>")        # after approval
```

A `204` means the request was **accepted**, not decided. Credit moves
`null → pending → approved` / `declined`, usually within one business day. Poll
`partner_get_customer` to check. Once `approved`, `net_terms_status` becomes
`pending_enrollment` and the buyer must finish enrollment before invoices can advance.

Credit status: `null` (not requested), `pending`, `approved`, `declined`, `hold`
(15+ days overdue), `deactivated`.

**Marketplace:** one credit decision covers every sub-merchant — a buyer approved for
$50K can spend it across any of them. **Single-merchant:** credit is per sub-merchant, so
check the buyer has a line with the *specific* sub-merchant before invoicing.

---

## Shipments

```python
partner_create_shipment(
  merchant_invoice_id      = "<invoice_id>",
  fulfillment_method       = "shipping_provider",   # or self_delivery, customer_pickup, services_only
  shipment_tracking_number = "1Z999AA10123456784",
  shipment_courier         = "ups"
)
partner_sync_shipment_tracking(shipment_id="<id>")   # pull live courier status
```

`shipment_tracking_number` and `shipment_courier` are required when
`fulfillment_method="shipping_provider"`. For the other methods Resolve expects a
proof-of-delivery file — attach it to the conversation and pass it through as the
document field.

Also available: `partner_list_shipments`, `partner_get_shipment`,
`partner_update_shipment`, `partner_delete_shipment`.

---

## Credit notes

```python
partner_create_credit_note(
  invoice_id     = "<invoice_id>",          # note: invoice_id, not merchant_invoice_id
  amount         = 150.00,
  number         = "CN-0042",
  reason_code    = "requested_by_customer",
  reason_message = "Buyer returned 3 units"
)
```

Valid `reason_code`: `chargeback`, `duplicate`, `fraudulent`, `requested_by_customer`,
`missing_remittance`, `overpayment_on_invoice`, `invoice_was_refunded`,
`double_payment_on_invoice`, `missing_invoice_in_resolve`, `credit_transfer`, `other`.

The invoice must be sent first. Reverse with `partner_void_credit_note`.

---

## Payments and payouts

```python
partner_list_payments(limit=25)
partner_get_payment(payment_id="<id>")
partner_create_payment(...)          # record a payment against an invoice
partner_list_payouts(limit=25)
partner_get_payout(payout_id="<id>")
partner_list_payout_transactions(limit=25)
partner_get_payout_transaction(payout_transaction_id="<id>")
```

Unlike a merchant account, a partner **can** record payments
(`partner_create_payment`). Payouts may be batched to the partner account or sent
directly to individual sub-merchants depending on the payout configuration set during
onboarding.

---

## Orders and charges are read-only

Partner API keys hold no order or charge write permissions. Only these exist:

```python
partner_list_orders(limit=25);   partner_get_order(order_id="<id>")
partner_list_charges(limit=25);  partner_get_charge(charge_id="<id>")
```

If the user asks to create, capture, or cancel an order or charge, explain that a
partner account cannot — it must be done by the sub-merchant on their own account.

---

## Webhooks

```python
partner_list_webhooks()
partner_upsert_webhook(
  endpoint_url = "https://yourserver.com/resolve-webhook",   # must be https
  events       = {                                          # object of booleans, not a list
    "invoice.created": True,
    "merchant.created": True,
    "merchant.underwriting_completed": True
  }
)
partner_delete_webhook(endpoint_url="https://yourserver.com/resolve-webhook")
```

Partner accounts get four **partner-only** topics on top of the shared ones:
`merchant.created`, `merchant.underwriting_completed`, `merchant.more_info_needed`,
`merchant.mal_updated` — use these to track sub-merchant onboarding and underwriting.

Shared topics: `invoice.created`, `invoice.balance_updated`, `customer.created`,
`customer.status_updated`, `customer.line_amount_updated`,
`customer.advance_rate_updated`, `customer.credit_decision_created`, `order.created`,
`order.updated`, `payment.created`, `payment.status_changed`, `payout.created`,
`payout.status_changed`.

Every event payload carries `data.merchant_id` identifying which sub-merchant it came
from. Verify the `x-webhook-signature` HMAC-SHA256 header.

---

## Common requests → tools

| Request | Tool(s) |
|---|---|
| "List my merchants" | `partner_list_merchants(limit=25)` |
| "Onboard a new merchant" | `partner_create_merchant(...)` |
| "Create an invoice for <sub-merchant>" | `partner_list_merchants` → `partner_create_invoice(merchant_id=..., merchant_invoice_url=<attachment>)` → `partner_update_invoice` → `partner_send_invoice` |
| "Find buyer Acme" | `partner_list_customers(filter={"business_name":{"eq":"Acme"}})` |
| "Run a credit check" | `partner_request_credit_check(merchant_customer_id=...)` |
| "Which invoices are unpaid?" | `partner_list_invoices(filter={"fully_paid":{"eq":False}}, limit=25)` |
| "Void an invoice" | `partner_void_invoice(merchant_invoice_id=...)` |
| "Add tracking" | `partner_create_shipment(...)` |
| "Issue a credit" | `partner_create_credit_note(invoice_id=...)` |
| "Show payouts" | `partner_list_payouts(limit=25)` |
| "Notify me when a merchant is onboarded" | `partner_upsert_webhook(events={"merchant.created": True})` |

---

## Async operations

A `200`/`204` means Resolve **accepted** the request, not that it finished. Credit
checks, invoice sends, advances, enrollment, and sub-merchant underwriting all complete
in the background. Tell the user it was submitted and re-fetch in a few minutes to
confirm.

---

## Error handling

- **401 authentication_error** — the key is invalid or the account is inactive.
- **403** — the partner permission set does not include this action (e.g. an order or
  charge write). Do not retry; explain the limitation.
- **404 not_found_error** — the ID doesn't exist *under your partner account*. A
  sub-merchant id from elsewhere will not resolve; re-check `partner_list_merchants`.
- **400 validation_error** — read `error.details`. Two common partner cases:
  `merchant_id` missing on `partner_create_invoice`, and `merchant_id` sent (marketplace)
  or omitted (single-merchant) on `partner_create_customer`.
- **`Invalid merchant_id`** — the id is not a sub-merchant of your partner account.
- **429 rate_limit_error** — back off and retry.
- **Invoice PDF download failure** — re-attach the file to the conversation and retry.

---

## Output style

- When listing anything cross-merchant, show which **sub-merchant** each record belongs
  to — it is the detail that matters most on a partner account.
- Show buyer name with `credit_status` and `amount_available` together.
- Show invoice number, amount, terms, and status together.
- Confirm before any destructive call (void, cancel, delete).
- Amounts are decimal dollars (e.g. `2500.00`), not cents — pass and display as-is.
