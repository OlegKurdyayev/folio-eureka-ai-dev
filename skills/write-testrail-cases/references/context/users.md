# FOLIO Users App — Business Logic Context

> This file is used by the `write-testrail-cases` skill to understand the business logic of the Users app before generating test cases.
> Sources: https://docs.folio.org/docs/users/

---

## What is Users

The Users app manages user records for both **patrons** (library borrowers) and **library staff**. There is no separate directory for staff — both types live in Users. The difference is that staff users have a username, password, and FOLIO permissions (User roles/capabilities) assigned. User records are the foundation for circulation (loans, requests, fees/fines, blocks) and for managing access to FOLIO.

---

## User Record Sections

1. **User information** — identity fields, patron group, status, expiration date, barcode
2. **Extended information** — enrollment date, external system ID, birth date, request preferences, department, username, password
3. **Contact information** — email, phone, addresses
4. **Custom fields** — institution-configured fields (if set up in Settings)
5. **Patron blocks** — manual blocks preventing circulation activity
6. **Reading room access** — access to restricted reading rooms
7. **Proxy/sponsor** — proxy relationships with other patrons
8. **Fees/fines** — outstanding and historical fees/fines
9. **Loans** — open and closed loans
10. **Requests** — open and closed requests
11. **User roles** — assigned authorization roles (Eureka)
12. **Service points** — assigned service points
13. **Notes** — internal notes
14. **Tags** — organizational tags

---

## Required Fields When Creating a User

- **Last name** (required)
- **Patron group** (required)
- **Status** (required; defaults to Active when patron group selected)
- **Email** (required)
- **Preferred contact** (required)

All other fields are optional unless institution-specific configuration requires them.

---

## User Information Fields

| Field | Notes |
|---|---|
| **Last name** | Required |
| **First name** | Optional |
| **Middle name** | Optional |
| **Preferred first name** | If set, displays instead of First name across FOLIO (Check out, notices, etc.) |
| **Patron group** | Required; determines loan policies, blocks thresholds, fees/fines |
| **Status** | Active or Inactive; tied to expiration date |
| **Expiration date** | Optional; when passed, status automatically changes to Inactive |
| **Barcode** | Library card barcode; used to locate patron at Check out/Check in |
| **User Type** | Patron or Staff; required in ECS tenants and for Reading Room Access |
| **Pronouns** | Optional; displayed after name in Check out |
| **Profile picture** | 100×100px; requires tenant-level setting enabled |

### Status behavior
- **Active** — can log in (staff), check out items, place requests
- **Inactive** — cannot log in, check out, or place requests; patron notices still sent; open fines still included in transfer reports
- Status changes to Inactive automatically when expiration date passes (scheduled process)

---

## Extended Information Fields

| Field | Notes |
|---|---|
| **Date enrolled** | Auto-populated from external system if available |
| **External system ID** | For linking to external identity systems |
| **Birth date** | YYYY-MM-DD format |
| **FOLIO number** | System-generated; read-only |
| **Request preferences** | Hold shelf (default) and/or Delivery |
| **Default pickup service point** | Required if Hold shelf is a request preference |
| **Delivery** | Must be enabled here for patron to use delivery fulfillment in Requests app |
| **Default delivery address** | Required if Delivery is selected as fulfillment preference |
| **Department** | Optional; departments configured in Settings → Users |
| **Username** | Required for Staff (login); optional for Patron (external system linking) |
| **Password** | Set via reset password email; cannot be set directly in the UI |

---

## Contact Information Fields

| Field | Notes |
|---|---|
| **Email** | Required |
| **Phone** | Optional |
| **Mobile phone** | Optional |
| **Preferred contact** | Required; Email / Mail (Primary Address) / Text Message |
| **Preferred email communications** | Optional; Programs / Services / Support (multiple selectable) |
| **Address** | Optional; multiple addresses allowed; one set as primary; Address types configured in Settings |

---

## User Status and Expiration

- Expiration date is **optional** — if blank, user stays Active indefinitely
- When expiration date passes: scheduled process changes status to **Inactive**
- Inactive users: cannot check out, cannot renew, cannot place requests
- Patron notices **continue** to be sent to inactive users
- Open fines **continue** to appear in transfer reports for inactive users
- Staff can **manually change status** to Active (editing user record)
- Staff can **manually edit expiration date** to extend borrowing privileges

---

## Proxy / Sponsor Relationships

- A **proxy** checks out items on behalf of a **sponsor**
- Must be configured on the **sponsor's** user record (Add proxy)
- Or on the **proxy's** user record (Add sponsor)
- When proxy checks out: "Who are you acting as?" dialog appears in Check out
- Loan recorded on sponsor's account
- Requires `Users: Can create, edit and remove proxies` capability to manage

---

## Patron Blocks

Manual blocks created by staff to prevent a patron from performing circulation actions.

### Block fields
- **Block borrowing** — prevents check out
- **Block renewals** — prevents loan renewal
- **Block requests** — prevents placing requests
- **Message** — reason shown to staff at point of service
- **Patron message** — reason shown to patron (in self-service portals)
- **Expiration date** — optional; block auto-expires on this date

### Automated blocks
Configured in Settings → Circulation → Patron blocks templates. Triggered automatically when thresholds are exceeded (e.g. max overdue items, max recalled items, max fines owed). These are separate from manual blocks — they appear in the Patron blocks section but are system-generated.

### Overriding blocks
- Requires `UI-Users Override-Patron-Block` capability (patron blocks)
- Requires `UI-Users Override-Item-Block` capability (item/circulation rule blocks)

---

## Fees/Fines Management

Staff can perform the following on fees/fines from the user record:

| Action | Description |
|---|---|
| **Pay** | Accept full or partial payment |
| **Waive** | Waive full or partial amount |
| **Refund** | Refund already-paid amount |
| **Transfer** | Transfer fine to another account/owner |
| **Cancel (mark as error)** | Cancel the fine as incorrectly charged |
| **Export** | Export fees/fines to CSV |

### Fee/fine statuses
- **Open** — outstanding, not yet paid/waived
- **Closed** — paid, waived, cancelled, or transferred

### Manual fee/fine creation
- Staff can create a fee/fine manually: Actions → New fee/fine
- Requires: Fee/fine owner (required), Fee/fine type (required), Amount (required)
- Optional: Additional information for staff, Notify patron checkbox

### Actual cost lost items
Items declared lost with "Actual cost" fee type require staff to manually apply a bill:
- Appear in **Lost items requiring actual cost** report
- Staff enter the actual cost amount
- Or process without billing (item returned without charge)
- Requires `Users: Can process lost items requiring actual cost` capability

---

## Loan Management

From the user record, staff can manage open and closed loans:

| Action | Capability required |
|---|---|
| View loans | `UI-Users Loans - View` |
| Renew loan | `UI-Users Loans-Renew` |
| Renew through override | `UI-Users Loans-Renew-Override` |
| Change due date | `UI-Users Loans-Due-Date - Edit` |
| Claim returned | `UI-Users Loans-Claim-Item-Returned` |
| Declare lost | `UI-Users Loans-Declare-Item-Lost` |
| Mark claimed returned as missing | `UI-Users Loans-Declare-Claimed-Returned-Item-As-Missing` |
| Anonymize loans | `UI-Users Loans-Anonymize` |
| Add patron/staff info | `UI-Users Loans-Add-Info` |

### Claim returned
- Staff marks item as "Claimed returned" when patron claims they returned it but item not found
- Item status → **Claimed returned**
- Loan remains open
- Fine clock paused (no further overdue fines)
- Staff can search for the item; if found → check in to close loan
- If not found → mark as **Missing** (`Declare claimed returned item as missing`)

### Declare lost
- Staff marks item as definitively lost
- Item status → **Declared lost**
- Lost item fee charged automatically (based on lost item fee policy)
- Loan closed

---

## User Roles (Eureka)

Authorization roles replace permission sets in Eureka. Assigned to users via the User roles accordion.

- Staff users need roles to access FOLIO apps and perform actions
- Patron users typically have no roles (no FOLIO login)
- Roles contain Capabilities and Capability Sets
- Assigned via: Users app → User record → User roles accordion → Add role
- Requires `Users: Can assign and unassign permissions to users` capability to manage

---

## Service Points

Staff users and patrons can have service points assigned:
- **Staff**: service points determine which slips they print, which queues they see
- **Patrons**: default pickup service point for holds
- Service points assigned in the user record
- Requires `Users: Can assign and unassign service points to users` capability to manage

---

## Deleting a User

- Actions → Check for open transactions → if none → can delete
- Cannot delete a user with open loans, open fees/fines, or open requests
- Requires `Users: Can delete user profile if user does not have any open transactions` capability
- Use "Check for open transactions" to verify before attempting delete

---

## Reports Available in Users App

| Report | Description |
|---|---|
| **Lost items requiring actual cost** | Items declared lost needing a manual cost applied |
| **Patron notice print jobs** | View/remove queued patron notice print jobs |
| **Overdue loans report** | List of loans past due date |
| **Claimed returned report** | Items with "Claimed returned" status |
| **Cash drawer reconciliation report** | Financial summary for a service point |
| **Financial transaction detail report** | Detailed fee/fine transaction history |
| **Refunds to process manually report** | Refunds that must be processed outside FOLIO |

---

## Search and Filters

### Search fields
- **Keyword**: Username, First/Preferred First/Last/Middle Name, Email, Barcode, User UUID, External System ID, Custom text fields
- **Barcode** — exact/starts with
- **Last name** — starts with
- **Username** — starts with

### Filters
- **Status**: Active / Inactive
- **Patron group**: configurable groups
- **Departments**: configurable
- **Tags**: requires Tags enabled in Settings

---

## Required Capability Sets (Eureka — selected key ones)

| Action | Capability Set |
|---|---|
| View user profile | `Data - UI-Users - View` |
| Create and edit users | `Data - UI-Users - Create` |
| Edit user profile | `Data - UI-Users - Edit` |
| Delete user | `Data - UI-Users - Delete` |
| View fees/fines and loans | `Data - UI-Users Feesfines - View` |
| Manage fees/fines | `Data - UI-Users Feesfines Actions - Manage` |
| View/manage patron blocks | `Data - UI-Users Patron-Blocks - Manage` |
| Manage proxies | `Data - UI-Users Proxies - Manage` |
| Assign/unassign roles | `Data - UI-Users Perms - Edit` |
| Assign/unassign service points | `Data - UI-Users User-Service-Points - Edit` |
| Create/reset password | `Procedural - UI-Users Reset Password - Execute` |
| Override item blocks | `Procedural - UI-Users Override-Item-Block - Execute` |
| Override patron blocks | `Procedural - UI-Users Override-Patron-Block - Execute` |
| View/edit reading room access | `Data - UI-Users Reading-Room-Access - Edit` |
| View profile pictures | `Data - UI-Users Profile-Pictures - View` |
| Manage profile pictures | `Data - UI-Users Profile-Pictures - Manage` |
| Process lost items (actual cost) | `Procedural - UI-Users Lost-Items-Requiring-Actual-Cost - Execute` |
| Check open transactions | `Data - UI-Users Open-Transactions - View` |
| View requests from user record | `Data - UI-Users Requests - Manage` |

---

## Key Business Rules for Test Cases

1. **Last name, Patron group, Status, Email, and Preferred contact are required** — saving without any of these returns a validation error
2. **Patron group selection auto-sets Status to Active** — staff must manually change to Inactive if needed
3. **Expiration date is optional** — if blank, user stays Active indefinitely; if set, scheduled process changes to Inactive when date passes
4. **Inactive users cannot check out, renew, or place requests** — patron notices and fine transfers still process for inactive users
5. **Password cannot be set directly in the UI** — must use "Send reset password email" action; user sets password via email link
6. **Delivery must be enabled on the user record for delivery requests** — without it, Delivery is not offered as a fulfillment option in Requests app; a delivery address must also be on the record
7. **Proxy relationship must be set on the sponsor's record** — the proxy cannot self-configure the relationship; staff must add it
8. **Patron blocks can expire automatically** — if expiration date is set on the block; manual blocks without expiration persist until staff removes them
9. **Automated blocks are separate from manual blocks** — automated blocks come from Settings → Circulation thresholds; cannot be deleted manually (only overridden)
10. **Overriding patron blocks requires separate capability from overriding item blocks** — two distinct procedural capabilities
11. **Cannot delete user with open transactions** — must check open loans, fees/fines, requests first; "Check for open transactions" action available
12. **User roles (Eureka) replace permission sets** — in Eureka, staff access is managed entirely through Authorization roles assigned in the user record; no direct permission assignment
13. **Username is required for Staff login, optional for Patron** — a patron with a username can be linked to an external system but still won't log into FOLIO
14. **Claim returned pauses overdue fine accrual** — the fine clock stops; if item is found and checked in, loan closes normally; if not found → mark as Missing
15. **Declared lost triggers automatic lost item fee** — based on the lost item fee policy; fee type determines if it's set cost or actual cost
16. **Actual cost lost items require manual staff action** — appear in Lost items requiring actual cost report; staff must apply or waive the cost
17. **Profile pictures require both tenant-level enablement and staff capability** — same pattern as Check out
18. **Preferred first name displays instead of first name throughout FOLIO** — affects display in Check out, patron notices, and any other app that shows the patron name
19. **Service points on user record determine default pickup location** — wrong service point assignment leads to wrong default for hold requests
20. **Custom fields are institution-specific** — only visible if configured in Settings → Users → Custom fields; agent should not assume they exist
