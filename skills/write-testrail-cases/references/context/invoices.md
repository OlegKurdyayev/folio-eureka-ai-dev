# FOLIO Invoices App - Business Logic Context

> Built: 2026-06-18 (Cypress-fragment refresh 2026-08-06; delta 2026-11-07) · Refreshed: 2026-11-25 · Release: Umbrellaleaf R2 2026 (ui-invoice 8.1.0 in progress; mod-invoice 6.1.x; Trillium R1 2026 = ui-invoice 8.0.0) · Builder: build-app-context (Refresh Mode) — see area-map.md → Freshness

> Generated/refreshed by the build-app-context workflow on 2026-06-18; **full refresh 2026-11-24**, audit remediation 2026-11-25 (source tags, capability names, TestRail case bodies read; Refresh Mode).
> Release basis: **Umbrellaleaf R2 2026** (ui-invoice master = 8.1.0 in progress; Trillium R1 2026 = ui-invoice 8.0.0 / mod-invoice 6.1.x, released 2026-04-17 per Jira UINV-632 / MODINVOICE-641). The Jira epics link the same releases under other labels (Trillium "R2 2025", Umbrellaleaf "R1 2026") — see Known Gaps.
> Sources:
> - TestRail suite 21: Invoices (329, 231 cases), Settings (Invoices) (1289, 26), Consortium (Invoices) (37957, 4), Permissions --> Invoices (1704, 12). Read-only; case ids cited as C…. Case bodies (preconditions/steps/expected) read so far: 50 of 273 (see Known Gaps for the list); the rest only by title/refs.
> - GitHub: folio-org/ui-invoice (package.json 8.1.0, CHANGELOG.md, translations/ui-invoice/en.json — identical to en_US.json, 511 keys), folio-org/mod-invoice (descriptors/ModuleDescriptor-template.json, ErrorCodes.java, PoLinePaymentStatusUpdateService.java, protected-fields classes, NEWS.md), folio-org/mod-invoice-storage (descriptor, ramls), folio-org/acq-models (mod-invoice-storage/schemas).
> - Jira folio-org: 181 Done UINV/MODINVOICE/MODINVOSTO issues resolved since 2024-06-01 scanned; ~45 read in detail.
> - docs.folio.org (GitHub source `content/en/docs/Acquisitions/Invoices`, `Settings/Settings_invoices`, `Additional topics/Invoice adjustments`, last dated 2026-05-15).
> - Confluence: "Process invoices against future fiscal years" (Acquisitions SIG), "Invoices" (Releases), Kafka audit design for Voucher (mod-invoice-storage).
> - stripes-testing (cypress/support/fragments/invoices, cypress/e2e/invoices — 173 specs) and folio-integration-tests (acquisitions/.../thunderjet/mod-invoice — 48 features).
> Strings marked [confirmed] appear in both GitHub source and TestRail signals.
> Strings marked [from source] come from GitHub source only (translations key named where useful).
> Strings marked [from cases] come from TestRail case corpus.

---

## What is Invoices

The Invoices app manages payments and credits for materials acquired by the library. An invoice records what is owed to a vendor, contains invoice lines (individual items being paid), and goes through an Approve → Pay workflow that drives Finance transactions.

Invoices behavior is tightly integrated with Orders and Finance. Approval and payment actions create and transition Finance transactions, update budget balances, and update linked POL payment status.

---

## Key Terms

| Term | Definition |
|---|---|
| Invoice | The header record representing a payable (or credit) document for a vendor. |
| Invoice line | A payable line item, often linked to a POL, with fund distribution and optional adjustments. |
| Voucher | Payment-request document generated on approval and used in export/accounting flows. |
| Pending payment | Finance transaction type created during Approve, before payment is finalized. |
| Payment transaction | Finance transaction type created during Pay, moving cost into Expended. |
| Lock total | Control requiring calculated invoice total to match a user-entered locked amount before Approve. |
| Release encumbrance | Option on line/payment flow controlling whether encumbrance releases on Approve vs Pay. |

---

## Invoice Lifecycle (Status)

```
Open / Reviewed → Approved → Paid
Approved → Cancelled
Paid → Cancelled
```

Only Approved and Paid invoices can be cancelled (BE error `cannotCancelInvoice`: "Only approved and paid invoices can be cancelled" — mod-invoice ErrorCodes.java; docs "Cancelling an invoice"). Open/Reviewed invoices are Deleted, not Cancelled. Paid can never transition to another status except via Cancel (`invalidInvoiceStatusTransitionOnPaidStatus`); an invoice cannot be paid without prior approval (`cannotPayInvoiceWithoutApproval`) unless "Approve and pay in one click" is on. Invoice line has its own `invoiceLineStatus` (Open, Reviewed, Approved, Paid, Cancelled, Error — acq-models invoice_line.json).

| Status | Description |
|---|---|
| **Open** | Default status on creation; editable |
| **Reviewed** | Optionally set before approval; still editable |
| **Approved** | Approved for payment; creates Pending payment transactions in Finance; editing is restricted |
| **Paid** | Payment confirmed; Pending payment transactions become Payment (or Credit) transactions in Finance (C357038, C350633); linked POL payment status updated at Pay time |
| **Cancelled** | Invoice voided; all related transactions voided; voucher status set to Cancelled; the invoice cannot be re-opened, approved or paid again (docs "Cancelling an invoice") |

---

## Record Hierarchy / Architecture

Invoices
- Invoice list and search
  - Invoice header
    - Invoice lines
      - Fund distributions
      - Line adjustments
    - Invoice-level adjustments
    - Voucher information
- Related integrations
  - Orders (POL/PO payment status)
  - Finance (pending payment, payment, credit/reversal)

---

## Invoice Header Fields

### Required fields

Source: docs Acquisitions/Invoices (Invoice information: "Invoice date, Status, and Batch group are the only required fields"); C2299.

- **Invoice date** — vendor invoice date
- **Status** — Open (default) or Reviewed; can approve/pay from either
- **Batch group** — groups invoices for voucher export

### Key header fields
- **Vendor invoice number** — duplicate check on save is on vendor + vendor invoice number + invoice date (docs; `invoice.isNotUnique.confirmation.message`)
- **FOLIO invoice number** — system-generated (`folioInvoiceNo`), searchable and exportable (UINV-556)
- **Source** — User, API, EDI (schema enum, acq-models invoice.json); docs also list MARC. Filterable.
- **Vendor** — must be an active vendor organization; non-vendor org can be selected but Approve will fail (ErrorCodes `organizationIsNotVendor`; C388558, C388559). The vendor's primary address is shown on create and view (C2299)
- **Fiscal year** — defaults to current FY; only editable by users with the `Invoice: Pay invoices in a different fiscal year` capability
- **Acquisition units** — controls access same as Orders/Finance; a user outside the invoice's unit gets `Operation is not permitted because user is not a member of the specified acquisition unit` (en.json `invoice.actions.error.userNotAMemberOfTheAcq`). Exception: an invoice created from an order whose unit restricts View can still be approved&paid / paid by a user who is not in that unit (MODINVOICE-516; C446069, C630444 — see Known Gaps). Applying a unit to a Paid invoice by a unit member with edit + manage-units capabilities works (C357051)
- **Bill to** — billing address from Settings > Tenant > Addresses (UINV-621 uses GET/PUT /tenant-addresses); dropdown options sorted alphabetically (UINV-601, C986308)
- **Currency / exchange rate** — default is the tenant primary currency; non-default shows `Current exchange rate`; `Use set exchange rate` + `Set exchange rate` override the provider rate (docs)
- **Enclosure needed**, **Check subscription overlap** (docs: logic not implemented — checkbox only), **Terms**, **Note**, **Approval date**, **Approved by**, **Payment date**, **Total units**, **Cancellation note** (entered in the Cancel dialog, optional)
- **Payment due** — due date for payment
- **Lock total / Lock total amount** — when Lock total is checked, all invoice lines + adjustments must equal the locked amount before Approve is allowed; the amount field is disabled until Lock total is checked (`invoice.lockTotalAmount.tooltip`: "Check Lock total to interact with this field")
- **Export to accounting** — sends voucher to external financial system
- **Payment method** — Cash, Credit card, EFT, Deposit account, Physical check, Bank draft, Internal transfer, Other

### Calculated totals (system-generated, read-only)
- **Sub-total** — sum of all invoice line sub-totals
- **Total adjustments** — sum of all adjustments (invoice-level + line-level), excluding "Separate from" adjustments and — since MODINVOICE-639/640 (mod-invoice 6.1.0) — excluding "Included in" adjustments (see Adjustments)
- **Calculated total amount** — Sub-total + Total adjustments; if negative → credit transaction

---

## Invoice Lines

Each invoice line represents one item being paid. Lines are linked to POLs.

### Invoice line key fields
- **Description** — what is being paid for
- **Invoice line number** — auto-generated, never reused after a line is deleted (MODINVOICE-647: delete line #2 of 2, the next new line gets #3; C389497, C410742)
- **Sub-total** — cost of this line before adjustments
- **Quantity** — number of units
- **Comment** — internal note
- **Fund distribution** — which fund(s) pay for this line and in what amount/percentage
- **Expense class** — sub-category within a fund
- **Linked POL** — purchase order line this invoice line is paying for (optional but common)
- **Adjustments** — line-level adjustments (tax, shipping, etc.)
- **Accounting code** — from vendor record, required if Export to accounting is on
- **Release encumbrance** — checkbox (`releaseEncumbrance`, required in schema); NOT checked by default on a new blank line (C389582). When checked, the encumbrance on the linked POL is released when the invoice is approved (docs; C350633: encumbrance "Released" with amount 0.00 right after Approve, before Pay)
- **Account number** — dropdown of the vendor organization's accounts; only active accounts are offered on create and edit (C413346, C413347); an already-selected inactive account stays displayed on edit (C411660; `invoiceLine.accountNumber.inactive`: "The selected account number is inactive."); a blank value can be chosen when the vendor has one active or one inactive account (C436930, C436933)
- **Vendor reference numbers / type**, **Subscription info / start / end** — subscription info, dates, comments and tags stay editable after Approve/Pay, also when the fund's budget is Inactive or the prior-FY budget is Closed (C350952, C919907, C919908, MODINVOICE-618)
- **Fund code** — auto-populated from fundId when missing (MODINVOICE-619, C934328)

### Invoice Lines table column headers (confirmed from `invoiceView.js` fragment, 2026-08-06)

The invoice lines table inside the Invoice detail pane shows these columns:
`POL number`, `Description`, `Fund code`, `Receipt status`, `Payment status`, `Quantity`, `Sub-total`, `Total`, `Total (Exchanged)`

Docs (Acquisitions/Invoices) list the full column set: `#`, `POL number`, `Description`, `Fund code`, `PO status`, `Receipt status`, `Payment status`, `Vendor reference number`, `Quantity`, `Sub-total`, `Adjustments`, `Release encumbrance`, `Total`, `Total (Exchanged)`, `Vendor code`; translation keys `invoice.details.lines.list.poStatus`, `.receiptStatus`, `.paymentStatus` confirm the status headers. "Folio invoice number" was removed from the invoice-line column (C350949). Invoice-line view also shows an `Other related invoice lines` accordion (columns incl. Vendor invoice #, Invoice line #, Fiscal year, Status, Quantity, Amount, subscription dates — docs; C409409; paginated C397995) and a Receiving history (C350391, C350908 past payments for the POL).

- `Total (Exchanged)` column appears when the invoice currency differs from the tenant base currency (multi-currency scenario). In same-currency scenarios this column is absent or blank. Use verbatim when asserting column presence in currency-mismatch tests.

### Linking an invoice line to a POL
- User clicks `Actions > Add line from POL` in the Invoice lines accordion (Select order lines modal, with Order format facets such as Physical Resource) or links via POL number (C2327, C353566)
- Fund distribution and details from the POL are automatically pre-filled on the invoice line (C2327); a confirmation pop-up (`Confirm`) appears when the POL currency or vendor differs from the invoice (C2327, C350741)
- Linking updates the POL's Payment status when the invoice is **paid** (not at Approve — see Pay Flow). A POL whose payment status is `Fully paid` or `Payment not required` cannot be added from the PO search tab (C357026, DRAFT); in the Add-line-from-POL flow a red banner and red exclamation icon alert when the POL is already Fully paid (docs). Changing the invoice currency while creating from an order shows a warning modal `Confirm currency change` / `You are changing the currency of this invoice but the amounts are not being exchanged from the POL currency to the new invoice currency. If you would still like proceed please click confirm.` (UINV-579, C808485, en.json `invoice.form.fromPO.currency.confirmModal.*`). POL currency is NOT overwritten by the vendor's currency when creating the invoice from a PO (C451593); invoice currency is taken from the related order (C440073, C440076) or from the vendor's `Vendor currencies` (C451451).

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
  - **Included in** — documents an amount already bundled in the price. **Conflict (see Known Gaps):** docs (Invoice adjustments page, 2024-12) say the subtotal is reduced (10% of $10 -> $0.91, subtotal $9.09), but MODINVOICE-639/640 (mod-invoice 6.1.0, 2026-03) reverted that: "Included in" adjustments are NOT used in line sub-total, Total adjustments, invoice sub-total or Calculated total amount (Expected: $10 line + $5 Included-in -> Total adjustments $0, Calculated total $10). Verified formula in the MODINVOICE-640 comment: 10% Included-in on a $20 line -> adjustment amount $1.82 = (20*10)/(100+10), subtotal unchanged, not shown in Total adjustments
  - **Separate from** — NOT included in calculated total; reported but not paid via this invoice
- **Export to accounting** — include in voucher export

### Preset adjustments
Configured in Settings → Invoices → Adjustments (`Description`, `Type` Percent/Amount, `Always show`, `Value`, `Pro rate`, `Relation to total`, `Export to accounting`). Can be marked "Always show" to appear on every new invoice automatically; the user can still remove it on the invoice (docs Settings_invoices). Docs list only "In addition to" and "Separate from" for presets, while the schema enum has three values (see Known Gaps).

### Adjustment rules from bugs
- An invoice-level prorated adjustment appears on every invoice line; on the invoice-line edit form its fields are read-only and cannot be changed, only at invoice level (UINV-630, C1292041). Server rejects adding/removing a prorated adjustment on a line when it is not/is present at invoice level (`cannotAddAdjustment`, `cannotDeleteAdjustment`; adjustment ids must be unique `adjustmentIdsNotUnique`).
- Non-prorated adjustment needs a fund distribution that sums to 100% / the adjustment amount (`adjustmentFundDistributionsNotPresent`, `adjustmentFundDistributionsSummaryMismatch`; UI: `At least one fund distribution should be present for every non-prorated adjustment` — en.json `invoice.actions.approve.error.adjustmentFundDistributionsNotPresent`).
- Two invoice-level adjustments with the same amount and expense class can be approved (C710367) and a paid invoice with two invoice-level adjustments can be cancelled (C710366). Invoice-level adjustment + invoice line multi-fund distributed by amounts gives a specific error on Approve (C407745). Approve-and-pay with an invoice-level adjustment whose fund differs from the order fund works (C825243; MODINVOICE-600).
- Adjustments are displayed in the order they were added (MODINVOICE-581). Adjustment combinations are validated on the create form ("disallows combinations", C359172; Cypress check-disallowed-combinations-in-invoice-adjustments-when-creating-invoice.cy.js).
- Prorate By amount example (docs): two lines $50 and $100, $10 adjustment -> $3.33 / $6.67; By line -> $5 / $5; By quantity (5 and 4 units) -> $5.56 / $4.44.

---

## Approve Flow

### What triggers Approve
User action: Actions → Approve

### What Approve requires
- Invoice status is Open or Reviewed
- Vendor is an active vendor organization (not just any org)
- If Lock total is enabled: sum of lines + adjustments must equal lock total amount
- At least one invoice line exists, and every line (and every non-prorated adjustment) has fund distributions that total 100% / the sub-total (docs; `fundDistributionsNotPresent`, `incorrectFundDistributionTotal`)
- Funds belong to one fiscal year (`Invoice cannot be approved because the funds used belong to multiple fiscal years`; `moreThanOneFiscalYearSeries` for different FY series)
- Related orders are not in Pending workflow status (en.json `invoice.details.hasPendingOrders`; C397321, C397326)
- User has `UI-Invoice Invoice Approve` capability (BE `userHasNoInvoiceApprovePermission`)
- Budget restrictions are validated by mod-finance-storage only (MODINVOICE-631: FundAvailabilityHolderValidator removed): expenditure restriction (formula includes credited: (allocated + netTransfers) * allowableExpenditure - (awaitingPayment + expended - credited), MODINVOICE-547) -> `Fund distribution amount exceeds the allowable expenditure amount in the <b>{fundCode}</b> fund.` (en.json `errors.budgetRestrictedExpendituresError`); encumbrance restriction -> `Fund distribution amount exceeds the allowable encumbrance amount in the <b>{fundCode}</b> fund.` (`errors.budgetRestrictedEncumbranceError`, 422 code `budgetRestrictedEncumbranceError`; C1231686, C1241224, UINV-623). The same errors appear when Edit changes the exchange rate of an Approved invoice (C1231686 step 2)

### What happens on Approve
1. Invoice status → **Approved**
2. **Pending payment** transaction created in Finance for each fund distribution on each invoice line and invoice-level not-prorated adjustment
3. Budget **Awaiting payment** increases; **Encumbered** decreases proportionally (if linked to POL)
4. **Approval date** set to today's date
5. **Approved by** field populated with approving user's name
6. A **Voucher** is generated (for accounting export purposes)
7. If an invoice line has **Release encumbrance** checked, the linked encumbrance is released and shows `Released` with amount 0.00 (C350633; docs). This also holds when the line sub-total (96.99) is below the PO encumbrance (98) with encumbrance/expenditure limits enforced: Approve succeeds, encumbrance `Released`, Awaiting payment 96.99, Initial encumbrance 98 (C399084, MODINVOICE-477)
8. The POL **Payment status is NOT changed at Approve**: since MODINVOICE-585 (mod-invoice 6.1.0, PR 531) the POL status update moved from Approve to Pay. Docs ("Approving an invoice") still state it becomes "Partially paid" on approve — outdated, see Known Gaps
9. Approve is atomic: if voucher creation or a later step fails, pending payments are rolled back (MODINVOICE-519; MODINVOICE-626 — Approve & pay on a fund changed from inactive to active used to leave an orphan pending payment, now rolled back with `Invoice was not approved/paid`); the invoice stays Open and the toast is `Invoice was not approved` (C1009057)
10. Budget effect with a credit line (3 lines on one POL: +10, -10, +10; release encumbrance off): after Approve Available and Cash balance are unchanged, Encumbered 10, Awaiting payment 10, Unavailable 20 and 3 `Pending payment` transactions exist; after Pay Cash balance is 10 lower, Expended 20, Credited 10 (Credited since Ramsons; Expended was 10 before), transactions are 2 `Payment` + 1 `Credit`, all with source `invoice` (C347897)
11. After Approve the voucher is shown in the Voucher accordion with status `Awaiting payment`, and the Pending payment transaction has Amount = line value and Source = invoice number (C10945); after Pay voucher status is `Paid` and the Payment transaction has the same Amount/Source (C3453). Expense classes: two fund distributions of the same fund with different expense classes create a Pending payment per expense class, turned into Payments on Pay (C15859)
12. No pending payment / voucher is created when the PO line's fund has an Inactive budget and the invoice uses another fund — approval fails with `Invoice was not approved`, status stays Open, no Pending payment/Payment on the invoice fund, encumbrance on the PO fund stays Unreleased (C1009057, C1009058, MODINVOICE-626)

---

## Pay Flow

### What triggers Pay
User action: Actions → Pay

### What happens on Pay
1. Invoice status → **Paid**
2. **Payment** transactions created in Finance for each fund distribution
3. Budget **Expended** increases; **Awaiting payment** decreases
4. Remaining **Encumbrance** on linked POLs is released only for lines with Release encumbrance checked (that release already happened at Approve — C350633: after Pay the fund shows $0 encumbrance, expended $16, status Released)
5. POL **Payment status** is updated at Pay by mod-invoice `PoLinePaymentStatusUpdateService`: **Fully Paid** if any invoice line on that POL has Release encumbrance = true, otherwise **Partially Paid**; POLs whose status is **Ongoing** or **Payment Not Required** are never changed (MODINVOICE-544, C466210, C466211)
6. If all POLs on the linked PO are now Fully Received + Fully Paid → PO auto-closes (docs "Paying an invoice": the order moves to Closed once fully received)
7. A total of 0 (e.g. one fund distribution negative) is payable: Pending payment per fund is replaced by Payment transactions for positive amounts and a Credit for the negative ones (C357038, C357042, C357044; Karate pay-invoice-with-0-value.feature)
8. Pay requires `UI-Invoice Invoice Pay` (`userHasNoInvoicePayPermission`). Approve and Pay can be separate actions only when Settings > Invoices > Approvals > "Approve and pay in one click" is off; the action then reads `Approve & pay`

### Paying / approving / cancelling against a previous fiscal year
- Needs `UI-Invoice Invoice Pay-Different-Fy` to see and change the Fiscal year dropdown (all past and current FY codes, newest first — docs; C387522, C387533); the FY field is read-only for Approved, Paid and Cancelled invoices (C387534, C387536, C387537). BE `cannotResetInvoiceFiscalYear`: FY cannot be set back to null once defined. Budgets of the past FY must allow the transaction (Active, enough funds; Closed -> C396376, Frozen/Inactive on cancel -> C422008, C422009).
- When the invoice is for a previous FY AND at least one line has Release encumbrance = true AND a related POL belongs to an Open **One-time** order, the UI opens the `Update order status` modal on Pay, Approve & pay and Cancel (UINV-577, UINV-600); the choice is sent as BE parameter `poLinePaymentStatus` (values: No Change, Awaiting Payment, Partially Paid, Fully Paid, Cancelled; MODINVOICE-573/585). Without the parameter the BE returns `poLinePaymentStatusNotPresent`. The modal is NOT shown when the order is not Open, or no line releases the encumbrance (C703334, C703346). For lines with Release encumbrance = false in the same payment, POLs currently `Awaiting Payment` of open one-time orders become `Partially Paid`, others unchanged (MODINVOICE-587).
- Results: No change -> POL status untouched (C700852); Awaiting payment (C700857); Partially paid for two orders (C700858); cancel -> Fully paid (C700861); Cancelled for one POL (C709266).

---

## Cancel Flow

### What triggers Cancel
User action: Actions → Cancel

### What happens on Cancel
- All Finance transactions are **voided** (Pending payment / Payment / Credit); budgets are restored
- Invoice status → **Cancelled**; voucher status → Cancelled
- The dialog has an optional **Cancellation note**, displayed in the Invoice information accordion
- POL payment status reverts only for POLs that are `Fully Paid` or `Partially Paid` and whose invoice was **Paid** with a known fiscal year: if no other Paid invoice line in the same FY references the POL it becomes `Awaiting Payment` (C466207), if another paid invoice exists it becomes `Partially Paid` (C466208) or stays `Fully Paid` (C466209); `Payment not required` and `Ongoing` never change (C466210, C466211; PoLinePaymentStatusUpdateService). Cancelling an Approved invoice does not change POL payment status
- Encumbrances: cancelling restores the initial encumbrance amount (C399092, C400612, C400614, C400615, C400618; MODINVOICE-613/616/628); an encumbrance of a Cancelled PO line stays Released (MODINVOICE-632, C1152357); Ongoing $0 order scenarios C825437, C829881, C844253, C844254, C844259, C844263, C844264, C864744; release-encumbrance=false scenarios C852110, C870004, C877073, C889713-C889715, C1028980-C1028985; manual unrelease scenarios C1045390, C1045391, C1045392, C1045395, C1045396
- Can be cancelled after the PO line's fund distribution was changed (MODINVOICE-615, C895660); transactions are voided only for the invoice's own funds — an Inactive/Closed budget of the POL's other fund does not void Pending payment/Payment of the invoice fund and the invoice stays Approved/Paid with an error (MODINVOICE-632; C1045977, C1045991)
- Cancelling an Approved invoice with a release-encumbrance line: before Cancel the encumbrance is `Released` with Awaiting payment = unit price; after Cancel it is `Unreleased` with Awaiting payment 0.00, the Pending payment amount is shown crossed out with a `Voided transaction` tooltip, toast `Invoice has been cancelled successfully` (C350728)
- Invoices with >90 lines can be cancelled (C375062); >50 lines approve & pay (C446075); >1000 lines export (C700851)

### Cancel permission
- Cancelling ANY invoice (Approved or Paid) requires `Invoice: Cancel invoices` (`UI-Invoice Invoice Cancel`; BE `userHasNoInvoiceCancelPermission`) — it can be assigned to a user (C350624)

---

## Voucher

A voucher is a payment request document generated when an invoice is Approved.

### Voucher fields
- **Voucher number** — system-generated
- **Voucher date** — date voucher was generated (= approval date)
- **Status** — `Awaiting payment` / `Paid` (UI keys `voucher.status.awaitingPayment`, `voucher.status.paid`); schema enum also has `Cancelled` (set when the invoice is cancelled, docs)
- **Type** — Payment, Pre-payment, Credit, Voucher (schema enum); **Exchange rate**, **Total**, vendor address (primary address from the Organization), **Account number**, **Accounting code**, **Enclosure needed**
- **Batch group** — inherited from invoice
- **Disbursement number / date / amount** — entered manually via View voucher > Actions > Edit after payment is processed externally; these (plus the voucher number when Settings > Invoices > Voucher number > "Allow voucher number to be edited" is on, C10941) are the only editable voucher fields (docs).
- **Voucher lines** — grouped by fund external account number (fund + expense-class extension, UINV-565/566, C423429): several invoice lines with the same external account yield ONE voucher line. The `Group` column was removed from voucher lines (UINV-629, C1152357). Table columns: Line number, Fund code, External account number, Amount, total at the bottom
- Printing: View voucher > Actions > Print

### Voucher export
- Invoices list > Actions → **Voucher export** (capability `UI-Invoice Voucher Export`; docs) → choose a batch group → **Run manual export**; confirm text `All vouchers created since last export for the batch group {selectedBatchGroupName} will be exported and this process cannot be reversed` (en.json `settings.actions.manualExport.message`). There is no date-range selector: the export takes every not-yet-exported voucher of the group (docs "Exporting vouchers")
- The export history table lists Date, Status (Pending, Generated, Uploaded, Error), Message (e.g. `Uploaded successfully to FTP location`) and a download arrow; file format comes from the batch group configuration (JSON or XML; if unset `File format not specified for voucher export. Please update Batch Group Configuration in system settings`)
- Run manual export is disabled until a batch group is selected; after Continue the toast is `Voucher export has been started successfully` and the history table (Date, Status, Message, download icon, paginator) appears; the `The list contains no items` text shows when the group has no exports (C442851, C397985, C10943, C353220). With cleared FTP credentials the row is `Generated` / `Credentials for FTP upload were not found` and download still works (toast `Vouchers have been successfully downloaded`, en.json `voucherExport.download.success`); with valid SFTP/XML the row is `Uploaded` / `Uploaded successfully to FTP location`; Status `Error` is possible when no valid FTP location is defined (C10943). Without the Voucher export capability the Invoices Actions menu has no export option (C353222). With only the vendor's default Accounting code the XML has `<accountingCode>` and no `<accountNo>` (C397985). Voucher > Actions > Print offers a portrait/landscape Layout dropdown (C442851).
- Only invoices with "Export to accounting" checked are included; unchecked invoices/adjustments are not exported or downloadable (C396387, C397317). Open and Reviewed invoices have no batch file to download (C396388, C396389)
- Voucher export dates are human-readable timestamps in both JSON and XML (MODINVOICE-595/601 regressions to epoch long; C825246, C825247). Export with cleared credentials (C442851); export with many lines (Karate batch-voucher-export-with-many-lines.feature). Invoice-level `Voucher export details` accordion: Batch group, Batch file name, Batch file status (Uploaded / Error) and a download icon (docs; C353217, C396367, C396358, C396369)

---

## Duplicate Invoice Detection

When saving a new invoice:
- If vendor + vendor invoice number + invoice date matches an existing invoice → popup `Save invoice` with message "This appears to be a duplicate invoice"; the popup carries information about and a link to the other invoice (docs; en.json `invoice.duplicateInvoice.title` = "Possible duplicate invoice")
- User can choose to proceed (Submit) or cancel
- The same duplicate check runs on `Approve & pay` (C440075)

---

## Finance Integration Summary

| Invoice action | Finance transaction created | Budget impact |
|---|---|---|
| Approve | Pending payment | Awaiting payment ↑, Encumbered ↓ |
| Pay | Payment | Expended ↑, Awaiting payment ↓ |
| Cancel (Approved) | Pending payment transactions voided | Awaiting payment ↓ (docs "Cancelling an invoice": all transactions against funds are voided) |
| Cancel (Paid) | Payment/Credit transactions voided | Expended ↓ (C350728 "voids payments/credits and Unreleases encumbrances") |
| Pay with total = 0 | Payment for positive parts, Credit for negative parts | C357038 |
| Negative total (credit invoice) | Credit | Available ↑ (C343209; credited is subtracted in the expenditure-restriction formula, MODINVOICE-547) |

---

## Settings > Invoices

Source: docs `Settings_invoices.md`, en.json `settings.*`, TestRail section 1289.

- **Approvals**: checkbox "Approve and pay in one click"; when on, the invoice action is `Approve & pay`, the confirmation is still shown and invoice + voucher become Paid (docs; C10937). With it off, Approve and Pay are separate (C388558 vs C388559, C397326 vs C397321 verify both modes).
- **Adjustments** (presets): New / edit / delete (`Delete {description}?` / `Delete adjustment?`; toasts `The adjustment <b>{name}</b> was created.`, `... was updated.`, `The adjustment was deleted.`, errors `Failed to create adjustment.`, `The adjustment was not deleted.`; C10947). Stored via invoice-storage `adjustment-presets` API.
- **Batch groups**: tenant ships with one system group `FOLIO`; it can be renamed but not deleted (docs; C10938); a group in use on invoices cannot be deleted (docs; C367942); create/edit (C343345); delete toast `The batch group <b>{term}</b> was successfully <b>deleted</b>`.
- **Batch group configuration**: Batch group, Location type (FTP / SFTP), Upload location (blank = file downloads to the user's computer), Port, Directory, Format (JSON / XML), Username, Password, `Show credentials`/`Hide credentials`, `Test connection` (toasts `Connection is valid`, `Connection error: {cause}`; C414961, C414963); save toasts `Batch group configuration has been successfully saved` / `Batch group configuration was not saved` (C10939). Credential permission tiers: see "Batch Group Credentials". `ui-invoice.batchVoucher.exportConfigs.credentials.view/edit` now also grant `settings.invoice.enabled` so Settings opens (UINV-608, UINV-593).
- **Voucher number**: Prefix (letters only — `Voucher number prefix must contains only Unicode letters`), Starting number, `Reset sequence`, checkbox "Allow voucher number to be edited" (C10941). Prefixes and numbering moved from mod-configuration to the internal invoice-storage settings API (UINV-607, MODINVOICE-604, MODINVOSTO-208) — no mod-configuration dependency remains (UINV-616). Failed start number save: `Start number could not be saved.`
- A "default value for Lock total" setting (UINV-463/464) is NOT present in docs, package.json or en.json — treat as not implemented.
- Invoice settings in tenant A do not apply in tenant B (ECS): saving `Approve and pay in one click` shows toast `Setting was successfully updated.` in tenant A, where Approve & pay works (toast `Invoice has been approved and paid successfully`); in tenant B with the box unchecked the invoice Actions menu has only Edit and Approve (C353608).

## Search & Filter pane (Invoices)

Source: docs "Searching for an invoice"; en.json `search.*`; TestRail C6723, C6724.

- Search options (dropdown, default All): Voucher number, Vendor invoice number, FOLIO invoice number (UINV-556), PO number, Accounting code. Result columns include Vendor name and Vendor invoice number (UINV-591); "Vendor invoice number" value is a hyperlink (C369085). Total count shows a `~` prefix above 50k results (UINV-554). Page titles verified in C451485, C451605.
- Filters: Status (Open, Reviewed, Approved, Paid, Cancelled), Vendor name, Invoice date, Acquisition unit, Tags, Payment due, Payment method, Approval date, Source (User, API, EDI, MARC), Export to accounting (Yes/No), Payment date, Batch group, Fund code (multi-select, UINV-548, C590839, C353228), Expense class, Lock total (From/To), Total amount (From/To; UINV-341, C388648, C353231, C353234; "Invoice Total Amount filter returns results outside range" UINV-635 closed as bad reference data), Fiscal year (C388539), Created by / Date created / Updated by / Date updated (C466116-C466118, UINV-494 "Created by") and the same four for Invoice line (C466120, C466122, C466123). Date created filtering honours the selected time zone (C466118, C466123). Tags filter fixed to avoid global OR (UINV-603). Applying a date range filter must not clear other filters (Cypress applying-date-range-filter-does-not-clear-other-filters.cy.js).
- Expense class dropdowns in fund distributions (and adjustments) offer only classes related to the invoice's fiscal year (C396400, C397319).

## Export results (CSV)

Source: docs "Exporting search results"; C1332520, C466125, C773245, C700851, C421999.

- Actions > Export results (CSV) opens `Export settings` with message `This export could take a few minutes. If you reload or close the page the export will not be completed. Once the file is ready it could take another minute for your browser to finish downloading the file. You can continue to work with invoices and invoice lines in a different browser tab if needed.` (en.json `exportSettings.message`); toast `Export has been started successfully`; error `Failed to load data for export results (CSV)`. Choose All or selected Invoice fields / Invoice line fields.
- Includes `Invoice fund distributions` column (UINV-604/605, C773245), Created by / Updated by columns (C466125), FOLIO invoice number (UINV-556), External account number including the expense-class extension (UINV-565, C423429). Invoices in currencies unsupported by the ECB provider still export with and without lines (C421999). Limits were raised for large exports (UINV-572; C700851: >1000 lines).
- Capability: `UI-Invoice Export CSV` (`ui-invoice.exportCSV`).

## API surface (mod-invoice 6.1.x, descriptors/ModuleDescriptor-template.json)

Interface `invoice` 7.0 (ui-invoice requires `invoice 7.0`, `voucher 2.1`, `batch-voucher.* 1.0/2.0`, `invoice-storage.settings 1.0`, `invoice-storage.adjustment-presets 1.0`, `finance.exchange-rate 1.0`, `tenant-addresses 1.0`, `orders 13.0`, `order-lines 4.0` — ui-invoice package.json). `permissionsRequired` values:

| Endpoint | Method | permissionsRequired |
|---|---|---|
| /invoice/invoices | GET / POST | invoice.invoices.collection.get / invoice.invoices.item.post |
| /invoice/invoices/{id} | GET / PUT / DELETE | invoice.invoices.item.get / .put / .delete (Approve/Pay/Cancel are PUT with a new `status`; optional query `poLinePaymentStatus`) |
| /invoice/invoice-lines, /invoice/invoice-lines/{id} | GET POST PUT DELETE | invoice.invoice-lines.collection.get / item.post / item.get / item.put / item.delete |
| /invoice/invoice-number | GET | invoice.invoice-number.item.get |
| /invoice/invoices/{id}/documents[/{documentId}] | GET POST DELETE | invoice.invoices.documents.collection.get / item.post / item.get / item.delete |
| /invoice/invoices/{id}/fiscal-years | GET | invoice.invoices.fiscal-years.collection.get |
| /invoice/invoice-lines/fund-distributions/validate | PUT | invoice.invoice-lines.fund-distributions.validate |
| /voucher/vouchers[/{id}], /voucher/voucher-lines[/{id}] | GET, PUT | voucher.vouchers.collection.get / item.get / item.put, voucher.voucher-lines.collection.get / item.get / item.put |
| /voucher/voucher-number/start[/{value}] | GET / POST | voucher-number.start.get / .post |
| /batch-voucher/export-configurations[/{id}] and /credentials, /credentials/test | CRUD | batch-voucher.export-configurations.* , .credentials.item.post/get/put, .credentials.test |
| /batch-groups[/{id}] | CRUD | batch-groups.collection.get / item.* |
| /batch-voucher/batch-voucher-exports[/{id}], /scheduled, /{id}/upload; /batch-voucher/batch-vouchers/{id} | CRUD / POST / GET | batch-voucher.batch-voucher-exports.*, ...scheduled.item.post, ...upload.item.post, batch-voucher.batch-vouchers.item.get |

mod-invoice-storage: `invoice-storage.invoices 5.0`, `invoice-lines 4.0`, `voucher-storage.vouchers 1.2`, `voucher-lines 2.1`, `batch-group-storage.batch-groups`, `invoice-storage.adjustment-presets 1.0`, `invoice-storage.settings 1.0` (voucher number prefix/start, replaces mod-configuration), `POST /invoice-storage/audit-outbox/process` (timer; Kafka audit events for Invoice and InvoiceLine — MODINVOSTO-187/188; Voucher create/edit events designed under MODINVOSTO-218/219, Confluence design doc). `GET /finance/calculate-exchange` and `POST /finance/calculate-exchange-batch` (mod-finance) provide exchange amounts (UINV-626; MODINVOICE-579).

Protected fields: these invoice fields cannot be modified on an already approved invoice (exact status set not verified in code; Karate feature name says "approved invoice") (HTTP 400 `protectedFieldChanging`, "Field can't be modified"): adjustments, adjustmentsTotal, approvedBy, approvalDate, chkSubscriptionOverlap, currency, exportToAccounting, folioInvoiceNo, invoiceDate, lockTotal, paymentTerms, source, voucherNumber, paymentId, poNumbers, vendorId, subTotal, total (InvoiceProtectedFields.java); invoice-line protected fields: adjustments, adjustmentsTotal, invoiceId, invoiceLineNumber, poLineId, productId, productIdType, quantity, subTotal, total (InvoiceLineProtectedFields.java). Karate: check-that-changing-protected-fields-forbidden-for-approved-invoice.feature. Lines cannot be added to an approved invoice (`prohibitedInvoiceLineCreation`; check-that-not-possible-add-invoice-line-to-approved-invoice.feature); Approved or Paid invoices cannot be deleted (`approvedOrPaidInvoiceDeleteForbiddenError`; docs: delete only in Open/Reviewed). Document upload: attach and remove files on Approved and Paid invoices is allowed (C360544, C360546, C360547, C15856); too-large file -> `File exceeds the size limit of {size}MB` (`documentIsTooLarge`).

Backend error codes seen by approve/pay/cancel (mod-invoice ErrorCodes.java): accountingCodeNotPresent, fundDistributionsNotPresent, incorrectFundDistributionTotal, cannotMixTypesForZeroPrice ("Fund distribution types cannot be mixed for a price of 0"), lockCalculatedTotalsMismatch, organizationIsNotVendor, organizationIsNotExist, fundCannotBePaid, inactiveExpenseClass, budgetExpenseClassNotFound, budgetNotFoundByFundId, budgetNotFoundByFundIdAndFiscalYearId, multipleFiscalYears, moreThanOneFiscalYearSeries, couldNotFindValidFiscalYear, currentFYearNotFound, externalAccountNoIsMissing, pendingPaymentCreationError / pendingPaymentUpdateError, transactionCreationFailure, cancelTransactionsError, errorUnreleasingEncumbrances, poLineUpdateFailure, poLinePaymentStatusNotPresent, userNotAMemberOfTheAcq, userHasNoFiscalYearUpdatePermission, userHasNoAcqUnitsPermission.

---

## Main Sections / UI Structure

### Panes (from ui-invoice source / TestRail pane names)
- Invoices [from source]
- Invoice details [from source]
- Voucher information [from source]
- Invoice line details [from source]

### Accordions
- Adjustments [from source]
- Voucher lines [from source]

### Key Navigation Paths
- Apps > Invoices
- Invoices > Actions > Approve
- Invoices > Actions > Pay
- Invoices > Actions > Cancel
- Invoices > Actions > Voucher export (docs "Exporting vouchers"; C442851)
- Settings > Invoices > Adjustments
- Settings > Invoices > Vouchers
- Settings > Invoices > Approvals

---

## Exact UI Texts

> Use these verbatim in expected results. Never paraphrase.
> ✅ Toasts, dialogs and error messages below were **confirmed verbatim against `folio-org/ui-invoice/translations/ui-invoice/en_US.json` on 2026-07-21** — every previously `[from source]` string matched the translation file exactly. `<b>…</b>` marks bold segments; `{placeholders}` interpolate at runtime.

### Toast Messages

| Event | Toast text | Source |
|---|---|---|
| Approve success | Invoice has been approved successfully | [confirmed] |
| Pay success | Invoice has been paid successfully | [confirmed] |
| Approve and pay success | Invoice has been approved and paid successfully | [confirmed] |
| Cancel success | Invoice has been cancelled successfully | [confirmed] |
| Invoice created | Invoice has been created | [confirmed] |
| Invoice saved | Invoice has been saved | [confirmed] |
| Invoice deleted | Invoice has been deleted | [confirmed] |
| Invoice duplicated | The invoice has been duplicated successfully | [confirmed] |
| Save duplicate warning | This appears to be a duplicate invoice | [confirmed] |
| Invoice line saved | Invoice line has been saved | [confirmed — C343209 + source] |
| Invoice line deleted | Invoice line has been deleted | [confirmed] |
| Invoice line relinked | Invoice line has been linked to POL {poLineNumber} | [confirmed] |
| Voucher save success | Voucher has been successfully saved | [confirmed] |
| Manual export started | Voucher export has been started successfully | [confirmed] |
| Manual export error | Vouchers were not exported | [confirmed] |
| Export results CSV started | Export has been started successfully | [confirmed] |

### Approve / Pay failure messages (negative-case gold — all [confirmed])

| Condition | Message text |
|---|---|
| Vendor has no accounting code | `Invoice can not be approved (paid), because the identified vendor must have an accounting code for this invoice to be exported to accounting` |
| Fund has no current budget | `Invoice cannot be approved because Fund <b>{fundCode}</b> has no current budget.` |
| Fund has no budget for FY | `Invoice cannot be approved because Fund <b>{fundCode}</b> has no current budget for fiscal year <b>{fiscalYear}</b>.` |
| Not enough money in fund (legacy; since Trillium the budget-restriction messages below replace it — C375959) | `One or more Fund distributions on this invoice can not be paid, because there is not enough money in <b>{fundCodes}</b>.` (en.json `invoice.actions.approve.error.fundCannotBePaid`) |
| Inactive expense class | `Invoice can not be <b>Approved</b> because expense class <b>{expenseClass}</b> is inactive.` |
| Fund distribution not 100% | `Fund distributions summary should be 100 % or equal to subtotal for every associated invoice lines` |
| Missing fund distribution | `At least one fund distribution should present for every associated invoice line` |
| Related order still Pending | `Invoice can not be approved. One or more of the related orders has a workflow status of "Pending". If desired, please open the related orders before approving this invoice.` |
| Vendor is inactive | `Vendor is inactive. Invoice cannot be approved or paid.` |
| Lock total mismatch | `Invoice cannot be approved. The Lock total amount of the invoice does not match the Calculated total amount of its invoice lines and adjustments.` |
| Multiple fiscal years | `Invoice cannot be approved because the funds used belong to multiple fiscal years` |
| Not acq-unit member | `Operation is not permitted because user is not a member of the specified acquisition unit` |
| Budget restricted encumbrance | `Fund distribution amount exceeds the allowable encumbrance amount in the <b>{fundCode}</b> fund.` |
| Budget restricted expenditure | `Fund distribution amount exceeds the allowable expenditure amount in the <b>{fundCode}</b> fund.` (en.json `errors.budgetRestrictedExpendituresError`) |
| Non-vendor organization | `Invoice cannot be approved because the associated Vendor Id belongs to a non-vendor organization.` |
| Organization missing | `Invoice cannot be approved because organization does not exist` |
| Fund distribution not 100% on a line | `Invoice could not be approved or paid. The fund distribution total must be distributed by 100 percent in invoice line {invoiceLineNumber}. Please update fund distribution details for that invoice line to continue.` |
| Non-prorated adjustment without fund | `At least one fund distribution should be present for every non-prorated adjustment` |
| Non-prorated adjustment fund total | `Fund distributions summary should be 100 % or equal to subtotal for every non-prorated adjustment` |
| Expense class not on fund | `<b>{expenseClassName}</b> expense class not found on <b>{fundCode}</b> Fund` |
| Inactive expense class (Pay) | `Invoice can not be <b>Payed</b> because expense class <b>{expenseClass}</b> is inactive.` (sic) |
| Inactive expense class (Approve & pay) | `Invoice can not be <b>Approved and Payed</b> because expense class <b>{expenseClass}</b> is inactive.` (sic) |
| No budget on Pay / Approve & pay | `Invoice cannot be paid because Fund <b>{fundCode}</b> has no current budget.` / `Invoice cannot be approved and paid because Fund <b>{fundCode}</b> has no current budget.` (+ `... for fiscal year <b>{fiscalYear}</b>.` variants) |
| Fund records / external account | `Invoice cannot be approved because fund records are not found` / `Invoice cannot be approved because fund record does not contain an externalAccountNo.` |
| Pending payment failure | `Invoice cannot be approved because pending payment creation has failed` / `... pending payment update has failed` |
| Current FY missing | `Invoice cannot be approved because current fiscal year not found for ledger` |
| Encumbrance/fund mismatch | `Invoice could not be approved. Encumbrance fund id does not match the fund id in the invoice line. Error must be corrected in database.` |
| Transactions failed | `One or more transactions record(s) failed to be created` |
| PO line update failure | `One or more purchase order line record(s) cannot be updated` |
| Cancel: no active budget for target FY | `This operation could not be completed successfully. There is either no budget for <b>{fundCode}</b> for the targeted fiscal year or the budget is not Active.` (C422007-C422009) |
| Cancel failed | `Invoice was not cancelled` |
| Order acq unit | `Operation is not permitted because user is not a member of the specified acquisition unit of the Order(s)` (en.json `invoice.actions.error.userNotAMemberOfTheAcq.order`; NOT reproduced by C446069/C630444, which expect success — see Known Gaps) |
| Save invoice line: no FY | `Invoice line cannot be saved because invoice is not assigned to a Fiscal year.` (UINV-580, C569566) |
| Save invoice line: no budget for FY | `Invoice line cannot be saved because Fund <b>{fundCode}</b> has no current budget for fiscal year <b>{fiscalYear}</b>.` (C569568, UINV-557); BE 404 code `budgetNotFoundByFundIdAndFiscalYearId` "Active budget not found by fund id and fiscal year id" when adding a POL whose fund has a budget only in another concurrent FY (C353566) |
| Save invoice line: inactive expense class | `Invoice line can not be saved because expense class <b>{expenseClass}</b> is inactive.` |
| Save invoice line failed | `Invoice line was not saved` |
| Currency not convertible | `The selected currency cannot be converted. Please configure an exchange rate provider that supports this currency.` (UINV-610, C852071) |

### Modal / Dialog Titles

| Modal | Trigger | Source |
|---|---|---|
| Approve invoice | Actions > Approve | [confirmed] |
| Pay invoice | Actions > Pay | [confirmed] |
| Cancel invoice | Actions > Cancel | [from cases] |
| Approve & pay invoice | Actions > Approve & pay | [confirmed] |
| Save invoice | Duplicate detection on save | [confirmed] |
| Run manual export | Settings voucher export action | [from cases] |
| Select order lines | Add invoice line from PO lines — via `Add Line from POL` button in "Invoice lines" accordion | [confirmed — C2327] |
| Create vendor invoice line | `Actions > New blank line` in "Invoice lines" accordion (manual invoice line) | [confirmed — C2326, C343209] |
| Currency-mismatch confirmation | POL and invoice currency differ when adding line from POL | [confirmed — C2327] |

### Confirmation dialog messages (confirmed — GitHub translations)

| Dialog | Heading / message |
|---|---|
| Approve | `Approve invoice` / `Are you sure you want to approve invoice?` |
| Pay | `Pay invoice` / `Are you sure you want to pay invoice?` |
| Approve & pay | `Approve & pay invoice` / `Are you sure you want to approve and pay invoice?` |
| Cancel invoice | `Cancel invoice` / `Are you sure you want to cancel this invoice? Any related transactions against funds will be voided.` |
| Delete invoice | `Delete {vendorInvoiceNo}?` / `Are you sure you want to delete this invoice and all of its invoice lines? This will also delete any attached documents.` |
| Duplicate invoice | `Duplicate invoice?` / `Are you sure you want to create a duplicate of this invoice and all of its invoice lines?` |
| Add line from POL — currency differs | `Confirmation` / `You are adding one or more purchase order lines that are in a different currency than the one identified on this invoice. If you would still like to add these please click confirm. ...` |
| Add line from POL — vendor differs | `Confirmation` / `You are adding one or more purchase order lines that reference a different vendor than the one identified on this invoice. ...` |
| Update order status (cross-FY pay with release encumbrance) | `Update order status` / `You are processing this invoice against a previous fiscal year and "release encumbrance" equals true for one or more invoice lines. One or more related orders has a type of One-time. Please indicate how the payment status of these order lines should be updated?` — exact radio labels per en.json `invoice.actions.updateOrderStatus.status.*`: **No change**, **Awaiting payment**, **Partially paid**, **Fully paid**, **Cancelled** (sentence case — the earlier "No Change"/"Awaiting Payment" capitalisation is the BE `poLinePaymentStatus` parameter value, not the UI label; "No Change" as default selection is [from cases (C700852)], not in translations). Selecting "No change" leaves the linked POL's Payment status untouched even though the invoice itself completes Approve/Pay. |
| Duplicate invoice? | Actions > Duplicate on an invoice detail pane → `Duplicate` / `Cancel` buttons | [from cases (C514954, C514958, C1332438)] |

### Button Labels (key actions)

Actions, Submit, Save and close, Save, Cancel, Continue, Add adjustment, Add fund distribution, Run manual export, Confirm, Delete, View voucher, Duplicate, Save & keep editing.

Approve / Pay / Approve & pay dialogs are confirmed with **Submit** in TestRail (C357038, C1231686, C350633); docs call the Approve confirm button "Approve" (docs/UI wording differ). Duplicate invoice dialog uses `Duplicate`; add-line-from-POL confirmations use `Confirm`.

### Error / Warning Messages

Approve/pay, budget, restriction and duplicate-warning texts are in the tables above. Others: `Invoice was not approved` (C1009057), `Invoice was not paid`, `Invoice was not approved/paid`, `Failed to load invoice data` (en.json `invoice.actions.*`, `invoice.actions.load.error`).

---

## Status Values

### Invoice workflow status

Open, Reviewed, Approved, Paid, Cancelled (en.json `invoice.status.*`).

### Voucher status

Awaiting payment, Paid (UI `voucher.status.*`); Cancelled exists in the schema enum and is set on invoice cancel.

### Voucher export (batch voucher export) status

Pending, Generated, Uploaded, Error (en.json `settings.BatchVoucherExports.status.*`; schema batch_voucher_export.json).

### POL payment status (integration)

Awaiting Payment, Fully Paid, Partially Paid; also Ongoing and Payment Not Required (never overwritten by invoice actions).

---

## Capability Sets (Eureka)

| Action | Capability Set | Confirmed by |
|---|---|---|
| View invoices | `Data - UI-Invoice Invoice - View` | Cypress dictionary `uiInvoiceInvoiceView`; docs; C196775 |
| Create invoices | `Data - UI-Invoice Invoice - Create` | Cypress `uiInvoiceInvoiceCreate`; C196773 |
| Edit invoices | `Data - UI-Invoice Invoice - Edit` | Cypress `uiInvoiceInvoiceEdit`; C196776 precondition |
| Delete invoices | `Data - UI-Invoice Invoice - Delete` | Cypress `uiInvoiceInvoiceDelete`; C196772 |
| Approve invoices | `Procedural - UI-Invoice Invoice Approve - Execute` | Cypress `uiInvoiceInvoiceApprove`; C196776 |
| Pay invoices | `Procedural - UI-Invoice Invoice Pay - Execute` | Cypress `uiInvoiceInvoicePay`; C196779 |
| Cancel invoices | `Procedural - UI-Invoice Invoice Cancel - Execute` | Cypress `uiInvoiceInvoiceCancel`; C350728 precondition |
| Pay in different fiscal year | `Procedural - UI-Invoice Invoice Pay-Different-Fy - Execute` | Cypress `uiInvoiceInvoicePayDifferentFy` |
| Export vouchers | `Procedural - UI-Invoice Voucher Export - Execute` | docs Invoices capability table; C442851, C397985 preconditions (not in Cypress dictionary) |
| Download batch file | `Procedural - UI-Invoice BatchVoucher - Execute` | docs; C196780, C353222 preconditions (not in Cypress dictionary) |
| Assign acquisition units | `Procedural - UI-Invoice Invoices Acquisitions Units Assign - Execute` (since Trillium; before: `Procedural - UI-Invoice Acq Unit Assignment - Execute`) | Cypress `uiInvoiceInvoicesAcquisitionsUnitsAssign`; C196777 precondition |
| Manage acquisition units | `Procedural - UI-Invoice Invoices Acquisitions Units Manage - Execute` (since Trillium; before: `Data - UI-Invoice Acq Unit Assignment - Manage`) | Cypress `uiInvoiceInvoicesAcquisitionsUnitsManage`; C196778, C357051 preconditions |
| Export search results | `Procedural - UI-Invoice Export CSV - Execute` | docs Invoices capability table; ui-invoice package.json `ui-invoice.exportCSV` (not in Cypress dictionary) |
| View Invoices settings | `Settings - UI-Invoice Settings - View` (OKAPI: `Settings (Invoices): View settings`, `ui-invoice.settings.view`) | docs Settings_invoices capability table (type settings / view) |
| Edit Invoices settings | `Settings - UI-Invoice Settings - Manage` (OKAPI: `Settings (Invoices): Can view and edit settings`, `ui-invoice.settings.all`) | docs; C353608 precondition (C196781 still spells it `data - UI-Invoice Settings - manage`) |
| View batch group credentials | `Data - UI-Invoice BatchVoucher ExportConfigs Credentials - View` (`ui-invoice.batchVoucher.exportConfigs.credentials.view`) | docs Settings_invoices capability table |
| Edit batch group credentials | `Data - UI-Invoice BatchVoucher ExportConfigs Credentials - Edit` (`ui-invoice.batchVoucher.exportConfigs.credentials.edit`) | docs; C353222, C397985 preconditions |

Source: Cypress dictionary (`cypress/support/dictionary/capabilitySets.js`), docs Capabilities tables (Invoices and Settings_invoices; the docs still list the pre-Trillium acquisition-unit names), ui-invoice 8.1.0 package.json permissionSets (display names), TestRail Permissions section 1704 and case preconditions. No capability name in this table is derived from naming convention. Renamed in 8.0.0 to be easier to tell apart (UINV-620); `Batch-Groups Collection (view)` is included in the `UI-Invoice Invoice` view set (UINV-602); `stripes-core.settings.read` is in the base sets (UINV-615). Required companion capabilities used in TestRail preconditions: `data - UI-Orders Orders - view` (C353566), `data - UI-Finance Fund-Budget - view` (C10945, C3453, C1231686, C357038); credit/budget setup in C347897 uses `data - UI-Finance Fund-Budget - create`. A view-only user lacks `finance.exchange-rate.item.get`, so `Calculated total amount (Exchanged)` shows the unconverted amount (UINV-641, Done 2026-11-04).

Capability behaviour from Permissions section 1704 (C196772-C196781): the Invoice data capabilities are cumulative and exclusive at the edges — View: can view, cannot edit; Edit: view+edit, cannot create or delete; Create: view+edit+create, cannot delete; Delete: view+edit+delete, cannot create; Settings - Manage: view and edit Approvals/Adjustments (General) and Batch groups/Batch group configuration/Voucher number (Vouchers). Approve needs Approve + Invoice Edit (C196776); Pay needs Pay + Invoice Create (C196779); batch file download needs BatchVoucher + Invoice View (C196780). Invoices list is paginated 50 per page with Previous/Next (C196775, UINV-348, UINV-480). Following an invoice link from an order's Related invoices without Invoice view shows `You don't have permission to view this app/record` (C353617). Other permission cases: C350624 (Cancel capability assignable), C196777/C196778 (acquisition units).

---

## Common Verification Patterns

### Approve creates pending payment

```
Action:   Click Actions > Approve and confirm in Approve invoice modal.
Expected: Invoice status changes to Approved; approval metadata is populated; pending payment transactions are created in Finance.
```

### Pay creates payment and relieves awaiting payment

```
Action:   Click Actions > Pay and confirm in Pay invoice modal.
Expected: Invoice status changes to Paid; payment transactions are created; budget Expended increases and Awaiting payment decreases.
```

### Duplicate detection on save

```
Action:   Save an invoice with existing vendor + vendor invoice number + invoice date combination.
Expected: Save invoice dialog appears with warning This appears to be a duplicate invoice; user can submit or cancel.
```

---

## Duplicate Invoice (Actions > Duplicate)

> Previously only cited via its toast string; this round confirms exact field-carry-over/reset behavior. [C514954, C514958, C1332438]

- Actions > Duplicate → confirmation modal (Duplicate/Cancel) → new invoice created with the **same vendor invoice number** and copies of all invoice lines/adjustments.
- Regardless of the SOURCE invoice's status (Open, Reviewed, Approved, Paid, Cancelled), the **duplicate is always created in "Open" status**, and all its invoice lines are duplicated as "Open" too.
- **Fiscal year is NOT copied**: if the current fiscal year (matching alpha-code family) cannot be resolved, it's left blank; otherwise the duplicate is auto-set to the CURRENT fiscal year, never the original invoice's (possibly past) fiscal year.
- Record metadata resets on the duplicate: "Resource created" = now, "Source" = the user performing the duplicate (not the original creator).
- A **Cancellation note** on a Cancelled source invoice is explicitly dropped — the field is not displayed on the duplicate AND is absent entirely from the `cancellationNote` field in the API response for the new invoice (confirmed via DevTools network inspection). [C1332438]
- Duplicated invoice lines keep their POL links and adjustments; the "Current encumbrance" hyperlink on a duplicated line references the NEW invoice's (current) fiscal year budget, not the original's.
- Duplicate is permission-gated (separate capability check) — a user lacking the relevant Invoice create/view permission cannot duplicate at all (Actions menu omits the option) (C514953).
- Edge cases covered by cases: Paid source with manually set exchange rate and a line not tied to a POL (C514955); Approved with two lines, one POL-linked (C514957); Paid with no current budget (C514959; toast fix UINV-571); Approved line with negative sub-total (C514960); Open with 3 lines where the budget of fund 1/2 was deleted (C514972); Paid with acquisition unit the user does not belong to (C514984); changing a line in a duplicated Paid invoice with prorated invoice-level adjustment (C1282784). Failure texts: `The invoice has not been duplicated`, `The invoice line <b>{invoiceLineNumber}</b> failed to duplicate.`
- Cancellation note exclusion is done client-side (UINV-636, ui-invoice 8.1.0).

---

## Version History (Invoice / Invoice Line)

> Undocumented prior to this round. [C613165, C877080, C663321]

- A clock ("Version history") icon appears top-right of the invoice detail pane (and separately for invoice lines). Clicking opens a 4th pane listing every saved version as a card, **most recent first**, each titled with a date/time in `MM/DD/YYYY, HH:MM AM` format.
- While the Version history pane is open: the underlying invoice's "Actions" button is hidden, and the Version-history/Tags icons on the invoice pane itself become disabled (can't stack panes).
- Selecting a card highlights every field that changed IN that version, in **yellow**, directly on the invoice detail pane behind it.
- The oldest card is labeled "Original version" (no changed-field list); the newest is labeled "Current version" plus a "Changed" section listing every changed field name.
- If a referenced record (e.g. the vendor Organization) is later deleted, older versions still resolve it as "Record deleted" in the Vendor details accordion, and re-opening the live invoice shows an error toast `Failed to load invoice data`.
- Adjustment-related field changes (Total adjustments, Calculated total amount, Description, Value, Pro rate, Relation to total) and currency/exchange-rate changes are tracked and highlighted like any other field.
- Approving an invoice creates its own version entry with Status, Approval date, Approved by, Voucher number, and Exchange rate all listed as changed.
- Version history also tracks Sub total and Calculated total amount (MODINVOICE-648, fixed 6.1.1). It shows human-readable values instead of internal IDs for adjustments and fund distributions (UINV-613, C877080); labels come from en.json `versionHistory.label.adjustments.*` (Adjustment description, Adjustment pro rate, Adjustment relation to total, Adjustment type, Adjustment value, Adjustment export to accounting) and `versionHistory.label.fundDistributions.*` (Fund distribution code / type / value / expense class). `No version history to display.` when empty (`invoice.versionHistory.noVersion`).
- Fields tracked include Vendor details and Extended information accordions (UINV-569), voucher lines, links and documents, PO line association/disassociation (UINV-474/475/476); Sub total and Calculated total amount are included (MODINVOICE-648). Since UINV-639 (ui-invoice 8.1.0) `metadata.updatedDate` is no longer shown as a changed field in the invoice and invoice-line version views. A user with all invoice capabilities can see versions (UINV-576).
- Audit events come from mod-invoice-storage's audit outbox (Invoice, InvoiceLine; edit events carry the old object — MODINVOSTO-216).

---

## "Save & Keep Editing" Button (Create Invoice / Create Invoice Line)

> New UI mechanic, not previously documented. [C663268]

- Both the standalone "Create vendor invoice" screen and "Create vendor invoice line" screen show three buttons in order: **Cancel** (always active), **Save & keep editing** (disabled until at least one field is filled), **Save & close** (same enable condition, styled as the primary blue button).
- Clicking "Save & keep editing" with missing mandatory fields shows the same inline red "Required!" validation as Save & close, without closing the screen.
- On a successful "Save & keep editing", a success toast appears, the screen stays open, and both Save buttons return to a disabled state until further edits are made.
- Cancelling with unsaved changes prompts a "Close without saving" confirmation; declining preserves the create screen, confirming discards the in-progress edits.
- **"Save & keep editing" is NOT displayed** when creating an invoice via the Orders app's Actions > "New invoice" flow (order-based invoice creation) — only the two standalone create-from-scratch flows offer it.

---

## Batch Group Credentials — Permission Tiers (Settings > Invoices)

> [C422074] plus adjacent cases in the same TestRail cluster (422071, 422073) confirm three distinct permission tiers.

- Settings > Invoices pane layout: **General** section (Approvals, Adjustments) + **Vouchers** section (Batch groups, Batch group configuration, Voucher number).
- Three permission tiers gate "Batch group configuration" specifically:
  1. `Settings (Invoices): View settings` — can view the settings pane but NOT batch group credentials at all.
  2. `Settings (Invoices): Batch group usernames and passwords: view` — can view+edit general invoice settings, but batch group username/password fields are visible-only (not editable).
  3. `Settings (Invoices): Batch group usernames and passwords: view and edit` — full edit access to Batch group configuration fields (Batch group, Location type, Upload location, Port, Directory, Format, Username, Password) plus an active "Show credentials" button that reveals the plaintext Username/Password.
- This is a deliberately separate, more restrictive permission from general Invoice-settings access — a useful negative-case axis (view-only vs edit-without-credentials vs full-credentials).

---

## Adjustment Fund Distribution — Strict Pro-Rate Coupling

> Refines/extends Key Business Rule 11. [C825250, C889747]

- An adjustment's **fund distribution is only meaningful/visible when "Pro rate" = "Not prorated"**. The moment "Pro rate" is switched to ANY other value (By line / By amount / By quantity), the adjustment's fund distribution is immediately hidden from the UI and stripped from the saved record — the API payload's `adjustments[].fundDistributions` becomes an empty array `[]`, confirmed on both PUT (editing an existing adjustment) and POST (creating a new one).
- This holds even if the Fund ID dropdown was left blank — switching Pro rate away from "Not prorated" clears it either way, and switching back to "Not prorated" does not restore a previously-cleared fund selection (user must re-pick the fund).
- Test design implication: any adjustment case that toggles Pro rate should assert both the UI (fund-distribution controls disappear) and the API response body (`fundDistributions: []`) for full coverage.

---

## Exchange Rate — Zero-Decimal Currency Display (JPY)

> [C1322894]

- When the tenant's primary/target currency is JPY (a zero-decimal currency), the UI's "Calculated total amount (Exchanged)" field displays as a whole number with no decimal places (e.g. `¥16717`), matching real-world yen formatting.
- However, the underlying `GET /finance/calculate-exchange` API call always returns its `amount` with an explicit `.0` decimal suffix regardless of target currency (e.g. `16717.0`) — the zero-decimal formatting is a UI-layer presentation choice only, not an API contract difference. This holds for both provider-calculated rates and manually-entered "Use set exchange rate" values.

---

## Fiscal Year Auto-Resolution Rules

> [C396373, C700852]

- Leaving "Fiscal year" blank when creating an invoice is allowed; on save, FOLIO auto-populates it with the CURRENT fiscal year whose alpha-code family matches the invoice date — the user never sees "undefined" persist.
- Once a Fiscal year value exists on a saved invoice (even an auto-populated one), the Edit page's "Fiscal year" dropdown **no longer offers a blank/undefined option** — a fiscal year must be explicitly (re)selected going forward; there's no way back to "undefined" once set.
- Approving/paying an invoice against a FISCAL YEAR THAT IS NOT the ledger's current one correctly posts Pending payment / Payment transactions against the ORIGINAL (past) fiscal year's budget, not the currently active rolled-over budget — verified by checking the resulting transaction's own "Fiscal year" field and the past-FY budget's transaction list directly.
- The "Update order status" modal's per-line "Status" (Released/Unreleased) on the resulting Encumbrance transaction can differ per invoice line depending on whether that line's PO line payment status was actually updated ("No Change" selection leaves an Unreleased/stale linkage on that specific line while other, fully-processed lines correctly show "Released").

---

## ECS / Multi-Tenant Notes

Consortium (Invoices) section 37957 holds 4 cases (bodies of C590842, C356846, C353608 read 2026-11-25; C610245 read 2026-07-22): C590842 (ECS | filter invoices by "Fund code" in member tenant — user is Staff type with `Data - UI-Invoice Invoice - View` in Member plus `Settings - UI-Consortia-Settings Settings NetworkOrdering - View` in Central; Fund A/B offered, Fund C bound to an acquisition unit the user is not in is NOT offered; selecting A returns invoices #1, #2, #4, adding B returns #1-#4, an invoice with no fund distribution is never returned; clearing the filter restores `<- Choose a filter or enter a search query to show results.`), C610245 (ECS | pay invoice linked with POL created in Member tenant), C356846 (a user cannot view an invoice created by another tenant's user and vice versa: searching the other tenant's invoice number shows `No results found for invoice number. Please check your spelling and filters.`), C353608 (invoice settings applied in tenant A are not applied in tenant B). Invoices, vouchers, batch groups and settings are tenant-local; primary behavior remains the same as the non-ECS lifecycle, with tenant/capability gating in consortial contexts. mod-invoice has no Central-tenant-specific endpoints in its ModuleDescriptor; the cross-tenant POL lookup lives in the shared stripes-acq-components modal.

**Canonical mechanic documented in orders.md** ("Cross-Tenant Location Lookup via 'Affiliation' Dropdown") — Invoices reuses the identical UI, but nested one level deeper: inside an invoice's `Invoice lines` accordion, `Actions > Add line from POL` opens a `Select order lines` modal that has its own `Location` facet with the same `Location look-up` > `Select locations` > `Affiliation` dropdown pattern (confirmed C471510).

- **Positive flow** (confirmed C471510): with the Central-tenant prerequisite setting on (`Settings > Consortium manager > Central ordering`), a Central-tenant user in `Add line from POL`'s Location facet can select a Member tenant in the `Affiliation` dropdown (default: Central), see only that tenant's own locations, check one, and the corresponding PO line (created against a Member tenant's location) is returned in the `Select order lines` modal's results — letting a Central-tenant invoice pull in a line ordered under a Member tenant's location.
- The "Location" facet's "Affiliation" dropdown (used to search POLs by location across tenants when adding an invoice line "from POL") is **Central-tenant-only** — from a Member tenant, the same "Add line from POL" > Location facet > location look-up modal does NOT show an Affiliation dropdown at all; only the Central tenant's own local locations are searchable directly. [C471512]
- Per orders.md's fuller mechanic (also true here, not yet separately case-confirmed in Invoices but confirmed identically in Orders/Receiving): the dropdown, when present, lists ALL consortium tenants regardless of the user's own assigned affiliations.
- Paying an invoice whose linked POL was created in a Member tenant, performed directly from that Member tenant (not Central), follows the exact same Pay flow, encumbrance-release, and voucher-status transitions as a non-ECS invoice — no special cross-tenant Pay restriction exists once the user has Member-tenant Invoice/Finance/Orders capabilities. [C610245]

---

## Existing automation coverage

- **Karate** (folio-org/folio-integration-tests, `acquisitions/src/main/resources/thunderjet/mod-invoice/`, runners `InvoicesApiTest.java`, `InvoicesExtendedApiTest.java`; 48 `.feature` files incl. reusables `approve-invoice`, `pay-invoice`, `cancel-invoice`, `create-invoice`, `create-invoice-line`): invoice.feature (main flow), approve-and-pay-invoice-with-past-fiscal-year, cancel-invoice, check-lock-totals-and-calculated-totals-in-invoice-approve-time, invoice-with-lock-totals-calculated-totals, check-approve-and-pay-invoice-with-zero-dollar-amount, pay-invoice-with-0-value, check-invoice-full-flow-where-subTotal-is-negative, check-invoice-line-validation-with-adjustments, check-invoice-lines-with-vat-adjustments, prorated-adjustments-special-cases, invoice-with-identical-adjustments, check-that-not-possible-add-invoice-line-to-approved-invoice, check-that-not-possible-pay-for-invoice-without-approved / -if-no-voucher, check-that-changing-protected-fields-forbidden-for-approved-invoice, check-invoice-and-invoice-lines-deletion-restrictions, check-invoice-lines-and-documents-are-deleted-with-invoice, check-that-can-not-approve-invoice-if-organization-is-not-vendor, check-error-respose-with-fundcode-upon-invoice-approval, check-remaining-amount-upon-invoice-approval, expense-classes-validation, create-voucher-lines-honor-expense-classes, voucher-with-lines-using-same-external-account, voucher-numbers, update-voucher, should_populate_vendor_address_on_get_voucher_by_id, batch-voucher-uploaded, batch-voucher-export-with-many-lines, exchange-rate-update-after-invoice-approval, set-invoice-fiscal-year-automatically, invoice-fiscal-years, fiscal-year-balance-with-negative-available, fund-code-auto-populate-invoice-lines, delete-line-check-next-line-number, edit-subscription-dates-after-invoice-paid, check-approve-and-pay-invoice-with-odd-pennies-number. API-verifiable rules in this file are therefore candidates for "already covered" in write-karate-first-cases.
- **Cypress** (folio-org/stripes-testing): fragments `cypress/support/fragments/invoices/` (invoices.js, invoiceView.js, invoiceLineDetails.js, invoiceNewForm.js, invoiceEditForm.js, invoiceLineEditForm.js, newInvoice.js, newInvoiceLine.js, newBatchGroup.js, settingsInvoices.js, voucherExportForm.js, voucherView.js, vouchers.js, vendorAddress.js, invoiceStates.js, invoiceStorageSettings.js, modal/); 173 specs under `cypress/e2e/invoices/` (approve/pay/cancel per FY, `pol-payment-status-*`, encumbrance cancel matrix, `duplicate-invoice-*`, `fiscal-year-*`, `filter-invoices-*`, `version-history-*`, `save-using-save-and-keep-editing-button-*`, restriction-message specs, inactive/closed budget specs, `settings/`). Many TestRail cases have a same-named spec (e.g. C700852 -> pol-payment-status-not-updated-if-no-change-selected-when-approving-invoice-against-previous-fiscal-year.cy.js); Jira FAT-* "Review of C…" links (FAT-25268 for C1231686/C1241224, FAT-27662 for C1332438) mark automated cases.

---

## Edge cases from bugs (Jira)

- Encumbrances with multiple expense classes on one fund distribution: both must be released on Approve/Pay with Release encumbrance = true (MODINVOICE-654, fixed 6.1.3; matching by POL + fund + expense class with fallback to POL + fund; credit and payment lines may share one encumbrance).
- Different funds in POL and invoice line: Pending payment/Payment go to the invoice line's fund; the POL fund's encumbrance is released if release encumbrance is checked (MODINVOICE-290, MODINVOICE-600; C378895 invoice with fund different from POL can be approved; C353596 fund distribution changed in POL before approval).
- Negative / over-limit encumbrance after paid + cancelled invoices (MODINVOICE-608, 613, 616, 628): case families are listed under Cancel Flow (encumbrances); initial encumbrance must be unchanged after cancelling and recalculated correctly when another invoice with release encumbrance true/false exists or after manual unrelease (C1028980, C1028981, C1028984, C1028985, C1045390-C1045396).
- Cancelling an invoice in a past FY reopens the order and creates an unreleased encumbrance in the current FY — accepted edge case, MODINVOICE-656 closed as won't do.
- Invoice not loading when the vendor organization was deleted: toast `Failed to load invoice data`, Vendor name blank, version history shows `Record deleted` (UINV-634; fixed 8.0.1).
- Invoice lines imported via EDI/Data Import: search results must show fund code (MODINVOICE-620); full account-number dropdown selectable on EDI lines (UINV-575); invoice imported from EDIFACT can be deleted after editing another invoice (C358141); supplemental invoice creation failure was reported (MODINVOICE-635, RRT).
- Large EDI files crashed Data Import (MODINVOICE-652/653 fixed 6.1.2) — Data Import log shows the `Invoice` section (TestRail section 131125).
- Exchange: ECB provider does not support all currencies (UINV-610, C852071); amounts must use the Finance exchange rate API (MODINVOICE-579); Treasury / Currencyapi calculated total fix (MODINVOICE-611); "No key required" provider with manually set rate (C740212, C844847); update exchange rates on lines with multiple funds (C378881); currency without ECB support when exporting (C421999).
- Multiple fiscal years on one invoice: MODINVOICE-642 / MODINVOSTO-214 prepare the module (fiscalYearId moves into fund distributions, Confluence "Process invoices against future fiscal years", UXPROD-4490) but the approve error `multipleFiscalYears` is still returned in the current release.

---

## Key Business Rules for Test Cases

1. Default invoice workflow status on creation is Open and the invoice is editable after save. (C2299, C353566; docs)
2. Approval is allowed from both Open and Reviewed statuses; Reviewed is optional, not mandatory. (C196776, C10945; docs)
3. Vendor must be a valid active vendor organization for successful approval. (C388558, C388559; ErrorCodes `organizationIsNotVendor`)
4. Duplicate detection is enforced on vendor + vendor invoice number + invoice date, requires an explicit user decision to proceed, and also runs on `Approve & pay`. (C440075; docs; en.json `invoice.isNotUnique.confirmation.message`)
5. Lock total prevents approval when calculated totals do not match the lock total amount. (en.json `lockCalculatedTotalsMismatch`; Karate check-lock-totals-and-calculated-totals-in-invoice-approve-time.feature)
6. Approve creates pending payment transactions and moves budget amounts into Awaiting payment. (C10945, C347897; MODINVOICE-346)
7. Pay creates payment transactions and moves budget amounts into Expended while decreasing Awaiting payment. (C3453, C347897)
8. Only Approved and Paid invoices can be cancelled, always with the `Invoice: Cancel invoices` capability; Cancel voids the pending payment/payment/credit transactions and sets the voucher to Cancelled. (docs Cancelling an invoice; ErrorCodes `cannotCancelInvoice`; C350624, C350728)
9. Negative calculated total posts credit behavior in Finance (increasing available budget). (C343209, C347897; MODINVOICE-547)
10. Separate from adjustments — and, since MODINVOICE-639/640 (mod-invoice 6.1.0), Included in adjustments — are not included in Total adjustments or Calculated total amount. (MODINVOICE-639, MODINVOICE-640; docs conflict in Known Gaps; TestRail Included-in cases C376007, C589259, C589504, C589508, C589510 predate the revert / are DRAFT)
11. Not prorated adjustments require an explicit fund distribution at adjustment level; it is kept only while Pro rate = Not prorated — switching to any other Pro rate value strips it from UI and from the saved record (`fundDistributions: []`), and prorated invoice-level adjustments are read-only on the invoice-line form. (ErrorCodes `adjustmentFundDistributionsNotPresent`; C825250, C889747, C1292041; UINV-614, UINV-630)
12. Release encumbrance (unchecked by default on a new blank line) controls release at Approve rather than at Pay: when checked the encumbrance shows 0.00/Released after Approve. (C389582, C350633, C399084)
13. Voucher is generated at approval (status Awaiting payment) and carries the metadata used in export/accounting workflows. (C10945; docs Approving an invoice)
14. Export to accounting controls voucher inclusion in batch export. (C396387, C397317)
15. An accounting code on the vendor is required for approval/payment when Export to accounting is on. (en.json `accountingCodeNotPresent`; C10943 preconditions)
16. POL payment status is changed at Pay (not Approve): Fully Paid if any line releases the encumbrance, else Partially Paid; Ongoing and Payment Not Required are never overwritten; Cancel of a Paid invoice returns it to Awaiting Payment / Partially Paid / keeps Fully Paid depending on other paid invoices in the same FY. (MODINVOICE-585, MODINVOICE-544, PoLinePaymentStatusUpdateService.java; C466207-C466211)
17. Duplicating an invoice always creates it as Open with all lines Open, resets fiscal year to the current FY (or blank), never copies the cancellation note (UI-side, UINV-636) and resets created-by metadata; vendor invoice number, lines (with POL links) and adjustments transfer. (C514954, C514958, C1332438; UINV-636)
18. Version history tracks every save on both Invoice and Invoice line, highlighting changed fields in yellow and labeling the oldest/newest cards Original/Current version. (C613165, C663321)
19. "Save & keep editing" is available on both standalone create screens (invoice, invoice line) but not on the order-triggered "New invoice" flow. (C663268, C663271)
20. Batch group username/password access is gated by a permission tier strictly separate from general Invoice-settings view/edit access. (C422071, C422073, C422074)
21. Zero-decimal currencies (e.g. JPY) are displayed without decimals in Calculated total (Exchanged) even though the exchange-calculation API always returns an explicit decimal value. (C1322894)
22. An invoice's fiscal year auto-resolves to the current FY if left blank at creation and cannot be reverted to blank once set; only users with `Invoice: Pay invoices in a different fiscal year` see/edit the field, and it is read-only for Approved, Paid and Cancelled invoices. (C396373, C388645, C396390, C387522, C387533, C387534, C387536, C387537; ErrorCodes `cannotResetInvoiceFiscalYear`)
23. Approve/Pay transactions always post against the fiscal year the invoice is scoped to, even when it is no longer the ledger's current one after a rollover. (C388520, C388538, C388563)
24. The "Add line from POL" modal's `Location` facet `Affiliation` dropdown is the same canonical cross-tenant lookup mechanic as Orders/Receiving — Central-tenant-only, positive flow confirmed C471510. (C471510, C471512)
25. Approve is allowed only with at least one invoice line, fund distributions totalling 100% of every line and non-prorated adjustment, all funds in one fiscal year, and no related order in Pending workflow status. (docs; en.json `invoice.details.hasPendingOrders`; ErrorCodes `fundDistributionsNotPresent`, `multipleFiscalYears`; C397321, C397326)
26. Budget restrictions are validated by mod-finance-storage; the UI shows the fund-specific message for encumbrance vs expenditure restriction, and credits reduce the expenditure used; the legacy `not enough money` message applies only before Trillium. (MODINVOICE-631, MODINVOICE-547, UINV-623; C1231686, C1241224, C375959)
27. Paying, approving&paying or cancelling an invoice against a previous fiscal year with a Release-encumbrance line on an Open One-time order requires a POL payment status choice (`Update order status` modal / `poLinePaymentStatus`); other combinations skip the modal. (MODINVOICE-573, MODINVOICE-585, MODINVOICE-587, UINV-577, UINV-600; C700852-C700861, C703334, C703346, C709266)
28. A vendor invoice line number is never reused after deleting a line. (MODINVOICE-647; C389497, C410742)
29. Invoices can be deleted only in Open or Reviewed; lines cannot be added to an Approved invoice; documents can still be attached to and removed from Approved and Paid invoices. (docs Deleting an invoice; ErrorCodes `approvedOrPaidInvoiceDeleteForbiddenError`, `prohibitedInvoiceLineCreation`; C360544, C360546, C360547)
30. Subscription info, comments and tags of invoice lines stay editable after Approve/Pay, including when the fund budget is Inactive or the prior-FY budget is Closed. (MODINVOICE-618; C350952, C919907, C919908)
31. A voucher has one voucher line per fund external account number (incl. expense-class extension); the Group column is not shown; only disbursement number/date/amount (and voucher number if enabled in settings) are editable. (docs Voucher lines; UINV-629, UINV-565; C1152357, C423429)
32. Voucher export takes all not-yet-exported vouchers of the batch group (no date range), includes only invoices/adjustments with Export to accounting checked, and writes human-readable timestamps in JSON and XML. (docs Exporting vouchers; MODINVOICE-595/601; C396387, C397317, C825246, C825247)
33. The system `FOLIO` batch group can be edited but not deleted, and a batch group in use on invoices cannot be deleted. (docs Settings > Invoices > Batch groups; C10938, C367942; en.json `settings.batchGroups.cannotDeleteTermMessage`)
34. "Approve and pay in one click" turns the Approve action into `Approve & pay`; both modes must be verified for the inactive-organization and Pending-order guards. (docs Settings > Approvals; C388558, C388559, C397321, C397326)
35. A user with only the Invoice view capability gets an unconverted "Calculated total amount (Exchanged)" because `finance.exchange-rate.item.get` is missing from the set — regression check for capability boundaries. (UINV-641)
36. Invoice currency defaults from the related order / vendor currencies, a currency-change warning modal appears when creating from an order, and unsupported-by-provider currencies show an informative toast. (UINV-579, UINV-610; C440073, C451451, C808485, C852071)
37. Subtotal/total of the invoice are recalculated when a line is added or removed. (MODINVOICE-570; C663350)
38. Search, filters and the CSV export honour time-zone for date created filters and include fund distribution and created/updated columns. (docs; C466118, C466123, C773245, C466125)
39. Pay with total = 0 is allowed: positive fund parts become Payment transactions and the negative part a Credit; fund distribution types cannot be mixed for a zero price. (C357038, C357042, C357044; ErrorCodes `cannotMixTypesForZeroPrice`)
40. An invoice-level adjustment fund differing from the line's fund still approves and pays; prorated adjustments cannot be added or deleted on a line when absent/present on the invoice. (C825243, C407745; ErrorCodes `cannotAddAdjustment`, `cannotDeleteAdjustment`)
41. Settings > Invoices > Voucher number prefix must contain letters only; voucher number prefix and start number live in invoice-storage settings (no mod-configuration). (en.json `voucherNumberPrefixNotAlpha`; UINV-607, UINV-616)
42. In ECS, invoices, vouchers and Invoices settings are tenant-local: one tenant cannot see the other's invoices or settings; the Fund code filter in a Member tenant offers only funds the user may see (a fund bound to an acquisition unit the user is not in is absent) and multi-select returns the union. (C356846, C353608, C590842)

---

## Authoring style (measured 2026-07-23)

Invoices is strongly **`Functional` (~95%)**, median ~8 steps, `User Journey` ~1% (`No`) — an acquisitions-family shape like Orders/Finance. Cases build the money context in **Preconditions** with exact values (Fund + Budget in an active Fiscal Year, an Open PO/POL to link, a Vendor/Organization, batch group, currency) and then walk the approve/pay/cancel/duplicate flow, **verifying the Finance side at its real destination** — the Pending payment / Payment / Credit transactions and the budget's Encumbered/Expended/Available values column-by-column, not just an invoice toast. Adjustment-calculation cases spell out the exact resulting `Total adjustments` / `Calculated total amount` numbers. Match this: exact monetary preconditions, `Functional` type, one coherent flow per case, Finance-transaction verification.

---

## Known Gaps / Items to Verify

- [ ] **"Included in" adjustment calculation — docs vs code.** Docs "Invoice adjustments" (dated 2024-12) and Settings docs describe Included-in as reducing the subtotal ($10 + 10% -> subtotal $9.09). MODINVOICE-639/640 (Done 2026-03, mod-invoice 6.1.0) reverted that and says Included-in is excluded from sub-total, Total adjustments and Calculated total. TestRail cases C589259, C589504, C589508, C589510 and C1273170, C1273172, C1282774, C1282775, C1282776 are still marked (DRAFT) and C376007 predates the revert. Verify on a live Umbrellaleaf environment before writing expected totals.
- [ ] **POL payment status at Approve.** Docs (Invoices > Approving an invoice, 2026-05-15) say Approve sets the POL to "Partially paid"; the code/Jira (MODINVOICE-585, PoLinePaymentStatusUpdateService) update POL status at Pay. The previous version of this file said "Awaiting Payment at Approve" — removed (source: MODINVOICE-585 comment "logic ... moved from approving to paying invoice"). Confirm the behaviour of a first Approve on a POL that is `Pending`/`Awaiting Payment` on a live system.
- [ ] **Relation to total options in presets.** Docs Settings > Adjustments list only "In addition to" and "Separate from"; schema and invoice docs have three values. UINV-630 adjusted the dropdown; not verified which values the preset form offers.
- [ ] **Release labels.** Jira epics call the releases "Trillium (R2 2025)" and "Umbrellaleaf (R1 2026)", mod-invoice NEWS.md heads 6.1.0 "Trillium R1 2025", ui-invoice 8.0.0 is "Trillium R1 2026 Module release" (UINV-632), and the refresh task names Umbrellaleaf R2 2026. This file uses ui-invoice 8.0.0 = Trillium R1 2026 and 8.1.0 (in progress) = Umbrellaleaf R2 2026. No `fixVersion` carries a release name (only module versions), so Jira scope was by resolution date >= 2024-06-01.
- [ ] **Source MARC.** Docs list `MARC` as an invoice Source filter value; schema enum has only User, API, EDI.
- [ ] **Default of "No change" in Update order status modal** is stated only by case C700852 (not by translations).
- [ ] **Lock total default setting** (UINV-463/464, Done 2024-07) has no matching docs, permission or en.json key — likely not shipped; do not write cases for it without confirmation.
- [ ] Confirm/Submit label in Approve/Pay dialogs: docs say `Approve`, TestRail cases say `Submit` (C357038, C1231686, C350633). Not checked in the live UI.
- [ ] Confluence: no invoice-specific functional spec found beyond Acquisitions SIG pages (Process invoices against future fiscal years — all use cases PENDING; Order and Invoice Analytics; Kafka audit design for Voucher). Multi-fiscal-year payments (UXPROD-4490) are not shipped: approve still fails with `multipleFiscalYears`.
- [ ] Voucher Kafka audit events (MODINVOSTO-218/219) are designed, not confirmed as released; no UI exposure.
- [ ] Some extracted capability strings in old cases include HTML artifacts (for example trailing </li>); use normalized values from table above when writing new cases.
- [ ] `en.json` and `en_US.json` are identical in ui-invoice (511 keys, no differences) at the time of this refresh.
- [ ] Whether a duplicated invoice's fiscal year can ever resolve to something OTHER than "current FY or blank" (e.g. if invoice date itself is edited pre-save) was not tested in the sampled cases.
- [ ] Data Import (EDIFACT) invoice creation, Export Manager "Receiving Invoices" and the `Invoice` Data Import log section (TestRail sections 131125, 17494, Importing EDIFACT files 17150) are outside this file; only referenced.
- [ ] **TestRail coverage is partial: 50 of 273 case bodies read.** Read in full: C1231686, C357038, C1009057, C350633 (2026-11-24 refresh); C514954, C514958, C1332438, C613165, C663268, C422074, C471512, C610245, C396373, C700852, C1322894, C825250, C889747, C934328 (2026-07-22 audit); read 2026-11-25: C2299, C2327, C10945, C3453, C15859, C347897, C350728, C353566, C357051, C375959, C399084, C446069, C630444, C10943, C353222, C397985, C442851, C590842, C356846, C353608 and the whole Permissions --> Invoices section 1704 (C196772-C196781, C350624, C353617). Not read (titles/refs only): the rest of sections 329 (about 190 cases, e.g. adjustment-calculation C375998/C375999/C376007, FY cases C388520-C396390, encumbrance matrices C399092-C1045396, currency C436952-C451593, filters C353228-C466123, duplicate edge cases C514955-C514984, DRAFT cases), 1289 (about 14: C10937-C10939, C343345, C10947, C353217, C396358-C396369, C353220, C396387-C396389, C397317, C367942, C414961, C414963, C422071, C422073, C825246, C825247) and the C-ids cited above only through their titles. Exact step wording of unread cases is not reproduced here; statements citing them rely on title/refs plus code/docs.
- [ ] **Acquisition units vs Pay (conflict).** The file previously cited C446069 for the error `...not a member of the specified acquisition unit...`. The case body expects the opposite: an invoice created from an order with an acquisition unit (all restrictions checked), user removed from the unit, `Approve & pay` succeeds, status Paid, a `Payment` and no `Pending payment` transaction (MODINVOICE-516; case flagged unstable/bug-created). C630444 (unit with View off, user not in unit, Approve then Pay) also expects success. en.json still has `invoice.actions.error.userNotAMemberOfTheAcq.order`; no TestRail case verified when it appears. Verify on a live environment.
- [ ] **Acquisition-unit capability names.** Docs (Invoices capability table) still list `UI-Invoice Acq Unit Assignment` (procedural/execute and data/manage); Cypress dictionary and TestRail (C196777, C196778, C357051: "since Trillium") use `UI-Invoice Invoices Acquisitions Units Assign/Manage` (procedural/execute). The table above uses the Trillium names; check which names a given environment exposes.
- [ ] **Invoice-line save error wording.** C353566 (updated 2026-11) expects `Invoice line cannot be saved because <Fund Beta> has no current budget.` (no fiscal year) when adding a POL whose fund has a budget only in a concurrent FY, while C569568/en.json give the `... for fiscal year <b>{fiscalYear}</b>.` variant. Not resolved.
- [ ] **Voucher information labels in C397985.** Step text says the vendor's accounting code appears in `Account number` and the account number in `Accounting code` (hyphen), which looks swapped relative to the XML tags; confirm the real labels on a live environment.
- [ ] **Create/save toast wording.** TestRail steps use `Invoice has been saved` (C353608, C356846) and `Invoice <number> has been saved` (C353566), while the Toast table lists `Invoice has been created` for creation; the ui-invoice en.json key was not matched to the exact create toast.
- [ ] **Capability names not in the Cypress dictionary.** Voucher Export, BatchVoucher, Export CSV, Settings View/Manage and the credentials sets are confirmed by docs and TestRail preconditions only; C196781 spells Settings manage as `data - UI-Invoice Settings - manage` while docs and C353608 use the `settings` type.
- [x] ~~Umbrellaleaf-tagged cases are not present in this subtree~~ — 2026-11-24: recently updated cases exist (C1231686, C1009057 carry custom_release id 20, label not resolved here); Umbrellaleaf tagging not enumerated.
- [x] ~~Jira extraction returned zero Done stories/bugs~~ — 2026-11-24: anonymous REST by project key returned 181 Done issues since 2024-06-01.
- [x] ~~Consortium (Invoices) ECS signals are sparse~~ — 2026-07-22 enrichment confirmed C471510 / C471512; section 37957 holds 4 cases (see ECS section).

> Audit round 2026-07-22 (14 cases read, listed in the last Known Gaps item) added the Duplicate, Version History, Save & keep editing, Batch-group credential tiers, Pro-rate coupling, JPY and Fiscal-year sections above.

---

## Changes since 2026-06-18 (refresh 2026-11-24)

> Change log only; the body above is authoritative. Sources: Jira UINV/MODINVOICE/MODINVOSTO Done since 2026-06-18; ui-invoice CHANGELOG 8.1.0 (in progress); TestRail suite 21 (no new cases; many updated 2026-08..2026-09, e.g. C1231686, C1241224, C1009057, C1332438, C1332520).
- Umbrellaleaf R2 2026 window: UINV-641 (exchanged total for view-only user, Done 2026-11-04), UINV-639 (no `metadata.updatedDate` in version history, 8.1.0), UINV-637 (Connected Tasks/Jobs fourth pane in invoice and invoice-line detail views, 8.1.0), UINV-636 (cancellation note not copied on duplicate, 8.1.0), UINV-634 (invoice not loaded if vendor deleted, 8.0.1), MODINVOSTO-216 (old object in EDIT audit events), MODINVOICE-628/654 (encumbrance fixes, 6.1.3), MODINVOICE-656 won't do.
- Trillium R1 2026 (8.0.0 / 6.1.x): MODINVOICE-639/640, MODINVOICE-631 + UINV-623, UINV-630, UINV-629, UINV-607/616, UINV-577/600, UINV-613, UINV-614, UINV-592, UINV-579, UINV-610, UINV-601 — each described in the body.
