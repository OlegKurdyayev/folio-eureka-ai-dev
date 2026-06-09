# FOLIO Receiving App — Business Logic Context

> This file is used by the `write-testrail-cases` skill to understand the business logic of the Receiving app before generating test cases.
> Sources: https://docs.folio.org/docs/acquisitions/receiving/

---

## What is Receiving

The Receiving app confirms that materials ordered through the Orders app have been physically delivered to the library. It manages **pieces** — individual receiving units tied to a Purchase Order Line (POL). Receiving is only possible for orders with status **Open** or **Closed**.

---

## Key Terms

| Term | Definition |
|---|---|
| **Receiving title** | The title of the ordered material; may have one or more pieces attached |
| **Piece** | A single receivable unit under a receiving title (e.g. one issue of a journal subscription) |
| **Package** | A PO representing multiple titles; individual titles must be added before they can be received |
| **Receive** | Confirm that a piece has been delivered to the library |
| **Unreceive** | Reverse a received piece back to Expected status |
| **Unreceivable** | Mark a piece that cannot be received (damaged, lost in transit, etc.) |

---

## Receiving Title

### How receiving titles are created
- **Automatically** when a PO is opened (for non-Package POLs with "Create inventory" setting)
- **Manually** for Package POLs — user creates receiving titles in Receiving app and links them to the Package POL

### Receiving title detail view — accordions
- **Title information** — title, publisher, edition, product IDs, subscription dates, claiming settings
- **POL details** — linked POL number, expected receipt date, receiving note
- **Expected** — list of pieces not yet received
- **Received** — list of pieces already received
- **Unreceivable** — list of pieces marked as unreceivable
- **Bound items** — pieces that have been bound together
- **Routing lists** — routing lists attached to this title

---

## Piece

### Piece lifecycle
```
Expected → Received
Expected → Unreceivable
Received → Expected (via Unreceive)
```

### Piece fields (when adding/editing)
**Piece details:**
- Caption — label for this piece (e.g. "v.1", "no.1", "Jan 2024")
- Copy number
- Enumeration
- Chronology
- Display summary
- Barcode
- Call number
- Receiving note
- Request (link to patron request)
- Claiming active / Claiming interval
- Internal note
- Display on holding

**Item details (when creating item on receive):**
- Create item — checkbox; when checked, creates an Item in Inventory on receive
- Material type
- Item status (set automatically to "In process" on receive)
- Location (receiving location)

### Piece status values
| Status | Meaning |
|---|---|
| **Expected** | Piece is expected but not yet received |
| **Received** | Piece has been confirmed as received |
| **Unreceivable** | Piece cannot be received (e.g. damaged, not delivered) |
| **Late** | Piece is past its expected receipt date (displayed as indicator, not a separate status) |

---

## Receiving Workflow

### Receiving a piece — two methods

**Quick receive** (fastest):
1. In the Expected accordion, click the checkbox next to a piece
2. Click "Quick receive" — piece moves to Received immediately with no additional prompts

**Receive** (with details):
1. In the Expected accordion, click the piece row to open it
2. Edit details if needed (barcode, call number, location, etc.)
3. Click "Receive" button — piece moves to Received

### "Must acknowledge receiving note" flag
If the POL has "Must acknowledge receiving note" checked:
- A dialog appears when receiving any piece from that title
- User must click "Continue" to proceed with receiving
- Cannot skip this step

### What happens when a piece is received
1. Piece status → **Received**
2. If "Create item" was enabled on the piece: Item in Inventory status → **In process**
3. POL receipt status updates based on total pieces received:
   - Some received, not all → **Partially Received**
   - All received → **Fully Received**
4. If all POLs reach Fully Received + Fully Paid: PO auto-closes

### Unreceiving a piece
1. In the Received accordion, click the piece
2. Click "Unreceive" — piece moves back to Expected
3. If an Item was created: Item status reverts to **On order**
4. POL receipt status recalculates accordingly

---

## Inventory Integration on Receiving

| "Create item" setting | On Receive | On Unreceive |
|---|---|---|
| Yes (create item) | Item created in Inventory with status "In process" | Item deleted from Inventory; piece returns to Expected |
| No (don't create item) | No item created; Holdings still updated | Piece returns to Expected; no inventory changes |

### After receiving and Check in
Staff should check in the item using the Check in app to complete the technical services workflow:
- Check in updates item status from "In process" to "Available" (or to "Awaiting pickup" if there is an open request)

---

## Package POLs and Receiving

- A Package POL in Orders cannot be directly received in Receiving app
- Titles in the package must first be **added** (either via Orders app "Add package title" or via Receiving app "New" action)
- Each added title then has its own pieces and can be received independently

---

## Claiming

If a piece is overdue (past expected receipt date and claiming is active):
- Piece displays a claiming indicator
- User can claim the piece: Actions → Claim
- Claiming generates a notification/claim to the vendor
- Claiming interval is configurable per title

---

## Routing Lists

- A routing list can be attached to a receiving title
- When a piece is received, the routing list can be printed to accompany the item through an internal routing workflow
- Routing lists contain: patron names, notes, sequence order

---

## Binding

- Multiple physical pieces can be bound together into a single bound item
- Bound pieces move to the **Bound items** accordion
- Binding creates a single Item in Inventory representing the bound volume

---

## Search and Filters

### Main search
Searches across: Title, POL number, Package name, Vendor reference number

### Key filters
- **Order status**: Open, Closed, Pending
- **Receiving status**: Expected, Received, Unreceivable
- **Order type**: One-time, Ongoing
- **Order format**: Physical resource, Electronic resource, P/E mix, Other
- **Location**, **Vendor**, **Material type**
- **Expected receipt date**, **Received date**, **Receipt due**
- **Acquisition unit**, **Rush**, **Bindery active**

**Important**: Receiving is only possible for Open or Closed orders. Pending orders do not appear as receivable.

---

## Exporting Search Results

- Actions → Export — downloads a .csv file
- Can select Title fields and/or Piece fields to include in export

---

## Required Capability Sets (Eureka)

| Action | Capability Set |
|---|---|
| View receiving information | `Data - UI-Receiving - View` |
| Receive and edit pieces | `Data - UI-Receiving - Edit` |
| Create pieces | `Data - UI-Receiving - Create` |
| Delete pieces | `Data - UI-Receiving - Delete` |
| Assign acquisition units to receiving title | `Procedural - UI-Receiving Acq-Units Assignment - Execute` |
| Manage acquisition units on receiving | `Data - UI-Receiving Acq-Units Assignment - Manage` |
| Export search results | `Procedural - UI-Receiving - Execute` |

---

## Key Business Rules for Test Cases

1. **Receiving only available for Open or Closed orders** — Pending orders cannot be received; attempting to receive against a closed order shows a pop-up warning
2. **Piece lifecycle is Expected → Received; reversible via Unreceive** — no skip steps
3. **"Must acknowledge receiving note" blocks receive until confirmed** — dialog must be dismissed with "Continue"
4. **Creating item on receive changes Item status to "In process"** — not "Available"; Check in is needed for that
5. **Unreceiving reverses Item creation** — if item was created on receive, it's deleted on unreceive
6. **Package POLs cannot be received directly** — titles must be added first
7. **POL receipt status updates automatically** — Partially Received when some pieces done; Fully Received when all done
8. **Auto-close triggers from Receiving** — when last piece is received AND all payment complete, PO closes automatically
9. **Quick receive vs Receive** — Quick receive skips the detail form; Receive allows editing piece data before confirming
10. **Check in is a separate step** — receiving marks "In process"; Check in app is needed to make item "Available"
11. **Claiming requires claiming to be active on the title** — configured per receiving title, not globally
12. **Acquisition units on receiving titles** — inherited from the order; can be managed by users with the appropriate capability
