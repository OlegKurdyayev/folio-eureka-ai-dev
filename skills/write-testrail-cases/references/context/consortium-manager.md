# FOLIO Consortium Manager App — Business Logic Context

> This file is used by the `write-testrail-cases` skill to understand the business logic of the Consortium manager app before generating test cases.
> Sources: https://docs.folio.org/docs/consortia/ and https://docs.folio.org/docs/settings/settings_consortium-manager/

---

## What is Consortium Manager

The Consortium Manager app allows libraries operating under **Enhanced Consortial Support (ECS)** to manage multi-tenant FOLIO deployments. It enables central administration of member libraries, shared settings, user affiliations, and cross-tenant visibility of Data Import/Export logs.

**ECS must be enabled** at the tenant level — Consortium Manager is not available in standard single-tenant FOLIO deployments.

---

## Key Terms

| Term | Definition |
|---|---|
| **ECS** | Enhanced Consortial Support — the architecture enabling multi-tenant consortia in FOLIO |
| **Central tenant** | The administrative/primary tenant managing the consortium |
| **Member** | An independent library's FOLIO tenant affiliated with the consortium |
| **Affiliation** | A user's association with a given tenant; "active affiliation" = tenant currently being worked in |
| **Primary affiliation** | The tenant where a user record was originally created; cannot be changed or removed |
| **Shadow account** | Auto-created copy of a staff user record in member tenants when affiliated; becomes inactive if affiliation removed |
| **Shared setting** | A setting managed centrally for ALL members; cannot be edited locally once shared |
| **Local setting** | A setting specific to one member library; not shared consortium-wide |

---

## User Types in ECS

User type is a **required field** in ECS-enabled environments.

| Type | Description |
|---|---|
| **Staff** | Library employees; have username, login to FOLIO, can have multiple affiliations and per-tenant permissions |
| **Patron** | Borrowers; have user records but do NOT log in to FOLIO; assigned primary tenant only; no affiliations with other tenants |
| **Shadow** | Auto-created in central tenant when a staff user is created; displays user's permissions for member tenants; deleted if staff user is deleted |
| **System** | At least one system-level account required to assign other roles/permissions |

---

## Members

- Members = individual library FOLIO tenants affiliated with the consortium
- **Cannot be added or removed** via Consortium Manager UI — only membership details (Code, Name) can be edited
- To add/remove members: requires the Consortia API module permission
- Member list viewable from the central tenant active affiliation
- Settings → Consortium Manager → Membership → edit Code or Name via pencil icon

---

## Affiliations

Every user in ECS must be affiliated with at least one tenant. Primary affiliation = the tenant where the record was created.

### Active affiliation
- User works in one tenant at a time — the "active affiliation"
- Switch via: service point menu (top right) → Switch active affiliation → select tenant → Save & close
- When active affiliation is switched: local data for that tenant + shared consortium data is displayed

### Assign an affiliation
- Staff can only assign affiliations to other users **for tenants they themselves are affiliated with**
- If a staff user is in only one tenant, they will NOT see the Affiliations accordion when editing other users
- Steps: Users app → find user → Edit → Affiliations accordion → Assign/unassign → check member → Save & close
- A shadow account is automatically created in the new member tenant when affiliation is assigned

### Unassign an affiliation
- **Primary affiliation and central tenant affiliation CANNOT be removed**
- Removing an affiliation → user status becomes **Inactive** for that member
- If staff user's primary tenant record is deleted → all shadow accounts in member tenants are also deleted
- Same prerequisite: staff must themselves be in the affiliation they want to unassign from another user

### Permissions per affiliation
- Staff can have **different permissions** in each affiliated tenant
- Managed via: Users app → Edit user → User permissions accordion → select Affiliation from dropdown → manage permissions
- To remove all permissions for an affiliation: click "Unassign all permissions" button or unassign the affiliation entirely
- Staff can only manage permissions for affiliations they themselves are assigned to

---

## Shared Settings

Shared settings are created at the **central tenant** and applied to **ALL members**. They ensure controlled vocabulary consistency across the consortium.

### Critical shared settings rule
- **The Share option cannot be changed after a setting is created and saved**
- To change from shared to local (or vice versa): delete the setting and recreate it with the correct Share option
- When shared: fields Source = "consortium", Member libraries = "ALL", Update = "System Consortia"

### Creating a shared setting (general flow)
1. Switch active affiliation to **central tenant**
2. Open Consortium Manager → select the app (Circulation, Data Import, etc.)
3. Click New
4. Fill in required fields
5. Check **Share** checkbox
6. Save → "Confirm share to all" modal appears: *"Are you sure you want to share [Name] with ALL members?"*
7. Click Confirm → setting shared to all members

### Creating a local (unshared) setting
1. In Consortium Manager → select app
2. Click **Select members** → check member(s) for which to create the local setting
3. Click New → fill fields → do NOT check Share
4. Save → "Confirm member libraries" modal lists the affected members
5. Click Confirm → setting saved for selected members only

---

## Shared Settings — Available by App

### Circulation
- **Request cancellation reasons** — can be shared so all members use the same cancellation reasons

### Data Import
- Shared **job profiles, mapping profiles, action profiles, match profiles** — central management of import workflows across members

### Data Export
- Shared **job profiles** — central management of export workflows

### Inventory
- **Alternative title types**
- **Classification identifier types**
- **Contributor types**
- **Formats**
- **Holdings note types**
- **Holdings sources**
- **Holdings types**
- **Identifier types**
- **Ill policies**
- **Instance note types**
- **Instance status types**
- **Item damaged statuses**
- **Item note types**
- **Loan types**
- **Location**
- **Material types**
- **Nature of content terms**
- **Resource types**
- **Statistical codes**
- **Statistical code types**

### Users
- **Address types**
- **Custom fields**
- **Departments**
- **Patron groups**
- **Waiver reasons**
- **Fee/fine — Manual charge types (owner)**
- **Fee/fine — Payment methods**
- **Fee/fine — Refund reasons**
- **Fee/fine — Transfer accounts**

---

## Cross-Tenant Log Visibility

From the central tenant, staff can view Data Import and Data Export logs for **any member tenant** without switching active affiliation.

### Viewing Data Export logs for a member
1. Switch to central tenant active affiliation
2. Apps → Consortium Manager → Settings pane → Data export
3. Select member from the **Member** dropdown
4. Log columns: File name, Status, Total, Exported, Failed, Job profile, Ended running, Run by, ID

### Viewing Data Import logs for a member
1. Switch to central tenant active affiliation
2. Apps → Consortium Manager → Settings pane → Data import
3. Select member from the **Member** dropdown
4. Log columns: File name, Status, Records, Job profile, Started running, Ended running, Run by, ID

---

## Sharing Local Instance Records

Individual instance records can be shared from a member tenant to the consortium (making them visible to all members):
- From Inventory: Actions → Share local instance with consortium
- Requires `Inventory: Share local instance with consortium` permission

---

## Required Permissions (Note: pre-Eureka permission names used in documentation)

| Permission | Description |
|---|---|
| **Consortia API module - all permissions** | Add/remove members via API |
| **Consortia: Assign and unassign affiliations** | Manage other users' affiliations |
| **Consortia: View affiliations** | View other users' affiliations (read-only) |
| **Consortium manager: Can create, edit and remove settings** | Full settings management (must also have per-tenant permissions) |
| **Consortium manager: Can share settings to all members** | Create settings shared to all members; only editable at central tenant |
| **Consortium manager: Can view existing settings** | View settings of affiliated members |
| **Inventory: Share local instance with consortium** | Share individual instance records consortium-wide |
| **Settings (Consortia): Can view and edit consortia membership** | Edit member details in Settings |
| **Settings (Consortia): Can view consortia membership** | View member list in Settings (read-only) |

---

## Key Business Rules for Test Cases

1. **ECS must be enabled** — Consortium Manager is completely unavailable in non-ECS tenants; all ECS-specific tests require an ECS environment
2. **Share option is irreversible after save** — cannot toggle shared/local after creation; delete and recreate is the only way to change
3. **Shared settings confirmed with "Confirm share to all" modal** — saving with Share checked always triggers a confirmation dialog; tests must include this step
4. **Local settings confirmed with "Confirm member libraries" modal** — saving without Share also triggers a confirmation; different modal content
5. **Primary affiliation cannot be removed** — unassigning all affiliations is impossible; primary stays; tests that try to remove primary should expect failure
6. **Central tenant affiliation also cannot be removed** — same protection as primary affiliation
7. **Unassigning affiliation sets user Inactive for that member** — shadow account deactivated; user can no longer work in that member tenant
8. **Shadow accounts auto-created and auto-deleted** — created when affiliation assigned; deleted when primary staff user deleted; cannot be manually created
9. **Staff can only manage affiliations for tenants they are in** — if user A is only in tenant X, they cannot assign/unassign tenant Y affiliations for anyone
10. **Staff can only manage permissions for tenants they are affiliated with** — same scoping rule applies to permission management
11. **Members cannot be added or removed via UI** — only Code and Name fields are editable; requires API for actual membership changes
12. **Active affiliation determines visible data** — switching affiliation changes which tenant's local data is shown; shared data always visible
13. **Patron users have no affiliations** — patrons exist only in their primary tenant; no cross-tenant access; no shadow accounts
14. **Permissions in Consortium Manager require per-tenant permissions too** — having "Can create, edit and remove settings" is not sufficient alone; user must also have permissions in each member tenant to carry out the actions
15. **Cross-tenant log viewing requires central tenant active affiliation** — cannot view other members' logs from a member tenant affiliation
16. **Inventory shared settings are the largest group** — ~20 controlled vocabulary types can be shared; all follow the same shared/local pattern
17. **Switching active affiliation is done via service point menu (top right)** — not via the main navigation; important for precondition steps in test cases
