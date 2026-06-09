# FOLIO Invoices App — Business Logic Context

> This file is used by the `write-testrail-cases` skill to understand the business logic of the Invoices app before generating test cases.
> Sources: https://docs.folio.org/docs/acquisitions/invoices/

---

## What is Invoices

The Invoices app manages payments and credits for materials acquired by the library. An invoice records what is owed to a vendor, contains invoice lines (individual items being paid), and goes through an Approve → Pay workflow that drives Finance transactions.

---

## Invoice Lifecycle (Status)

```
Open / Reviewed → Approved → Paid
Open / Reviewed → Cancelled
Paid → Cancelled (only with special capability)
```

| Status | Description |
|---|---|
| **Open** | Default status on creation; editable |
| **Reviewed** | Optionally set before approval; still editable |
| **Approved** | Approved for payment; creates Pending payment transactions in Finance; editing is restricted |
| **Paid** | Payment confirmed; Payment transactions created in Finance; encumbrances released |
| **Cancelled** | Invoice voided; all transactions reversed |

---

## Invoice Header Fields

### Required fields
- **Invoice date** — vendor invoice date
- **Status** — Open (default) or Reviewed; can approve/pay from either
- **Batch group** — groups invoices for voucher export

### Key header fields
- **Vendor invoice number** — must be unique per vendor + date combination (duplicate check on save)
- **Vendor** — must be an active vendor organization; non-vendor org can be selected but Approve will fail
- **Fiscal year** — defaults to current FY; only editable by users with the `Invoice: Pay invoices in a different fiscal year` capability
- **Acquisition units** — controls access same as Orders/Finance
- **Payment due** — due date for payment
- **Lock total / Lock total amount** — when Lock total is checked, all invoice lines + adjustments must equal the locked amount before Approve is allowed
- **Export to accounting** — sends voucher to external financial system
- **Payment method** — Cash, Credit card, EFT, Deposit account, Physical check, Bank draft, Internal transfer, Other

### Calculated totals (system-generated, read-only)
- **Sub-total** — sum of all invoice line sub-totals
- **Total adjustments** — sum of all adjustments (invoice-level + line-level), excluding "Separate from" adjustments
- **Calculated total amount** — Sub-total + Total adjustments; if negative → credit transaction

---

## Invoice Lines

Each invoice line represents one item being paid. Lines are linked to POLs.

### Invoice line key fields
- **Description** — what is being paid for
- **Invoice line number** — auto-generated
- **Sub-total** — cost of this line before adjustments
- **Quantity** — number of units
- **Comment** — internal note
- **Fund distribution** — which fund(s) pay for this line and in what amount/percentage
- **Expense class** — sub-category within a fund
- **Linked POL** — purchase order line this invoice line is paying for (optional but common)
- **Adjustments** — line-level adjustments (tax, shipping, etc.)
- **Accounting code** — from vendor record, required if Export to accounting is on
- **Release encumbrance** — checkbox; when checked, the encumbrance on the linked POL is released when the invoice is approved (vs. when it's paid)

### Linking an invoice line to a POL
- User clicks "Add fund distribution from POL" or links via POL number
- Fund distribution from the POL is automatically pre-filled on the invoice line
- Linking updates the POL's Payment status when the invoice is approved/paid

---

## Adjustments

Adjustments are additional charges on top of the material cost (e.g. shipping, tax). Can be at invoice level or invoice line level.

### Adjustment fields
- **Description** — required
- **Value + Type** — amount (or percentage) of the adjustment; negative value = credit
- **Pro rate** — how the adjustment is distributed across invoice lines:
  - **By line** — equal share per line regardless of amount or quantity
  - **By amount** — proportional to each line's subtotal
  - **By quantity** — proportional to each line's quantity
  - **Not prorated** — applied at invoice level; requires specifying a fund
- **Relation to total**:
  - **In addition to** — added on top of subtotal; increases total
  - **Included in** — already bundled in subtotal; reduces material cost component
  - **Separate from** — NOT included in calculated total; reported but not paid via this invoice
- **Export to accounting** — include in voucher export

### Preset adjustments
Configured in Settings → Invoices → Adjustments. Can be marked "Always show" to appear on every invoice automatically.

---

## Approve Flow

### What triggers Approve
User action: Actions → Approve

### What Approve requires
- Invoice status is Open or Reviewed
- Vendor is an active vendor organization (not just any org)
- If Lock total is enabled: sum of lines + adjustments must equal lock total amount
- Budget must have sufficient funds (if Restrict expenditures is on the ledger)

### What happens on Approve
1. Invoice status → **Approved**
2. **Pending payment** transaction created in Finance for each fund distribution on each invoice line and invoice-level not-prorated adjustment
3. Budget **Awaiting payment** increases; **Encumbered** decreases proportionally (if linked to POL)
4. **Approval date** set to today's date
5. **Approved by** field populated with approving user's name
6. A **Voucher** is generated (for accounting export purposes)
7. POL **Payment status** updates to **Awaiting Payment** (for linked lines)

---

## Pay Flow

### What triggers Pay
User action: Actions → Pay

### What happens on Pay
1. Invoice status → **Paid**
2. **Payment** transactions created in Finance for each fund distribution
3. Budget **Expended** increases; **Awaiting payment** decreases
4. Remaining **Encumbrance** on linked POLs is released (unless "Release encumbrance" was already checked at Approve step)
5. POL **Payment status** updates to **Fully Paid** (for fully paid lines) or **Partially Paid**
6. If all POLs on the linked PO are now Fully Received + Fully Paid → PO auto-closes

---

## Cancel Flow

### What triggers Cancel
User action: Actions → Cancel

### What happens on Cancel
- All Finance transactions are **reversed** (credits applied back to budgets)
- Invoice status → **Cancelled**
- POL payment status reverts

### Cancelling a Paid invoice
- Requires the `Invoice: Cancel invoices` capability
- All payment transactions are reversed in Finance

---

## Voucher

A voucher is a payment request document generated when an invoice is Approved.

### Voucher fields
- **Voucher number** — system-generated
- **Voucher date** — date voucher was generated (= approval date)
- **Status** — Awaiting payment / Paid
- **Batch group** — inherited from invoice
- **Disbursement number** — can be entered manually after payment is processed externally
- **Disbursement date** — date external payment was made

### Voucher export
- Actions → Export vouchers → downloads XML or JSON for external financial system
- Export includes all invoices in the selected batch group within the specified date range
- Only invoices with "Export to accounting" checked are included

---

## Duplicate Invoice Detection

When saving a new invoice:
- If vendor + vendor invoice number + invoice date matches an existing invoice → popup warning: "This appears to be a duplicate invoice"
- User can choose to proceed (Submit) or cancel

---

## Finance Integration Summary

| Invoice action | Finance transaction created | Budget impact |
|---|---|---|
| Approve | Pending payment | Awaiting payment ↑, Encumbered ↓ |
| Pay | Payment | Expended ↑, Awaiting payment ↓ |
| Cancel (Approved) | Reversal of pending payment | Awaiting payment ↓ |
| Cancel (Paid) | Credit | Expended ↓ |
| Negative total (credit invoice) | Credit | Available ↑ |

---

## Required Capability Sets (Eureka)

| Action | Capability Set |
|---|---|
| View invoices | `Data - UI-Invoice Invoice - View` |
| Create/Edit invoices | `Data - UI-Invoice Invoice - Create`, `Edit` |
| Delete invoices | `Data - UI-Invoice Invoice - Delete` |
| Approve invoices | `Procedural - UI-Invoice Invoice Approve - Execute` |
| Pay invoices | `Procedural - UI-Invoice Invoice Pay - Execute` |
| Cancel invoices | `Procedural - UI-Invoice Invoice Cancel - Execute` |
| Pay in different fiscal year | `Procedural - UI-Invoice Invoice Pay-Different-Fy - Execute` |
| Export vouchers | `Procedural - UI-Invoice Voucher Export - Execute` |
| Download batch file | `Procedural - UI-Invoice BatchVoucher - Execute` |
| Assign acquisition units | `Procedural - UI-Invoice Acq Unit Assignment - Execute` |
| Manage acquisition units | `Data - UI-Invoice Acq Unit Assignment - Manage` |
| Export search results | `Procedural - UI-Invoice Export CSV - Execute` |

---

## Key Business Rules for Test Cases

1. **Default status on creation is Open** — not Draft; invoice is immediately editable after save
2. **Can approve/pay from both Open and Reviewed** — Reviewed is optional; not required before Approve
3. **Vendor must be an active vendor org** — selecting a non-vendor org blocks Approve with an error
4. **Duplicate check on save** — same vendor + vendor invoice number + invoice date triggers popup; user must confirm or cancel
5. **Lock total blocks Approve** — if Lock total is checked and line amounts don't sum to the lock total, Approve button is blocked or returns an error
6. **Approve creates Pending payment in Finance** — budget Awaiting payment increases immediately on Approve
7. **Pay creates Payment transaction** — Expended increases; Awaiting payment decreases; encumbrances released
8. **Negative total creates credit** — if Calculated total amount is negative, a Credit transaction is recorded in Finance
9. **"Separate from" adjustments excluded from total** — not counted in Calculated total amount or Total adjustments
10. **Not-prorated adjustments need a fund** — if Pro rate = Not prorated, a fund distribution must be specified for the adjustment
11. **Release encumbrance checkbox** — if checked on invoice line, encumbrance is released at Approve rather than at Pay
12. **Fiscal year defaults to current FY** — only users with the "Pay in different FY" capability see the dropdown and can select a prior FY
13. **Cancel reverses all transactions** — for Approved: pending payments reversed; for Paid: payments reversed (requires Cancel capability)
14. **POL payment status updates automatically** — Awaiting Payment on Approve, Fully Paid on Pay; drives PO auto-close
15. **Voucher generated on Approve** — payment request document for external accounting system; contains disbursement fields for manual entry after external payment
16. **Export to accounting controls voucher inclusion** — only invoices (and adjustments) with this checkbox checked are included in batch voucher export
17. **Accounting code required when Export to accounting is on** — if Export to accounting is selected but vendor has no accounting code, invoice cannot be approved
18. **Acquisition units gate access** — same as Orders/Finance; user must be in the assigned unit to view/edit
19. **Batch groups control voucher export grouping** — invoices are grouped and exported together by batch group; required field on invoice
20. **Credit invoice (negative total)** — when invoice total is negative, a Credit transaction is posted to Finance budgets; Available increases
