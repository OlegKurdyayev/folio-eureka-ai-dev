# FOLIO Check out App — Business Logic Context

> This file is used by the `write-testrail-cases` skill to understand the business logic of the Check out app before generating test cases.
> Sources: https://docs.folio.org/docs/access/check-out/checkout/

---

## What is Check out

The Check out app allows staff to lend items to patrons. Staff first locate the patron, then scan item barcodes to create loans. The app applies circulation policies (loan periods, blocks, limits) and handles proxy borrowing, due date overrides, and loan comments.

---

## Core Check out Flow

1. Locate patron (scan patron barcode or use Patron look-up)
2. Patron details displayed — review blocks, fees/fines, open loans
3. Scan/enter item barcode → item appears in Scanned Items with due date
4. Handle any pop-ups that appear
5. Repeat for additional items
6. End session → notices sent to patron

---

## Step 1 — Locating a Patron

Two methods:

### Scan/enter barcode
- Scan the barcode from the patron's library card, or type it and press Enter

### Patron look-up
1. Click **Patron look-up**
2. Search by name, email, or username
3. Optional: filter by Status (Active/Inactive) or Patron group
4. Click patron → barcode auto-populated → patron details displayed

### Patron notes pop-up
If the patron's user record has a note configured to display at Check out → note appears as a **pop-up** after patron details load. Must be dismissed before proceeding.

---

## Patron Details Displayed

After locating a patron, the following are shown in the **Scan patron card** pane:

| Field | Notes |
|---|---|
| **Name** | Last name, Preferred first name (or First name if no preferred) |
| **Pronouns** | Displayed in parentheses after name (if filled in user record) |
| **Barcode** | Patron barcode |
| **Profile picture** | If enabled for the tenant + user has permission |
| **Patron group** | The patron's assigned group |
| **Status** | Active or Inactive |
| **User expiration** | Date borrowing privileges expire |
| **Open loans** | Count; clickable link if user has View loans capability |
| **Fees/fines owed** | Total amount; clickable link if user has View fees/fines capability |
| **Open requests** | Count; clickable link if user has View requests capability |
| **Patron blocks** | Displayed if any automated or manual blocks exist |

**Without appropriate capabilities:** Open loans, fees/fines, and open requests show as plain text (not links).

---

## Proxy Borrowing

A **proxy** is a patron who checks out items on behalf of another patron (the **sponsor**).

### Requirements
- The proxy relationship must be set up in the **Users app** on the sponsor's user record first
- Cannot proxy for someone not configured in Users

### Proxy checkout flow
1. Scan proxy patron's barcode
2. **"Who are you acting as?"** dialog appears
3. Select the sponsor's name → click **Continue**
4. Both proxy and sponsor details displayed in separate Borrower boxes
5. Profile pictures of both patron and sponsor shown (if enabled)
6. Loan is recorded against the **sponsor's** account; proxy is noted

---

## Step 2 — Scanning the Item

- Scan or type item barcode → item appears in Scanned Items table with **Due date** and **Time**
- Total items scanned counter increments
- Loan is created immediately on scan

### Flag icon on due date
A **flag icon** appears next to the due date if the item will be due **sooner than expected** due to:
- Service point closure during the normal loan period
- A **recall** on the item (another patron requested it back early)
- Patron account expiration date is sooner than the normal due date

---

## Pop-ups That Can Occur During Check out

| Pop-up | When triggered |
|---|---|
| **Item status** | Item is in a status that may not allow checkout (e.g. In transit, Awaiting pickup) |
| **Check out notes** | Item has check out notes attached |
| **Multiple and/or missing pieces** | Item has piece count > 1 or missing pieces note |
| **Patron block** | Patron has an automated block (e.g. too many overdue items, fines exceed limit) or a manual block |
| **Item block** | Item has a non-circulating loan type or material type block |
| **Override circulation policy** | Staff with override capability can bypass a block — override dialog appears |

### Patron blocks
- **Automated blocks**: triggered by configurable thresholds (max overdue items, max recalled items, max fines owed) — configured in Settings → Circulation → Patron blocks
- **Manual blocks**: created manually on a patron's user record by staff
- Both types block checkout; staff with `Can override item blocks` capability can override

### Overriding blocks
- Requires `UI-Users Override-Item-Block` capability
- Override dialog appears; staff enter a comment/reason and confirm
- Override is logged in the Circulation log

---

## Scanned Items Table

| Column | Description |
|---|---|
| **Due date** | Calculated due date/time per loan policy + flag icon if sooner than expected |
| **Time** | Time of checkout |
| **Title** | Title of the item |
| **Barcode** | Item barcode |
| **Actions (…)** | Additional actions per loan |

---

## Actions Menu (…) Per Scanned Item

| Action | Description |
|---|---|
| **Item details** | Opens Inventory app item record |
| **Loan details** | Opens Users app loan details (renewal, claim returned, declare lost, change due date, comments) |
| **Loan policy** | Opens Settings app loan policy details |
| **Change due date** | Opens dialog to set a new due date/time |
| **Add patron info** | Adds a patron-facing loan comment (visible in notices) |
| **Add staff info** | Adds a staff-only loan comment (not included in notices) |

---

## Changing the Due Date

1. Click `… > Change due date`
2. Enter new date and/or time in **Change Due Date** dialog
3. Click **Save and Close** → due date updated on loan

No restriction on how far forward or back the due date can be set (subject to capabilities).

---

## Loan Comments

Two types of loan comments, both stored on the loan record:

### Patron info comment
- Visible to the patron (can be included as a token in patron notices)
- **Only the most recent** patron info comment is included in notice tokens
- Previous patron info comments are superseded (not deleted — still in Circulation Log)
- Added via `… > Add patron info` or in Loan details

### Staff info comment
- Visible to staff only; NOT included in patron notices
- Old staff info comments **cannot be edited** — add a new comment instead
- Added via `… > Add staff info` or in Loan details
- Recorded in Circulation Log

### Common use cases for loan comments
- ILL transaction numbers
- Reason for a changed due date
- Tracking claimed-returned searches

---

## Due Date Receipts

- Can be printed from `… > Loan details → Print due date receipt`
- Content configured in Settings → Circulation → Staff slips
- Includes borrower and loan due date information

---

## Profile Pictures at Check out

- Requires: profile pictures **enabled** at tenant level + staff user has `UI-Users Profile-Pictures` capability
- Displayed in a 100×100px area to the right of the Borrower box
- If no picture exists: placeholder image shown
- For proxy checkout: profile pictures of **both** proxy and sponsor displayed

---

## Session Management

### End session manually
- Click **End session** → Scanned Items cleared → notices sent to patron

### Auto-end session
- Default: **3 minutes of inactivity** → session ends automatically
- Configurable in Settings → Circulation → Auto-end session
- Can be disabled entirely

### Effect of session end
- Scanned Items table cleared
- All pending patron notices dispatched

---

## Circulation Policy Application

Checkout behaviour is governed by **loan policies** configured in Settings → Circulation. Key policy elements:

| Policy element | Effect on checkout |
|---|---|
| **Loan period** | Determines due date |
| **Renewable** | Whether patron can renew |
| **Recalls allowed** | Whether recalls can be placed on checked-out items |
| **Item limit** | Max items of a type patron can borrow |
| **Fixed due date schedule** | Due date set to a specific calendar date instead of loan period |
| **Grace period** | Extra time before overdue fines start |

The **service point's calendar** (open/closed hours) affects due date calculation — items never come due when the service point is closed.

---

## Required Capability Sets (Eureka)

| Action | Capability Set |
|---|---|
| Check out items (full) | `Data - UI-Checkout - Manage` |
| Check out circulating items only | `Procedural - UI-Checkout Circulation - Execute` |
| View fees/fines as link | `Data - UI-Checkout ViewFeeFines - View` |
| View loans as link | `Data - UI-Checkout ViewLoans - View` |
| View requests as link | `Data - UI-Checkout ViewRequests - View` |
| Override item/patron blocks | `Procedural - UI-Users Override-Item-Block - Execute` |
| View profile pictures | `Data - UI-Users Profile-Pictures - View` |

---

## Key Business Rules for Test Cases

1. **Patron must be located before scanning items** — cannot scan an item without an active patron session; item scan without patron returns an error
2. **Patron status must be Active** — checking out to an Inactive patron is blocked
3. **Patron blocks prevent checkout** — automated or manual blocks halt the scan; override requires `Override-Item-Block` capability + comment
4. **Proxy relationship must be pre-configured in Users** — cannot ad-hoc proxy without setup; "Who are you acting as?" dialog only appears for configured proxies
5. **Loan is recorded against the sponsor, not the proxy** — the sponsor's account is charged; proxy's account is not
6. **Flag icon means due date was adjusted** — indicates service point closure, recall, or patron account expiration shortened the loan period
7. **Recall shortens due date** — if another patron has a recall on the item, the due date may be set sooner than the normal loan period allows
8. **Patron info comment supersedes previous** — only most recent patron info comment appears in notices; old ones still in Circulation Log
9. **Staff info comment cannot be edited** — add a new comment to update; previous comments remain in log
10. **Check out: all permissions does NOT include viewing loans/requests/fees as links** — those require separate View capabilities; without them, counts show as plain text only
11. **Override-Item-Block is a separate capability** — not included in Check out: all permissions; must be explicitly assigned
12. **Profile pictures require both tenant-level setting AND staff capability** — enabling one without the other means pictures don't show
13. **Due date receipts are configured in Staff slips settings** — content is customizable; not hardcoded
14. **Session auto-ends after 3 minutes inactivity by default** — important for preconditions; if a test relies on session staying open, this setting must be considered
15. **Notices sent at End session, not during checkout** — same pattern as Check in; batch dispatch at session end
16. **Service point calendar affects due date** — items won't come due when the service point is closed; due date pushed to next open day/time
17. **Item status pop-up appears for non-standard statuses** — e.g. scanning an item that is In transit or Awaiting pickup triggers a status pop-up; staff must acknowledge before loan is created
