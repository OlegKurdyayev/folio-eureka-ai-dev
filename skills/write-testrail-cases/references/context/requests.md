# FOLIO Requests App — Business Logic Context

> This file is used by the `write-testrail-cases` skill to understand the business logic of the Requests app before generating test cases.
> Sources: https://docs.folio.org/docs/access/requests/requests/

---

## What is Requests

The Requests app allows library staff to create and manage patron requests for items. FOLIO supports two kinds of requests: **Item level** (default, on an individual item record) and **Title level** (on the instance; FOLIO picks the item to fill it). Requests drive the hold shelf, transit, and delivery workflows.

---

## Request Types

| Type | When used |
|---|---|
| **Page** | Item is currently **Available** — staff pull it off the shelf |
| **Hold** | Item is currently **not available** — patron wants it when it becomes available |
| **Recall** | Item is currently **not available** but needed **immediately** — may shorten the current borrower's loan period |

**Note:** FOLIO allows recalls on some non-loaned item statuses. When a recalled item is not currently on loan, the workflow is the same as a Page.

Request types available depend on the item's current status and the circulation rules.

---

## Request Statuses

### Open statuses
| Status | Meaning |
|---|---|
| **Open - Not yet filled** | Request created; item not yet retrieved; expiration date (if set) is in the future |
| **Open - In transit** | Item checked in and en route to the patron's pickup service point |
| **Open - Awaiting pickup** | Item checked in at the patron's pickup service point; waiting for patron to collect |
| **Open - Awaiting delivery** | Delivery request; item checked in but not yet checked out to requester |

### Closed statuses
| Status | Meaning |
|---|---|
| **Closed - Filled** | Patron checked out the item (hold shelf) OR delivery request was closed and checked out |
| **Closed - Cancelled** | Request cancelled by staff or patron before or after item became available |
| **Closed - Pickup expired** | Item reached hold shelf but patron didn't pick it up before hold shelf expiration date |
| **Closed - Unfilled** | Request expiration date passed while request was still open (not yet filled) |

**Exception:** Open - Awaiting pickup requests remain open until the **Hold shelf expiration date** passes (not the Request expiration date).

---

## Item Level Requests

### Creating an item level request
1. Actions → New
2. Ensure "Create title level request" is **not** checked
3. Scan/enter item barcode
4. Scan/enter requester barcode (or use Requester look-up)
5. Select **Request type** (options depend on item status)
6. Optional: Set **Request expiration date**
7. Optional: Enter **Patron comments**
8. Select **Fulfillment preference**: Hold shelf or Delivery
9. Select **Pickup service point** (or Delivery address for delivery)
10. Save & close → patron receives "request received" notification (if notice policy configured)

### Request expiration date
- Optional field
- If set and request is still Open - Not yet filled when date passes → status → **Closed - Unfilled**
- If NOT set → request never auto-closes to Unfilled
- **Exception:** Open - Awaiting pickup requests ignore the Request expiration date; they close only when Hold shelf expiration date passes

### Editing requests
- Only **open** requests can be edited
- Closed requests cannot be edited

### Duplicating requests
- Open requests can be duplicated, but requester **or** item must be changed (cannot have two requests on the same item by the same requester)
- Closed requests can be duplicated, but requester barcode must be re-entered

### Moving a request to another item (same instance)
- Actions → Move request
- Available for moving between items on the **same instance** only
- Use case: item goes missing, need to balance queue
- If moved to a loaned item that wasn't recalled: loan is recalled → patron gets recall notice → due date may be shortened or extended per loan policy
- If current request type not allowed on target item: staff must select an allowed type in confirmation dialog

### Cancelling requests
- Only **open** requests can be cancelled
- Cancellation requires selecting a **Reason for cancellation** (configurable in Settings)
- If reason = "Other": additional information is **required**
- After cancel: status → **Closed - Cancelled**; patron receives cancellation notice (if configured)
- If a Page request is cancelled and no other requests exist in queue: item status → **Available**
- If cancelled while In transit or Awaiting pickup: request status changes to Closed - Cancelled, but **item status does NOT change** until item is checked in

---

## Title Level Requests

Must be enabled in **Settings → Circulation → Title level requests**.

### Creating a title level request
1. Actions → New
2. Check "Create title level request"
3. Use Title look-up to find the instance (requires `Inventory: View instances` capability)
4. Enter requester barcode or use Requester look-up
5. Select request type, expiration date, patron comments, fulfillment preference, pickup service point
6. Save & close

### How FOLIO decides which item fills a title level request
1. Checks for an **Available** item on the instance
2. If available and circulation rule allows paging → request becomes a Page for that item
3. If multiple items available: FOLIO prefers items whose **effective location's primary service point** matches the patron's **pickup service point**
4. If no match at pickup service point: looks at same Library → same Campus → same Institution (location hierarchy)
5. If no available item: request stays **Open - Not yet filled** until an item becomes available (returned, checked in)

### Title level requesting — important limitations and rules
- Cannot turn off title level requesting while any open title level requests exist
- Title level requests **not supported for multi-volume sets** — must use item-level requests instead
- Items newly added via Receiving, Inventory, or Data Import must be **checked in** before they are available to fill open title level requests
- Setting "Fail to create title level hold when request is blocked by circulation rule":
  - If **ON**: title level hold requests follow circulation rules; blocked holds are not created
  - If **OFF**: title level hold requests go through even when holds are blocked by circulation rules; request stays Open - Not yet filled since no item can be associated

---

## Request Queue

### Queue behavior
- Each item has its own request queue
- Requests are served in queue order (FIFO) unless manually reordered
- **Page requests cannot be moved above other Page requests** in the queue

### Reordering the queue
- Actions → Reorder queue
- Drag and drop to reposition requests
- Requires `Requests: Reorder queue` capability (separate from general edit capability)
- Moving requests does NOT require Move request capability (that's for moving between items)

---

## Fulfillment Preferences

### Hold shelf (default)
- Item delivered to a service point's hold shelf
- Patron picks up in person
- Requires selecting a **Pickup service point**

### Delivery
- Item delivered directly to the patron (mailing address or campus delivery)
- Requires patron to have **delivery enabled** on their user record
- Staff can enable delivery on the user record manually (Users app → Edit → Delivery)
- Requires a **Delivery address** on the patron's user record

### Processing a delivery request at Check in
Two options at the "Route for delivery request" dialog:
- **Close and check out** → item checked out to patron immediately → status → Closed - Filled
- **Close** → status → Open - Awaiting delivery (item held, not yet checked out)

---

## Hold Shelf Clearance

When a request expires on the hold shelf (Closed - Pickup expired) or is cancelled while Awaiting pickup:
- Item appears on the **Hold shelf clearance report**
- Export: Actions → Export hold shelf clearance report (CSV)
- Staff use this to identify items on hold shelf that need to be re-shelved

---

## Page Requests — Collecting and Printing

### Pick slips (Page requests)
- Actions → Print pick slips → prints all outstanding Page request pick slips for the current service point
- Also: Actions → Generate page requests CSV export

### Hold request search slips
- Actions → Print hold request search slips → for Hold requests where item needs to be searched
- Also: Actions → Generate hold requests CSV export

---

## Search and Filters

### Search options
- Keyword search by title
- "Starts with" search by: item barcode, requester barcode, item call number

### Filters
- **Request type**: Hold, Page, Recall
- **Request status**: Open - Not yet filled, Open - In transit, Open - Awaiting pickup, Open - Awaiting delivery, Closed - Filled, Closed - Cancelled, Closed - Pickup expired, Closed - Unfilled
- **Request level**: Item, Title
- **Tags**
- **Pickup service point**
- **Retrieval service point**
- **Print status** (requires Settings → Circulation → View print details enabled)

### Export search results
- Actions → Export search results to CSV
- Exports all visible columns + additional request and item fields

---

## Remote Storage Requests

Items stored in remote storage (e.g. off-site warehouse) can be requested. When a remote storage item is requested, the system triggers a retrieval workflow to the remote storage system. The item is retrieved and routed to the pickup service point.

---

## Required Capability Sets (Eureka)

| Action | Capability Set |
|---|---|
| View requests | `Data - UI-Requests - View` |
| View and create requests | `Data - UI-Requests - Create` |
| View, edit, and cancel requests | `Data - UI-Requests - Edit` |
| All request functions | `Data - UI-Requests - Manage` |
| Move request to another item / reorder queue | `Procedural - UI-Requests MoveRequest - Execute` |
| Reorder queue only | `Procedural - UI-Requests ReorderQueue - Execute` |
| Title look-up when creating title level request | `Data - UI-Inventory Instance - View` |

---

## Key Business Rules for Test Cases

1. **Request type options depend on item status** — not all types (Hold/Page/Recall) are available for every item status; available types determined by circulation rules
2. **Page = available item; Hold = unavailable; Recall = urgent, may shorten loan** — these definitions drive expected results in test steps
3. **Request expiration date is optional** — if not set, Open - Not yet filled requests never auto-close to Unfilled; if set, they close on that date
4. **Awaiting pickup ignores request expiration date** — only Hold shelf expiration date closes Awaiting pickup requests
5. **Closed requests cannot be edited or cancelled** — only open requests allow edit/cancel actions
6. **Cancelling Page request with empty queue returns item to Available** — if other requests exist in queue, item stays Paged/In transit
7. **Cancelling In transit or Awaiting pickup request: item status unchanged until check in** — request goes Closed - Cancelled but item must be physically checked in to update status
8. **Cannot duplicate same requester + same item combination** — must change at least one to save a duplicate
9. **Moving request between items triggers recall if target item is on loan** — patron gets recall notice; due date may change
10. **Title level requests require "check in" before newly added items fill them** — items added via Receiving, Inventory, Data Import must be checked in first
11. **Title level requests cannot be disabled while open TLRs exist** — settings change is blocked
12. **Title level requests not supported for multi-volume sets** — must use item-level for those
13. **Delivery requires delivery enabled on patron record + delivery address** — without both, delivery fulfillment preference cannot be used
14. **Hold shelf clearance report identifies expired/cancelled hold shelf items** — items on hold shelf that need re-shelving; exported as CSV
15. **Page requests cannot be moved above other Page requests in queue** — queue reordering has this constraint
16. **Reorder queue and Move request are separate capabilities** — having one does not grant the other
17. **Title look-up for title level requests requires Inventory: View instances capability** — without it, the Title look-up button doesn't work
18. **Pickup notice sent at check in session end** — same as Check in behavior; not sent immediately when item reaches hold shelf
19. **"Other" cancellation reason requires additional information text** — mandatory field when reason = Other
20. **Recall may shorten current borrower's due date** — depends on loan policy settings; flag icon appears on loan in Check out
