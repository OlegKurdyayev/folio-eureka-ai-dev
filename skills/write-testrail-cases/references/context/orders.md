# FOLIO Orders App — Business Logic Context

> This file is used by the `write-testrail-cases` skill to understand the business logic of the Orders app before generating test cases.
> Sources: https://docs.folio.org/docs/acquisitions/orders/ and https://folio-org.atlassian.net/wiki/spaces/FOLIOtips/pages/5669035/Orders

---

## What is Orders

The Orders app manages the full purchase order lifecycle: creating orders, opening them to vendors, receiving materials, and linking to Finance (encumbrances) and Inventory (instances, holdings, items).

---

## Record Hierarchy

```
Purchase Order (PO)
  └── Purchase Order Line (POL)  — one or more per PO
        └── Piece  — individual receiving units (in Receiving app)
```

---

## Purchase Order (PO)

### Order types
| Type | Description |
|---|---|
| **One-time** | Single purchase; order closes when fully received and paid |
| **Ongoing** | Recurring (e.g. subscription); does not auto-close |
| **Ongoing subscription** | Subscription with renewal dates and review periods |

### Workflow status (order lifecycle)
| Status | Description |
|---|---|
| **Pending** | Created but not yet sent to vendor; can be edited freely |
| **Open** | Sent to vendor; encumbrance created in Finance; limited editing |
| **Closed** | Fully received + paid, or manually closed; no new activity |

**Pending → Open**: triggered by user action (Actions → Open) or automatically on import  
**Open → Closed**: automatically when all POLs are fully received + paid; or manually (Actions → Close)  
**Closed → Open**: manually via Actions → Reopen  
**Open → Pending**: manually via Actions → Unopen (releases encumbrances, allows full editing again)

### What happens when a PO is Opened
1. **Encumbrance** created in Finance for each POL fund distribution
2. **Date ordered** set to today
3. **Approval date** set to today (if not already approved)
4. **Inventory records created** based on POL "Create inventory" setting:
   - Attempts to match to existing Instance via Product ID
   - If match found: creates Holdings + Item on existing Instance
   - If no match: creates new Instance + Holdings + Item
5. **POL fields become restricted** (certain fields no longer editable without unopen)

### What happens when a PO is Closed
- Reason for closure is recorded (Complete, Cancelled, or custom reason)
- Encumbrances are released back to the budget
- No new POLs can be added

### Fields editable while Open (selected)
PO level: Vendor, Acquisition unit, Assigned to, Bill to, Ship to, Tags, Notes  
POL level: Receiving note, Receipt status, Payment status, Fund distribution, Unit price, Quantity, Vendor reference number, URL, etc.  
**NOT editable while Open** (require Unopen): Order type, Title, Acquisition method, Order format, Product ID

### Approval
Orders can require approval before opening (configurable in Settings → Orders). If approval is required:
- Order must be approved first (Actions → Approve) before it can be opened
- "Approved" flag visible on PO

---

## Purchase Order Line (POL)

### Order format (determines required fields)
| Format | Description |
|---|---|
| **Physical resource** | Physical material; requires physical details |
| **Electronic resource** | E-resource; requires e-resource details |
| **P/E mix** | Both physical and electronic |
| **Other** | Other format |

### Key POL sections
- **Item details**: Title, Contributor, Product ID (ISBN, ISSN, etc.), Publication date, Publisher, Edition
- **POL details**: Acquisition method, Order format, Receipt date, Receipt status, Payment status, Rush, Collection, Tags
- **Vendor**: Vendor account, Vendor reference number/type
- **Cost details**: Unit price, Quantity ordered, Currency, Discount, Additional cost, Total estimated price
- **Fund distribution**: Which fund(s) cover this line and in what percentage/amount
- **Location**: Holding location(s) for the ordered item(s)
- **Physical resource details**: Material type, Create inventory setting, Material supplier
- **E-resources details**: Access provider, Activation status, Trial, URL

### Inventory interaction ("Create inventory" setting on POL)
| Setting | What is created on Open |
|---|---|
| Instance, holdings, item | Full hierarchy created |
| Instance, holdings | Instance + Holdings only |
| Instance only | Instance only |
| None | No Inventory records created |

### POL matching to Instance on Open
1. If a specific Instance was manually linked: uses that Instance (unless bib data was changed)
2. If bib data was changed after linking: creates new Instance
3. If no Instance linked: matches on Product ID (ISBN/ISSN) to find existing Instance
4. If multiple matches on Product ID: uses first match returned by Inventory API
5. If no match: creates new Instance (source = FOLIO)

### POL status fields
**Receipt status**: Pending / Awaiting Receipt / Partially Received / Fully Received / Receipt Not Required / Cancelled  
**Payment status**: Pending / Payment Not Required / Partially Paid / Fully Paid / Cancelled

### Auto-close logic
PO closes automatically (Reason: Complete) when ALL POLs have:
- Receiving: Fully Received or Receipt Not Required or Cancelled
- Payment: Fully Paid or Payment Not Required or Cancelled

---

## Acquisition Units in Orders

Orders (PO and POL) support Acquisition Units — same behavior as Finance:
- If an Acquisition Unit is assigned to an order, only users in that unit can view/edit it
- Users with "bypass acquisition unit" permission can access all orders
- Visible in search results list: "Acquisition unit" column

---

## Finance Integration

### On Open
- **Encumbrance** created for each POL fund distribution
- Reduces Available balance in the associated budget
- Encumbrance amount = estimated cost of the POL

### On Close / Cancel
- Encumbrances are **released** (funds return to Available)

### On Invoice approval
- **Pending payment** transaction created (linked to POL)
- Encumbrance reduced accordingly

### On Invoice payment
- **Payment** transaction created
- Pending payment resolved

### Restrict encumbrances (on Ledger)
- If enabled: FOLIO checks budget before creating encumbrance on Open; rejects if insufficient
- If disabled: Encumbrance created even if budget is overdrawn (shows overencumbered)

---

## Inventory Integration

### Items created on PO Open
- Item status set to **"On order"** when PO is opened
- Item status changes to **"In process"** when item is received (via Receiving app)
- After receiving and Check in: item status becomes **"Available"**

### Holdings location
- Determined by POL Location field
- If specified location's Holdings doesn't exist: new Holdings created on Open
- If Holdings already exists: item added to existing Holdings

---

## Settings → Orders

Key settings that affect Orders behavior:
- **Opening purchase orders**: require approval before opening (Yes/No)
- **Order number prefix/suffix**: configurable number format
- **PO lines limit**: maximum number of POLs per PO (default: 999)
- **Acquisition methods**: configurable list (e.g. Purchase, Approval, Gift, etc.)
- **Reasons for closure**: configurable list of closure reasons

---

## Search and Filters

### Order search filters
Status (Pending/Open/Closed), Approved, Acquisition unit, Assigned to, Date opened, Order type, Vendor, Tags, Reason for closure, Re-encumber, Subscription, Renewal date, Bill to, Ship to

### Order line search filters
Receipt status, Payment status, Acquisition unit, Acquisition method, Location, Fund code, Order format, Material type, Vendor, Tags, Source, Rush, Access provider, Expected receipt date, Date created/updated

---

## Actions Menu on Orders

| Action | Available when |
|---|---|
| Edit | Pending or Open (limited fields) |
| Open | Pending (with at least 1 POL) |
| Unopen | Open |
| Close | Open |
| Reopen | Closed |
| Approve | Pending (if approval workflow enabled) |
| Duplicate | Any status |
| Delete | Pending only |
| Print | Any status |
| Add PO line | Pending only |
| View/Create invoice | Open or Closed |
| Update encumbrances | Open |

---

## Required Capability Sets (Eureka)

| Action | Capability Set |
|---|---|
| View orders | `Data - UI-Orders - View` |
| Create/Edit orders | `Data - UI-Orders - Create`, `Edit` |
| Delete orders | `Data - UI-Orders - Delete` |
| Open/Close/Reopen orders | `Procedural - UI-Orders Open - Execute` |
| Approve orders | `Procedural - UI-Orders Approve - Execute` |
| Manage acquisition units | `Procedural - UI-Orders Acquisition Units Assign - Execute` |
| View order lines | `Data - UI-Orders Order Lines - View` |
| Create/Edit order lines | `Data - UI-Orders Order Lines - Create`, `Edit` |

---

## Key Business Rules for Test Cases

1. **Cannot open PO without at least one POL** — Actions → Open is disabled until a POL is added
2. **Opening creates encumbrance** — Finance budget Available is reduced immediately on Open
3. **Unopen releases encumbrance** — Available returns to budget when order is uneopened
4. **Certain fields locked while Open** — Order type, Title, Acquisition method, Order format require Unopen to edit
5. **Auto-close requires all POLs complete** — one partially received POL prevents auto-close
6. **Pending orders can be deleted** — Open and Closed orders cannot be deleted via UI
7. **POL matching uses Product ID** — if no Product ID, a new Instance is always created on Open
8. **Changed bib data on POL triggers new Instance** — even if Instance was manually linked
9. **Item status "On order"** — set when PO is opened; changed to "In process" on receipt
10. **Acquisition units gate access** — same as Finance; user must be in the assigned unit
11. **Approval setting is configurable** — some tenants require approval before opening; others don't
12. **Cannot add POLs to Open order** — must Unopen first to add new POLs
13. **Finance budget restrictions apply** — if Ledger has "Restrict encumbrances" = true, Open fails if budget insufficient
14. **One-time orders auto-close** — when fully received + paid; Ongoing orders do not auto-close
