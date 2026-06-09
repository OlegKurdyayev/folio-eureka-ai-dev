# FOLIO Data Import App — Business Logic Context

> This file is used by the `write-testrail-cases` skill to understand the business logic of the Data Import app before generating test cases.
> Sources: https://docs.folio.org/docs/data-import/ and https://docs.folio.org/docs/settings/settings_data_import/

---

## What is Data Import

Data Import is the FOLIO app for batch importing records from external files (MARC 21, EDIFACT) into FOLIO. It creates and/or updates records in Inventory, MARC Authority, Orders, and Invoices apps. Two main areas: **Landing Page** (run imports, view logs) and **Settings** (configure profiles, file extensions, MARC field protections).

---

## Supported Record Types

### Can be Created AND Updated
- FOLIO Instances (Inventory)
- FOLIO Holdings (Inventory)
- FOLIO Items (Inventory)
- SRS MARC Bibliographic records
- MARC 21 Authority records

### Can be Created ONLY (not updated)
- FOLIO Orders and Purchase Order Lines
- FOLIO Invoices and Invoice Lines (EDIFACT)

### Cannot be Updated via Data Import
- SRS MARC Holdings records (create only)

---

## Profile Hierarchy (4 levels)

Recommended creation order: Field Mapping → Action → Match → Job

```
Job Profile  (top level — defines the full import workflow)
  ├── Match Profile  (optional — match incoming to existing records)
  │     ├── [if match found] → Action Profile → Field Mapping Profile
  │     └── [if no match] → Action Profile → Field Mapping Profile
  └── Action Profile  (without match — always applies)
        └── Field Mapping Profile
```

### Job Profile
- Entry point — user selects this when running an import
- Required fields: Name, Accepted data type (MARC or EDIFACT)
- Optional: Description, Tags
- Contains: linked Match profiles and/or Action profiles in an Overview accordion
- Match profiles can be **nested** to enable multi-field matching
- Action and Match profiles must exist before linking to a Job Profile
- Can be: edited, duplicated, deleted, tagged, exported/imported via API

### Match Profile
- Defines how incoming records are matched to existing FOLIO records
- Required fields: Name, incoming record format, existing record format, Match criteria (field + field)
- Can match: Instance, Holdings, Item, MARC Bibliographic, MARC Authority
- Reusable across multiple Job Profiles
- Can be: edited, duplicated, deleted, exported/imported via API

### Action Profile
- Defines what action to take on the matched (or unmatched) record
- Required fields: Name, Action type, FOLIO record type
- Action types:
  - **Create** — create a new FOLIO record
  - **Update** — update an existing FOLIO record
  - **Modify** — modify the incoming MARC record before create/update (MARC only)
- FOLIO record types available depend on Action type selected
- Links to one Field Mapping Profile (optional but common)
- An action profile can only link to **one** field mapping profile
- Edits to an Action Profile are reflected everywhere it is used
- Can be: edited, duplicated, deleted, tagged, exported/imported via API

### Field Mapping Profile
- Maps MARC/EDIFACT fields to FOLIO record fields
- Two subtypes:
  - **MARC Field Mapping** — for Inventory, Authority records
  - **EDIFACT Field Mapping** — for Invoices
- Required fields: Name, Record type
- Can be: edited, duplicated, deleted, tagged, exported/imported via API

---

## Default and System Profiles

### System profiles — CANNOT be edited OR duplicated
- quickMARC - Create MARC holdings and inventory holdings field mapping profile
- quickMARC Derive - Create Inventory Holdings
- Default - Create MARC Authority action profile
- Default - Create MARC holdings action profile
- quickMARC - Create Holdings and SRS MARC Holdings job profile
- quickMARC - Derive a new SRS MARC Bib and Instance job profile

### System profiles — CANNOT be edited, but CAN be duplicated
- CREATE MARC Authority field mapping profile
- Inventory Single Record - Default Create/Update Instance field mapping & action profiles
- Inventory Single Record - Default Update MARC Bib field mapping & action profiles
- quickMARC Derive - Create Inventory Instance field mapping & action profiles
- Inventory Single Record - Default match profiles
- Inventory Single Record - Default Create/Update Instance job profiles

### System profiles — CAN be edited AND duplicated
- EDIFACT field mapping profiles
- Default - Create instance action profile
- Default - Create instance and SRS MARC Bib job profile
- Default - Create SRS MARC Authority job profile

### Critical rule
**If a system/default profile is edited, there is NO way to revert to the original defaults.** Always duplicate first and edit the copy.

---

## Import Flow

1. Staff uploads a MARC or EDIFACT file (browse or drag-and-drop)
2. File must have a registered extension (Settings → File Extensions) — if not, upload blocked
3. **Select job profile** window appears → staff select the appropriate profile
4. Profile details panel shows: Name, Accepted data type, Description, Tags, Overview (linked action/match profiles), previous jobs using this profile
5. Actions → Run → **Are you sure?** confirmation dialog → Run
6. Job appears in **Running** accordion with progress
7. Job completes → log row appears in Logs pane

---

## Landing Page Layout

### Left pane: Jobs
- Upload area (browse or drag-and-drop)
- **Running** accordion — currently running jobs with progress bar; can be collapsed
- Job profile selection window with search

### Central pane: Logs
Columns: File name, Status, Records, Job profile, Started running, Ended running, Run by, ID

- **Default view**: most recent logs for non-single-record imports
- Single Record Import logs hidden by default
- All columns sortable
- Actions → **View all** to see all logs including Single Record Imports
- Logs are **never deleted** unless a user with delete capability explicitly deletes them

---

## Log Statuses

| Status | Meaning |
|---|---|
| **Completed** | All records processed successfully |
| **Completed with errors** | Some records processed, some failed |
| **Fail** | No records processed |

### Viewing results (Completed / Completed with errors)
- Click **File name** in Logs → Summary view shows: Created, Updated, Discarded, Error counts per record type
- Clicking a record-level link opens the individual record in FOLIO

### Viewing errors (Fail / Completed with errors)
- Click the **row** (not file name) → per-record error log with error messages

---

## Log Filters (View All)

| Filter | Options |
|---|---|
| **Search** | Keyword (ID + File name), ID only, File name only |
| **Errors in import** | Yes / No |
| **Date** | From / To date range |
| **Job profile** | Select from dropdown |
| **User** | User look-up |
| **Inventory single record imports** | Yes / No |

---

## Detailed Log View

Clicking File name opens the detailed log with a record-level summary:

- **Summary table**: number of Created / Updated / Discarded / Error records per entity type (Instance, Holdings, Item, MARC Bib, Authority, Order, Invoice)
- Clicking the count link → list of individual records with their outcome
- For errors: clicking the title link shows the system error message

---

## File Extensions (Settings)

Located in **Settings → Data Import → File Extensions**

- Controls which file formats are accepted for upload
- If a file has an extension NOT in the list → upload blocked before profile selection
- Each extension: extension value (e.g. `.mrc`, `.dat`), data type (MARC, EDIFACT), blocked/allowed status
- Can be: created, edited, deleted

---

## MARC Field Protections (Settings)

Located in **Settings → Data Import → MARC Field Protection**

- Fields listed here are NOT overwritten during record updates even if incoming record has different data
- Can protect an entire field or a specific field + subfield combination
- **System defaults (cannot be removed)**:
  - `001` — FOLIO Instance HRID
  - `999 ff` — SRS record UUID (`$s`) and FOLIO Instance UUID (`$i`)
- User-defined protections: can be added, edited, deleted
- **If a system default protection is edited, it cannot be reverted** — always duplicate/recreate

---

## Single Record Import (from Inventory)

A special import path triggered from Inventory app (not from Data Import landing page):
- **Actions → Overlay source bibliographic record** — updates existing MARC bib from external Z39.50/SRU source
- **Actions → Find and save** (or Import) — imports a new record from an external source

Single Record Import behavior:
- Logs appear only in **View All** with "Inventory single record imports = Yes" filter
- Not shown in the default Logs pane view
- Uses dedicated system profiles that cannot be edited:
  - Inventory Single Record - Default Create Instance job profile
  - Inventory Single Record - Default Update Instance job profile
  - Inventory Single Record - Default match for existing SRS record
  - Inventory Single Record - Default match for no SRS record

---

## Instance Source Behavior on Import

| Scenario | Result |
|---|---|
| New MARC bib imported | Instance created with source = MARC; SRS record created |
| Existing instance updated via Data Import | SRS MARC record updated; Instance updated from SRS |
| Instance created without MARC (FOLIO native) | source = FOLIO; no SRS record |
| LDR 05 = 'd' in imported MARC | Instance set for deletion; Suppress from discovery + Staff suppress = Yes automatically |

---

## Required Capability Sets (Eureka)

### Data Import app
| Action | Capability Set |
|---|---|
| Upload files, run import, view logs | `Data - UI-Data-Import - Manage` |
| View logs only | `Data - UI-Data-Import - View` |
| Delete import logs | `Data - UI-Data-Import Logs - Delete` |

### Data Import Settings
| Action | Capability Set |
|---|---|
| View settings only | `Settings - UI-Data-Import Settings ReadOnly - View` |
| View, create, edit, delete settings | `Settings - UI-Data-Import Settings - Manage` |

---

## Key Business Rules for Test Cases

1. **4-level profile hierarchy** — Field Mapping → Action → Match → Job; each level must exist before the next can reference it; recommended creation order matters
2. **Match profile is optional** — if no match profile in job profile: all records treated as new (Create action only); no matching attempted
3. **Action profile links to exactly one field mapping profile** — cannot link to multiple; changing the field mapping profile requires editing or duplicating the action profile
4. **Editing an Action Profile propagates everywhere it is used** — changes affect all Job Profiles that contain it; unlike Data Export where profiles lock after use
5. **System defaults cannot be reverted after editing** — always duplicate first; "cannot edit" profiles are fully protected; "can edit" profiles are editable but irreversible
6. **File must have a registered extension** — upload blocked at file selection if extension not in Settings → File Extensions; error shown before profile selection
7. **Single Record Import logs hidden by default** — only visible via View All + "Inventory single record imports = Yes" filter
8. **001 and 999 ff fields are always protected** — cannot be overwritten on any update import; system-level protection, not removable
9. **EDIFACT invoices/orders can only be Created** — no Update action available for these record types
10. **MARC Holdings cannot be updated** — Data Import can only Create MARC Holdings, not update existing ones
11. **Completed with errors = partial success** — some records imported; error log shows which ones failed and why
12. **Fail status = no records created/updated** — clicking row shows system error; no record changes occurred
13. **Logs are never auto-deleted** — unlike many other apps; only a user with `Can delete import logs` capability can delete them
14. **Running accordion shows live progress** — can be collapsed but jobs continue running; important for test preconditions requiring waiting for completion
15. **Delete logs is a separate capability** — not included in the manage capability; must be explicitly assigned
16. **Nested match profiles for multi-field matching** — Job Profile can have match profiles inside match profiles for complex matching logic; if outer match fails → falls through to inner match
17. **LDR 05 = 'd' triggers set-for-deletion** — imported records with this Leader value automatically have suppress flags set; behaves same as manual "Set for deletion" in Inventory
18. **Profile export/import is API-only** — no UI button for exporting/importing profiles; done via FOLIO API; relevant for preconditions involving profile migration between environments
