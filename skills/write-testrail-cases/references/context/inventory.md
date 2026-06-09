# FOLIO Inventory App — Business Logic Context

> This file is used by the `write-testrail-cases` skill to understand the business logic of the Inventory app before generating test cases.
> Sources: https://docs.folio.org/docs/metadata/inventory/ and https://docs.folio.org/docs/platform-essentials/item-status/itemstatus/

---

## What is Inventory

The Inventory app manages the library's physical and virtual collections. It stores bibliographic, holdings, and item data in a three-level hierarchy. Inventory is the central hub that integrates with Orders, Check in, Check out, Requests, Data Import, Data Export, Bulk Edit, and MARC Authority apps.

---

## Record Hierarchy

```
Instance (bibliographic record)
  └── Holdings (location + call number)
        └── Item (individual physical piece)
```

- An **Instance** can have multiple Holdings records
- A **Holdings** record can have multiple Item records
- Every Holdings record must be linked to an Instance
- Every Item record must be linked to a Holdings record
- Instance, Holdings, and Item each have their own UUID and HRID

---

## Instance Record

### What it is
The Instance record contains bibliographic and administrative information about a resource. It is what is traditionally called a bibliographic record. Instances are either:
- Created manually in FOLIO (source = **FOLIO**)
- Generated from a MARC record in SRS (source = **MARC**)
- Generated from eHoldings (source = **EPKB**)
- Loaded via Data Import

### Source field behavior
- Source is **system-assigned automatically** — cannot be manually edited
- Source = MARC → an underlying SRS MARC record exists; editing goes through quickMARC
- Source = FOLIO → no underlying MARC record; editable directly in the Instance form
- Source = MARC instances edited via quickMARC — changes cascade to the Instance from the MARC record

### Administrative data fields
| Field | Notes |
|---|---|
| **Suppress from discovery** | Hides instance (and associated holdings/items) from discovery layer |
| **Staff suppress** | Hides from staff who lack `UI-Inventory Instance Staff-Suppressed-Records` capability |
| **Previously held** | Marks that the library once owned this but no longer does; retains record for administrative purposes |
| **Set for deletion** | Marks instance for deletion; sets LDR 05 = 'd' in underlying MARC record |
| **Instance HRID** | System-assigned sequential human-readable ID |
| **Source** | FOLIO / MARC / EPKB — system-assigned |
| **Cataloged date** | Date when considered cataloged |
| **Instance status term** | Cataloged, Uncataloged, Batch loaded, Not yet assigned — configurable in Settings |
| **Mode of issuance** | Bibliographic categorization; auto-set from MARC fixed fields for MARC instances |
| **Statistical codes** | Locally assigned values for filtering/sorting; multiple allowed |
| **Administrative notes** | Free text notes about the record |

### Title data fields
- **Resource title** (required)
- **Alternative titles** (multiple; each requires a type from Settings)
- **Index title** — normalized for browsing/searching (no initial articles)
- **Series statements**
- **Preceding titles** — can be linked to another Instance in FOLIO
- **Succeeding titles** — can be linked to another Instance in FOLIO

### Other Instance sections
- **Identifiers** — ISBN, ISSN, LCCN, etc. (type + value pairs; types configurable in Settings)
- **Contributors** — name + contributor type; one can be marked "Primary"
- **Descriptive data** — publication info, physical description, language, etc.
- **Instance notes** — typed notes (note type + note text + staff only flag)
- **Electronic access** — URLs with relationship types
- **Subjects** — subject headings with source
- **Classification** — classification number + type (LC, Dewey, etc.)
- **Instance relationship** — analytics and bound-with relationships to other instances
- **Related instances** — predecessor/successor relationships

### Creating an Instance
Three methods:
1. **Manually** — Actions → New → fill Instance form
2. **Fast Add** — Actions → New (Fast Add) → minimal form creating Instance + Holdings + Item in one step
3. **Import** — Actions → Import (single record import from external Z39.50/SRU source)

### Set for deletion (soft delete)
- Actions → Set record for deletion (or check "Set for deletion" checkbox in Admin data)
- Sets LDR 05 = 'd' in underlying MARC record
- Sets Suppress from discovery = true, Staff suppress = true automatically
- Record still exists in FOLIO; not permanently deleted
- Can be exported with "Deleted" Data Export job profile within the purge window

### Deleting an Instance
- Instance can only be deleted when it has no Holdings records
- Actions → Delete instance
- For MARC instances: underlying SRS record is also deleted

### Duplicating an Instance
- Actions → Duplicate — creates a copy of the Instance
- For MARC source instances: the duplicate is created as source = FOLIO (no underlying MARC)

---

## Holdings Record

### What it is
Holdings records contain location, call number, and volume information. They tell staff where items are physically located.

### Key Holdings fields
| Field | Notes |
|---|---|
| **Holdings HRID** | System-assigned sequential human-readable ID |
| **Permanent location** (required) | The home location of the holdings |
| **Temporary location** | Temporary location override |
| **Holdings type** | Configurable type (e.g. Physical, Electronic, Serial) |
| **Call number** | Call number prefix + number + suffix |
| **Call number type** | LC, Dewey, NLM, etc. — configurable |
| **Copy number** | Copy identifier |
| **Holdings notes** | Typed notes (note type + note text + staff only) |
| **Electronic access** | URLs with relationship types |
| **Acquisition data** | Acquisition method, order format, receipt status |
| **ILL policy** | Interlibrary loan policy for this holdings |
| **Digitization policy** | Digitization policy flag |
| **Suppress from discovery** | Suppresses this holdings (and its items) from discovery layer |
| **Statistical codes** | Multiple allowed |

### Adding a MARC Holdings record
- Requires: existing MARC 852 field in MARC bib record
- Created via quickMARC or Data Import

### Adding a FOLIO Holdings record
- Actions on Instance → Add holdings → fill Holdings form

### Deleting Holdings
- Holdings can only be deleted when it has no Item records
- If Holdings has items: must delete or move items first

---

## Item Record

### What it is
Item records represent individual physical (or virtual) pieces. They carry barcode, circulation status, material type, and loan information.

### Key Item fields
| Field | Notes |
|---|---|
| **Item HRID** | System-assigned |
| **Barcode** | Physical barcode on the item; used for check out/check in |
| **Status** | Item status — see full table below |
| **Material type** (required) | Configurable (e.g. Book, DVD, Equipment) |
| **Permanent loan type** (required) | Loan policy type (e.g. Can circulate, Non-circulating) |
| **Temporary loan type** | Temporary override |
| **Permanent location** | Location override at item level |
| **Temporary location** | Temporary location override |
| **Copy number** | Copy identifier |
| **Enumeration** | Volume/issue enumeration |
| **Chronology** | Date-based enumeration |
| **Volume** | Volume label |
| **Call number** | Item-level call number (overrides Holdings call number) |
| **Number of pieces** | Count of pieces; if > 1 triggers pop-up at check in |
| **Missing pieces** | Description of missing pieces; triggers pop-up at check in |
| **Item notes** | Typed notes including Check in notes and Check out notes |
| **Statistical codes** | Multiple allowed |
| **Tags** | Organizational tags |

### Effective location (calculated)
The **Effective location** is calculated — NOT directly editable:
```
If Item has Temporary location → Effective location = Item Temporary location
Else if Item has Permanent location → Effective location = Item Permanent location
Else if Holdings has Temporary location → Effective location = Holdings Temporary location
Else → Effective location = Holdings Permanent location
```
Effective location determines which service point the item routes to on check in and which circulation rules apply.

### Creating an Item
- From Holdings row on Instance record: click "Add item"
- Fill required fields: Material type, Permanent loan type
- Status defaults to **Available** when created manually

### Deleting an Item
- Most item statuses allow deletion (subject to dependencies check)
- Items with these statuses **cannot** be deleted: Aged to lost, Awaiting delivery, Awaiting pickup, Checked out, Claimed returned, Declared lost, On order, Paged

---

## Item Status — Complete Table

| Status | Set by | Can check out? | Can request? | Can delete? |
|---|---|---|---|---|
| **Aged to lost** | Automatic (system) | ❌ (cannot override) | ❌ | ❌ |
| **Available** | Inventory (on create), Check in, Data Import | ✅ | Page only | ✅ |
| **Awaiting delivery** | Check in | ✅ | Hold/Recall only | ❌ |
| **Awaiting pickup** | Check in | ✅ | Hold/Recall only | ❌ |
| **Checked out** | Check out | ❌ (already out) | Hold/Recall only | ❌ |
| **Claimed returned** | Users app | ❌ (cannot override) | ❌ | ❌ |
| **Declared lost** | Users app | ❌ (cannot override) | ❌ | ❌ |
| **In process** | Receiving app, Inventory, Data Import | ✅ | Hold/Recall only | ✅ |
| **In process (non-requestable)** | Inventory, Data Import | ✅ (with warning) | ❌ | ✅ |
| **In transit** | Check in, Data Import | ✅ | Hold/Recall only | ✅ |
| **Intellectual item** | Inventory, Data Import | ❌ (cannot override) | ❌ | ✅ |
| **Long missing** | Inventory, Data Import | ✅ (with warning) | ❌ | ✅ |
| **Lost and paid** | Automatic (system after Declared lost + payment) | ❌ (cannot override) | ❌ | ✅ |
| **Missing** | Inventory | ✅ (with warning) | ❌ | ✅ |
| **On order** | Orders app (on PO Open) | ✅ | Hold/Recall only | ❌ |
| **Paged** | Requests app | ✅ | Hold/Recall only | ❌ |
| **Restricted** | Inventory | ✅ (with warning) | ❌ | ✅ |
| **Unavailable** | Inventory | ❌ (cannot override) | ❌ | ✅ |
| **Unknown** | Inventory, Data Import | ✅ (with warning) | Hold/Recall only | ✅ |
| **Withdrawn** | Inventory | ✅ (with warning) | ❌ | ✅ |

### Statuses that require separate capabilities to set manually in Inventory
Each of these is a **procedural capability**:
- In process → `UI-Inventory Items Mark-In-Process`
- In process (non-requestable) → `UI-Inventory Items Mark-In-Process-Non-Requestable`
- Intellectual item → `UI-Inventory Items Mark-Intellectual-Item`
- Long missing → `UI-Inventory Items Mark-Long-Missing`
- Missing → `UI-Inventory Item Mark-As-Missing`
- Restricted → `UI-Inventory Items Mark-Restricted`
- Unavailable → `UI-Inventory Items Mark-Unavailable`
- Unknown → `UI-Inventory Items Mark-Unknow`
- Withdrawn → `UI-Inventory Items Mark-Withdrawn`

### Item statuses do NOT auto-revert
After check in, item status becomes **Available** or **In transit** — FOLIO does NOT restore the previous status. Libraries must implement workflows to manually restore statuses like Restricted after an item circulates.

---

## Searching and Browsing

### Search options
- **Keyword** (default) — searches across multiple fields
- **Contributor** — contributor name
- **Title** / **Identifier** / **Subject** / **Call number** / **Barcode**
- **HRID** / **Instance UUID** / **Authority UUID**
- **Query** (CQL) — advanced structured query language search

### Browse options
- **Call numbers** — browse by call number
- **Contributors** — browse by contributor name
- **Subjects** — browse by subject heading

### Key search filters
- Effective location (item), Holdings permanent location
- Resource type, Format, Mode of issuance, Language
- Source (FOLIO/MARC/etc.)
- Suppress from discovery, Staff suppress
- Item status, Material type
- Statistical code, Tags
- Date range, Date created, Date updated
- Instance status

---

## Moving Items and Holdings

### Move items within an instance
- Move an item from one Holdings record to another on the **same Instance**
- From Holdings row: select item checkbox → Move to holdings (same instance)
- Requires `UI-Inventory Item - Execute` capability

### Move holdings/items to another instance
- Move an entire Holdings record (with all its items) to a **different Instance**
- Actions on source Instance → Move holdings/items to another instance
- Requires `UI-Inventory Holdings - Execute` capability

---

## MARC Editing (quickMARC)

- Instance with source = MARC: Actions → Edit MARC bibliographic record → opens quickMARC
- Holdings with MARC 852: Actions → Edit MARC holdings record → opens quickMARC
- Saves are **asynchronous** — toast "Record has been saved and is in process. Changes may not appear immediately"
- 001 and 999 ff fields are **protected** — cannot be edited in quickMARC

### Overlay source bibliographic record
- Actions → Overlay source bibliographic record
- Imports a new MARC record from external Z39.50/SRU source to replace the current MARC
- Uses Single Record Import job profiles

---

## Actions on Instance Record

| Action | Notes |
|---|---|
| **Edit** | Edit Instance form (FOLIO source) or quickMARC (MARC source) |
| **Duplicate** | Copy Instance; MARC duplicates become FOLIO source |
| **Overlay source bibliographic record** | Replace underlying MARC from external source |
| **Import** | Import single record from external source |
| **New holdings** | Add a Holdings record |
| **New order** | Create a purchase order from this Instance |
| **New request** | Create a request for this Instance |
| **Set record for deletion** | Soft delete (sets LDR 05='d', suppresses) |
| **Delete** | Hard delete (only if no Holdings; requires delete capability) |
| **Save UUIDs** | Download .csv of matching Instance UUIDs |
| **Save CQL query** | Download .cql query file |
| **Export (MARC)** | Export via Data Export app |
| **Export (JSON)** | Export as JSON |
| **Share local instance** | ECS/consortium: share with all members |
| **Version history** | View previous versions of the record |

---

## Version History

- Available on Instance and Holdings records
- Shows previous versions with timestamps and usernames
- Can view but not restore previous versions directly

---

## Fast Add Record

A simplified form to create Instance + Holdings + Item simultaneously with minimal required fields. Used for quick additions at the point of need (e.g. a book being checked in that isn't in the system yet).

---

## In Transit Items Report

- Actions → In transit items report
- Generates a CSV of all items currently with status **In transit**
- Requires `UI-Inventory Items In-Transit-Report` capability

---

## Required Capability Sets (Eureka)

| Action | Capability Set |
|---|---|
| View instances, holdings, items | `Data - UI-Inventory Instance - View` |
| Create instances | `Data - UI-Inventory Instance - Create` |
| Edit instances | `Data - UI-Inventory Instance - Edit` |
| Delete instances | `Data - UI-Inventory Instance - Delete` |
| Create holdings | `Data - UI-Inventory Holdings - Create` |
| Edit holdings | `Data - UI-Inventory Holdings - Edit` |
| Delete holdings | `Data - UI-Inventory Holdings - Delete` |
| Move holdings | `Procedural - UI-Inventory Holdings - Execute` |
| Create items | `Data - UI-Inventory Item - Create` |
| Edit items | `Data - UI-Inventory Item - Edit` |
| Delete items | `Data - UI-Inventory Item - Delete` |
| Move items | `Procedural - UI-Inventory Item - Execute` |
| Mark item Missing | `Procedural - UI-Inventory Item Mark-As-Missing - Execute` |
| Mark item Withdrawn | `Procedural - UI-Inventory Items Mark-Withdrawn - Execute` |
| Mark item In process | `Procedural - UI-Inventory Items Mark-In-Process - Execute` |
| Mark item Restricted | `Procedural - UI-Inventory Items Mark-Restricted - Execute` |
| Mark item Unavailable | `Procedural - UI-Inventory Items Mark-Unavailable - Execute` |
| Mark item Long missing | `Procedural - UI-Inventory Items Mark-Long-Missing - Execute` |
| Mark item Intellectual item | `Procedural - UI-Inventory Items Mark-Intellectual-Item - Execute` |
| Mark item Unknown | `Procedural - UI-Inventory Items Mark-Unknow - Execute` |
| Set record for deletion | `Procedural - UI-Inventory Instance Set-Records-For-Deletion - Execute` |
| Import single bibliographic record | `Procedural - UI-Inventory Single-Record-Import - Execute` |
| Create order from instance | `Data - UI-Inventory Instance Order - Create` |
| View staff suppressed records | `Data - UI-Inventory Instance Staff-Suppressed-Records - View` |
| In transit items report | `Data - UI-Inventory Items In-Transit-Report - Create` |
| All Inventory permissions | `Data - UI-Inventory - Manage` |

---

## Key Business Rules for Test Cases

1. **Record hierarchy is strict** — Holdings must link to an Instance; Item must link to Holdings; no orphan records
2. **Source field is system-assigned and read-only** — FOLIO vs MARC vs EPKB; determines editing path (form vs quickMARC)
3. **MARC source instances edited via quickMARC only** — the Instance form is read-only for MARC-sourced records; must go through quickMARC or Overlay
4. **Duplicating a MARC instance creates a FOLIO source copy** — no underlying MARC is created for the duplicate
5. **Set for deletion is a soft delete** — sets suppress flags + LDR 05='d'; record still exists; can be exported as deleted
6. **Instance delete requires no Holdings** — must delete or move all Holdings first; same pattern: Holdings delete requires no Items
7. **Effective location is calculated, not directly editable** — it follows a 4-level priority hierarchy; determines routing on check in
8. **Item statuses do not auto-revert after circulation** — Available/In transit after check in; libraries must restore statuses like Restricted manually
9. **Each markable item status requires its own separate procedural capability** — a user with Edit items cannot mark items Missing, Withdrawn, etc. without the specific Mark capability
10. **Creating an item manually sets status to Available** — automatic; not configurable on creation
11. **On order status set by Orders app on PO Open** — not settable manually in Inventory; cleared to In process on receiving
12. **Staff suppress requires a specific view capability to see suppressed records** — without `Staff-Suppressed-Records` capability, staff suppressed instances are invisible
13. **Previously held checkbox does not suppress the record** — it's informational only; record remains visible and searchable
14. **Number of pieces > 1 always triggers pop-up at Check in** — same for Missing pieces note; staff must acknowledge before completing check in
15. **Checked out, Claimed returned, Declared lost, Aged to lost** — these four statuses block checkout and cannot be overridden
16. **In process (non-requestable), Long missing, Missing, Restricted, Unknown, Withdrawn** — allow checkout but show a warning that staff must confirm
17. **001 and 999 ff are protected in quickMARC** — cannot be edited; 001 = FOLIO HRID; 999 ff = SRS/Instance UUIDs
18. **quickMARC saves are asynchronous** — changes may not appear immediately after save
19. **Holdings call number is inherited by items** unless item has its own call number; effective call number string includes prefix + number + suffix + copy number
20. **Moving holdings to another instance moves all items with it** — the entire Holdings subtree moves
21. **Intellectual item status = placeholder/dummy record** — no physical object; cannot circulate; used for analytics, e-resources, or catalog management
22. **In transit items report** requires its own separate capability — not included in general view/edit permissions
