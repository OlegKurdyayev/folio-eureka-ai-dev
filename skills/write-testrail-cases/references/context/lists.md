# FOLIO Lists App (FQM) — Business Logic Context

> This file is used by the `write-testrail-cases` skill to understand the business logic of the Lists app before generating test cases.
> Sources: https://docs.folio.org/docs/lists/

---

## What is Lists

The Lists app (powered by **FQM — FOLIO Query Machine**) provides actionable, queryable lists of records across FOLIO apps. Unlike static reports, lists persist, can be refreshed on demand, edited, duplicated, and exported. Each list can contain up to **1.25 million records**.

Key use cases:
- Create custom lists when built-in reports aren't sufficient
- Query across multiple apps and record types in a single list
- Enable workflows and actions on a set of records (e.g. feed results into Bulk Edit)
- Use pre-generated (canned) lists provided by FOLIO
- In ECS-enabled tenants: perform cross-tenant queries

---

## Key Terms

| Term | Definition |
|---|---|
| **FQM** | FOLIO Query Machine — the query engine powering the Lists app |
| **List** | A saved, queryable, refreshable set of records |
| **Record type** | The type of FOLIO data the list is built on (e.g. Items, Invoices, Users) |
| **Field** | The attribute used to query or display records |
| **Operator** | Function applied to match records (equals, contains, is null, etc.) |
| **Value** | The input compared against the field using the operator |
| **Query** | A combination of field + operator + value (can be multi-line with AND/OR) |
| **Record set** | The set of records retrieved by the query |
| **Canned list** | Pre-built list provided with FOLIO (not created by users) |
| **Source** | The FOLIO user account that created the list |

---

## Record Types Supported

Lists can be built on the following record types:

- **Budgets** (with Fiscal year and Fund/Ledger data)
- **Fund with ledger**
- **Holdings**
- **Instances**
- **Invoice lines**
- **Invoices**
- **Items**
- **Loans**
- **Organizations**
- **Purchase order lines**
- **Purchase order lines with titles**
- **Purchase orders**
- **Transactions**
- **Users**
- **Voucher lines with fund**
- **Voucher lines with invoice, fund, organization**
- **Vouchers**

---

## Creating a List

### Required list information fields
- **List name** — required; must be unique per user
- **Record type** — required; determines available query fields and column headings
- **Description** — optional
- **Visibility** — **Shared** (visible to all users) or **Private** (visible only to the creator)
- **Status** — **Active** (refreshable) or **Inactive** (frozen, not refreshable)

### Build a query
- Select one or more fields from the record type's supported fields
- Select an operator per row
- Enter or select a value
- Multiple conditions can be combined with **AND** / **OR**
- Query can be tested before saving: **Test query** runs a preview of matched records
- Lists are not generated until the user runs/refreshes them

---

## List Statuses

| Status | Meaning |
|---|---|
| **Active** | List can be refreshed; query is live |
| **Inactive** | List is frozen; cannot be refreshed; query preserved |

---

## List Visibility

| Visibility | Who can see it |
|---|---|
| **Shared** | All users with Lists view capability |
| **Private** | Only the creator |

---

## Viewing a List

### List detail pane shows:
- **List information** — name, description, visibility, status, source (creator), record type, created/updated dates
- **Query** — the saved query conditions
- **Field headings** — which columns are shown in the result table (customizable)
- **Results table** — records matching the query (shown after refresh)

### Filters in the Lists landing page
- **Record type** — filter lists by record type
- **Status** — Active / Inactive
- **Visibility** — Shared / Private
- **Source** — filter by creator

---

## Refreshing a List

- **Refresh** updates the record set based on the current data in FOLIO
- Results reflect near real-time data
- Inactive lists cannot be refreshed
- Refresh is triggered manually by the user (Actions → Refresh list)
- After refresh: timestamp "Last updated" is updated

---

## Updating a Record Set (Refresh)

The record set in a list is **not live** — it reflects data at the time of the last refresh. Users must manually refresh to see current data.

---

## Editing a List

- Name, description, visibility, status, and query can be edited
- Changing the record type requires building a new query from scratch (changing record type clears existing query)
- After editing, a refresh is needed to update the record set

---

## Duplicating a List

- Actions → Duplicate — creates a copy of the list with a new name
- All fields, query conditions, and column settings are copied
- The duplicate starts as a separate independent list
- Useful for creating variations of existing lists

---

## Deleting a List

- Actions → Delete — permanently removes the list and its saved query
- Deletion requires `Lists (Delete)` capability
- Users with only View or Edit capability cannot delete lists

---

## Exporting a List

- Actions → Export — downloads the current record set as a .csv file
- Export reflects the **last refresh** state of the list (not live data)
- Requires `Lists (Export)` capability
- Large lists (up to 1.25M records) can be exported
- Export is processed and downloaded directly (not via Export Manager)

---

## Query Operators

| Operator | Meaning |
|---|---|
| **equals** | Exact match |
| **not equal to** | Does not match |
| **contains** | Field contains the value (substring) |
| **contains all** | Field contains all specified values |
| **not contains all** | Field doesn't contain all specified values |
| **contains any** | Field contains at least one value |
| **not contains any** | Field contains none of the specified values |
| **starts with** | Field starts with the value |
| **greater than** | Numeric/date comparison |
| **less than** | Numeric/date comparison |
| **greater than or equal to** | Numeric/date comparison |
| **less than or equal to** | Numeric/date comparison |
| **is null/empty** | Field has no value |

---

## Canned Lists (Pre-built)

FOLIO ships with pre-generated lists for common use cases. These:
- Are visible to all users with List view capability
- Cannot be deleted by regular users
- Can be duplicated and customized
- Are refreshed like any other list

---

## Custom Fields (Users)

If custom fields are configured for User records (in Settings → Users → Custom fields):
- Custom fields are **available as column headings** in the list preview
- Custom fields are **NOT available** for building a query in the UI
- They appear in the exported .csv

---

## Cross-Tenant Queries (ECS)

In ECS-enabled (multi-tenant consortium) deployments, the Lists app supports cross-tenant queries — querying data across multiple member institutions. This is a specialized ECS feature and behavior differs from single-tenant deployments.

---

## Integration with Bulk Edit

Lists app and Bulk Edit work together in a common workflow:
1. Create a list in the Lists app with a query to identify the target records
2. Export the list as .csv (UUIDs or other identifiers)
3. Upload the .csv into Bulk Edit using the Identifier method
4. Apply bulk changes to those records

This is the recommended approach for complex record selection before bulk editing.

---

## Required Capability Sets (Eureka)

| Action | Capability Set (Resource - Action) |
|---|---|
| View lists | `Module Lists Enabled - View` |
| Create, edit, refresh lists | `Module Lists Refresh - View` |
| Delete lists | `Module Lists - Delete` |
| Export lists | `Module Lists Export - View` |
| All permissions (admin) | `Module Lists - Manage` |

---

## Key Business Rules for Test Cases

1. **Maximum 1.25 million records per list** — far larger than Bulk Edit's 100k limit; lists are for viewing/exporting, Bulk Edit for changing
2. **List name must be unique per user** — two lists with the same name owned by the same user are not allowed; different users can have lists with the same name
3. **Record type cannot be changed after the query is built** — changing record type clears the query; user must rebuild from scratch
4. **Record set is NOT live** — data reflects the last refresh time; must manually refresh to see current data
5. **Inactive lists cannot be refreshed** — status must be set to Active before running a refresh
6. **Private lists are visible only to the creator** — other users with Lists capability cannot see private lists; only creator can share, edit, delete, or export them
7. **Shared lists are visible to all users with view capability** — but only the creator (or admin) can edit or delete them
8. **Export reflects last refresh, not live data** — if records changed after last refresh, exported .csv will not include those changes
9. **Delete capability is separate from Edit** — a user can edit lists without being able to delete them
10. **Export capability is separate from Edit** — a user can edit lists without being able to export them
11. **Canned lists can be duplicated but not deleted** — institutional pre-built lists are protected
12. **User custom fields are column-only** — can display in list results but cannot be used in query conditions
13. **Test query is optional but recommended** — unlike Bulk Edit where test is required, in Lists it is optional; user can save without testing
14. **Query supports AND/OR logic** — multiple conditions can be combined; important for preconditions when describing complex queries
15. **Duplicate creates independent copy** — changes to the duplicate do not affect the original list
16. **Lists app feeds into Bulk Edit via CSV export** — the typical workflow is: Lists query → export → Bulk Edit identifier upload
17. **Refresh updates "Last updated" timestamp** — this timestamp is visible on the list and is the indicator of data freshness
18. **Cross-tenant queries are ECS-specific** — behavior differs in consortium deployments; single-tenant test cases should not assume cross-tenant features
