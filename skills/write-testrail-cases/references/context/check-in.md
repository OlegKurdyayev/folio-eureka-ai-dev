# FOLIO Check in App — Business Logic Context

> This file is used by the `write-testrail-cases` skill to understand the business logic of the Check in app before generating test cases.
> Sources: https://docs.folio.org/docs/access/check-in/checkin/

---

## What is Check in

The Check in app processes items returned to the library at a service point. It closes open loans, updates item statuses, triggers request fulfillment (hold shelf or delivery), routes items in transit to other service points, records in-house use, and handles associated fees/fines.

---

## Core Check in Flow

1. Staff scans item barcode (or types it and presses Enter)
2. FOLIO checks item status and any open requests or loans
3. Appropriate pop-ups appear (if triggered)
4. Staff dismisses pop-ups → item appears in **Scanned Items** table
5. Item status updates based on context (see Item Status Outcomes below)
6. Staff ends session when done → notices sent to patrons

---

## Item Status Outcomes After Check in

| Scenario | Resulting Item Status |
|---|---|
| No open requests, checked in at effective location service point | **Available** |
| No open requests, checked in at a different service point | **In transit** |
| Open page/hold request, checked in at pickup service point | **Awaiting pickup** |
| Open delivery request, staff clicks "Close and check out" | Checked out to patron |
| Open delivery request, staff clicks "Close" only | **Awaiting delivery** |
| Item part of floating collection, checked in at floating location | **Available** (effective location changes) |
| In-house use recorded | **Available** (status unchanged; usage stat recorded) |

---

## Pop-ups That Can Occur During Check in

Pop-ups are **mandatory** — there is no way to disable them. Each must be acknowledged before proceeding.

| Pop-up | When triggered |
|---|---|
| **Item status** | Item is in a status that requires staff attention (e.g. In transit, Awaiting pickup) |
| **Check in notes** | Item has check in notes attached to it |
| **Multiple and/or missing pieces** | Item record has a number of pieces > 1, or has a missing pieces note |
| **Awaiting pickup for a request** | Item is checked in at the patron's requested pickup service point |
| **In transit** | Item needs to be routed to another service point |
| **Route for delivery request** | Item has an open delivery request |

---

## Scanned Items Table Columns

| Column | Description |
|---|---|
| **Time returned** | "Processed as" time (backdated if applicable). Click info icon to see actual check in time |
| **Title** | Title of the item |
| **Barcode** | Item barcode |
| **Effective call number string** | Full call number including prefix, suffix, copy number |
| **Location** | Effective location of the item |
| **In-house use** | House icon appears if in-house use was recorded |
| **Status** | Item status after check in |
| **Actions (…)** | Access additional actions and details |

---

## Actions Menu (…) Per Scanned Item

| Action | Available when |
|---|---|
| **Print transit slip** | Item went In transit during check in |
| **Loan details** | Opens Users app loan details |
| **Patron details** | Opens Users app patron record |
| **Item details** | Opens Inventory app item record |
| **Fee/fine details** | If fees/fines exist for this loan |
| **New Fee/Fine** | Always available after check in |
| **Check in notes** | If check in notes are attached to the item |

---

## Routing to Another Service Point (In Transit)

An item is routed In transit when:
- Checked in at a service point **not assigned** to its effective location
- Item has an open request with pickup at a **different** service point

### In transit flow
1. Scan barcode → **In transit** dialog appears
2. Dialog shows: destination service point
3. Optional: print routing/transit slip (checkbox; default = checked)
4. Click **Close** → item status → **In transit**; transit slip printed if checked

---

## Floating Collections

A floating collection allows items to stay at whichever library service point they are returned to, instead of being routed back to their home location.

### Rules
- Both the item's current location AND the check-in location must be designated as **floating** in Settings → Tenant
- If both are floating: item's effective location **changes** to the check-in location; item becomes Available; no transit slip generated
- If only one is floating (or neither): normal routing applies
- Only one floating location per service point is recommended
- Floating must be enabled at installation level

---

## Hold Shelf Fulfillment (Request at Pickup Service Point)

When a requested item is checked in at the patron's chosen pickup service point:

1. Scan barcode → **Awaiting pickup for a request** dialog appears
2. Dialog shows: patron name, pickup service point
3. Optional: print hold slip (checkbox)
4. Click **Close** → item status → **Awaiting pickup**
5. FOLIO sets **hold shelf expiration date/time** automatically (based on service point settings)
6. Pickup notice sent to patron when the check in **session ends** (if notice policy configured)

**Important:** The pickup notice is NOT sent immediately on check in — it is sent when the session ends.

---

## Delivery Fulfillment (Delivery Request)

A delivery request is triggered when the item is checked in at **any** location (not just a specific service point).

### Two options at the "Route for delivery request" dialog:

**Option 1 — Check out immediately:**
- Click **Close and check out** → Check out window opens → item automatically checked out to patron

**Option 2 — Wait to process:**
- Click **Close** → item status → **Awaiting delivery**
- Staff processes delivery separately

---

## Backdating Check in

Used when the library was closed on the actual return date and fines should not accrue.

1. Click the **pencil icon** next to Date/Time Returned
2. Change date and/or time to the desired backdated date
3. Scan item as normal
4. Scanned Items table shows backdated time in Time returned column
5. Click **info icon** to see actual check in time alongside backdated time

**Effect:** Fees/fines calculated as if item was returned on the backdated date; overdue fines reduced or eliminated accordingly.

---

## In-House Use Recording

Used to record that an available item was used within the library without being checked out (e.g. reference items, items on display).

### Requirements
- Item must be **Available** (not on loan)
- Item must have **no open requests**
- Must be checked in at a service point **assigned to the item's effective location**

### How it works
- Scan barcode → house icon appears in In-house use column
- Item status remains **Available**
- Usage statistic is recorded for reporting purposes

---

## Fees/Fines at Check in

### Automatic fees/fines
- If the item is overdue, overdue fines are calculated automatically on check in
- Fine amount shown in **Time returned** column
- Staff can click **… > Fee/fine details** to view, pay, waive, or cancel the fine

### Manual fee/fine creation
Staff can create a new fee/fine directly from Check in:
1. `… > New Fee/Fine`
2. Select Fee/fine owner (required)
3. Select Fee/fine type (required)
4. Enter amount
5. Optional: add staff note, toggle patron notification
6. Click **Charge only**

---

## Session Management

### Ending a session
- Click **End session** → clears Scanned Items table → patron notices sent
- **Pickup notices** for hold requests are sent at session end (not during check in)

### Auto-end session
- Default: session ends automatically after **3 minutes of inactivity**
- Configurable in Settings → Circulation → Auto-end session
- Can be disabled entirely

### Effect of session end
- Scanned Items table cleared
- All pending notices dispatched to patrons

---

## Staff Slips

Staff slips are printed documents generated during check in. Configured in Settings → Circulation → Staff slips.

| Slip | When printed |
|---|---|
| **Transit slip** | Item goes In transit |
| **Hold slip** | Item checked in at pickup service point (hold shelf fulfillment) |
| **Request delivery slip** | Item has a delivery request |

Each slip has a print checkbox at the pop-up dialog. Staff can opt out per item.

---

## Loan Anonymization

Libraries can configure FOLIO to anonymize loans after check in (removes link between loan record and patron):
- Configured in Settings → Circulation → Loan anonymization
- Option: *Anonymize closed loans immediately after loan closes* — anonymization occurs after the session ends
- Scheduled process run by system administrator

---

## Required Capability Sets (Eureka)

| Action | Capability Set |
|---|---|
| Check in items + backdate | `Data - UI-Checkin - Manage` (Capability Set) |

Note: The Check in capability set does **not** include viewing item or patron details — those require separate Inventory and Users view capabilities.

---

## Key Business Rules for Test Cases

1. **Item checked in at its effective location service point → Available** (if no open requests)
2. **Item checked in at a different service point → In transit** — transit slip dialog appears; destination shown
3. **Hold request at pickup service point → Awaiting pickup** — hold slip dialog; hold shelf expiration date set automatically
4. **Delivery request triggers at any service point** — not tied to a specific pickup location; two resolution options: check out immediately or set Awaiting delivery
5. **Pop-ups are mandatory** — cannot be disabled; Check in notes, missing pieces, item status all trigger pop-ups that must be dismissed
6. **Pickup notice sent at session end, not at check in** — notices batch until End session is clicked
7. **In-house use requires Available item + no requests + same service point** — all three conditions must be met; item status stays Available
8. **Floating collection changes effective location** — both locations must be designated floating; item stays at check-in location without transit slip
9. **Backdating affects fee/fine calculation** — overdue fines recalculated based on backdated return date; actual check in time still recorded and accessible via info icon
10. **Auto-end session after 3 minutes inactivity by default** — configurable; important for test preconditions that assume session stays open
11. **Check in capability does not include patron/item detail viewing** — requires separate Users and Inventory capabilities for the Actions menu options
12. **New Fee/Fine from Check in requires Fee/fine owner and type** — both are required fields; amount is also required
13. **Transit slip can be printed after the fact** — via … > Print transit slip in Actions menu, if item went In transit during the session
14. **Check in notes pop-up is separate from the note visibility** — notes must be reviewed and dismissed before proceeding; cannot be skipped
15. **Anonymization runs after session ends** — not immediately at check in; timing depends on scheduler configuration
16. **Multiple pieces pop-up requires staff verification** — item records with piece count > 1 or missing pieces note always trigger this pop-up at check in
