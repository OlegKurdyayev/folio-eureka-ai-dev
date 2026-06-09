# FOLIO MARC Authority App — Business Logic Context

> This file is used by the `write-testrail-cases` skill to understand the business logic of the MARC Authority app before generating test cases.
> Sources: https://docs.folio.org/docs/metadata/inventory/marcauthority/ and https://docs.folio.org/docs/settings/settings_marc_authority/

---

## What is MARC Authority

The MARC Authority app manages authority records in FOLIO. Authority records establish the authorized form of names, subjects, and other controlled terms used in bibliographic (bib) records. When an authority heading changes, linked bib fields are automatically updated. All authority records are stored in **Source Record Storage (SRS)** in MARC 21 format.

---

## Key Terms

| Term | Definition |
|---|---|
| **Authority record** | A record that establishes the authorized form of a name, subject, or subject subdivision; also stores references and interrelationships |
| **Authorized access point** | The heading from the 1XX field — the approved form of the term |
| **Heading** | Same as access point — the searchable element (name, title, subject, etc.) |
| **Reference** | A 4XX (See from) or 5XX (See also from) entry pointing to the authorized heading |
| **Authority source** | The authority file the record belongs to (e.g. LC Name Authority File, MeSH); configured in Settings |
| **Number of titles** | The count of bib records controlled by (linked to) this authority record |

---

## Authority Record Structure

Every authority record has:
- **UUID** — system-assigned unique identifier
- **HRID** — human-readable ID (format determined by authority file prefix + sequence)
- **LDR** — Leader: only positions 05 (Record status), 06 (Type of record, must = 'z'), 17 (Encoding level), 18 (Punctuation policy) are editable
- **001** — system-supplied HRID; non-repeatable; **cannot be edited or deleted**
- **010 $a** — LCCN; non-repeatable; field 010 can only contain one $a; used for $0 linking in bib records
- **1XX** — authorized heading (required and non-repeatable); type determined by tag:

| 1XX Tag | Type of heading |
|---|---|
| 100 | Personal name |
| 110 | Corporate name |
| 111 | Conference/Meeting name |
| 130 | Uniform title |
| 150 | Topical subject |
| 151 | Geographic name |
| 155 | Genre/Form |
| 162 | Children's subject heading |

- **4XX** — "See from" references (variant forms → authorized heading)
- **5XX** — "See also from" references (related authorized headings)
- **999 ff** — SRS record UUID (`$s`) + authority UUID (`$i`); system-supplied; **cannot be edited or deleted**

---

## Creating Authority Records

### Via Data Import (batch)
- Use the "Default - Create SRS MARC Authority" job profile (or a custom authority create profile)
- Preferred method for loading large sets of authority records

### Via quickMARC (single record)
1. Actions → New → quickMARC editor opens with a blank authority record template
2. Fill in LDR, 001 (auto-filled), required 1XX field, and any additional fields
3. Save & close
- Requires `quickMARC: Create a new MARC authority record` capability

---

## Editing Authority Records (quickMARC)

1. Open the authority record → click **Edit** (top right of third pane)
2. quickMARC editor opens in edit mode

### Actions available in quickMARC
- **Add field** — click `+` icon; new field appears below
- **Delete field** — click trash can icon
- **Move field up/down** — up/down arrow icons
- **Edit field text** — click into field; use `{dollar}` to enter a literal `$` (not a subfield delimiter)
- **Save & close** — saves and closes; confirmation toast appears

### quickMARC validations and restrictions
- Subfields delimited by `$`
- Tags must be exactly 3 numeric characters
- Indicators must be single characters
- **LDR** — only positions 05, 06, 17, 18 editable via dropdowns; position 06 must always = 'z'
- **001 and 999 ff** — system-supplied; cannot be edited or deleted
- **010** — non-repeatable; can only contain one $a
- **1XX** — required; non-repeatable
- Additional validation rules can be configured at tenant level via mod-record-specifications API
- **Validation rules apply ONLY when editing in quickMARC** — records imported via Data Import bypass these validations

### After saving
- Toast: *"Record has been saved and is in process. Changes may not appear immediately."*
- Save is **asynchronous** — the update propagates to linked bib fields in the background

---

## Deleting Authority Records

1. Open authority record → Actions → Delete
2. **Confirm deletion of authority record** dialog → click Delete
3. Record deleted; toast confirms

### What "deleted" means in FOLIO
- UI deletion is a **soft delete then hard delete**:
  - Record immediately disappears from search results
  - SRS marks the record with **LDR 05 = 'd'** (set for deletion)
  - Record stays in SRS for a configurable **purge window** (default: **7 days**)
  - After purge window: record permanently removed from SRS
- During the purge window: record can be exported using the **Deleted authority export job profile** in Data Export
- After purge: record no longer exists → export returns "Record not found" error

### Effect on linked bib records
- When an authority record is deleted: all linked bib fields are **unlinked**
- The heading text remains in the bib field as plain text
- `$0` and `$9` subfields are removed from the bib field
- The bib record itself is **NOT deleted**

### Export rules after deletion

| State | Default authority export profile | Deleted authority export profile |
|---|---|---|
| Record exists, not deleted | ✅ Exports successfully | ❌ Error: "This profile can only be used to export authority records set for deletion" |
| Record set for deletion (within purge window) | ❌ Error: "Authority record \<UUID\> is set for deletion and cannot be exported using this profile" | ✅ Exports successfully |
| Record permanently purged (after 7 days) | ❌ Error: "Record not found" | ❌ Error: "Record not found" |

---

## Searching for Authority Records

### Search options (select from dropdown)
- **Keyword** — searches title, contributor, identifier fields
- **Identifier (all)** — all identifier types
- **LCCN** — Library of Congress Control Numbers (010 $a)
- **Personal name** — ordered: Last Name, First Name, Initial
- **Corporate/Conference name**
- **Geographic name** — ordered: City, State/Province, Country
- **Name-title** — ordered: Last Name, First Name, Title
- **Uniform title**
- **Subject** — single term, phrase, or group of terms
- **Children's subject heading**
- **Genre**
- **Advanced search** — multi-field with AND/OR/NOT operators and Exact phrase/Contains all/Starts with/Contains any modifiers

### Search filters
| Filter | Notes |
|---|---|
| **Authority source** | Filter by authority file name; configurable in Settings |
| **References** | Exclude "See from" and/or "See also from" references |
| **Thesaurus** | Based on 008 field position 11 (Subject heading system/thesaurus) |
| **Type of heading** | Personal name / Corporate name / Conference name / Topical / Geographic name / Genre / Uniform title / Other |
| **Date created** | Date range filter |
| **Date updated** | Date range filter |

Filter logic: multiple values in same filter = OR; multiple different filters = AND.

---

## Browsing Authority Headings

Browse provides a headings list (alphabetical walk-up browse) rather than keyword search.

### Browse options
Personal name, Corporate/Conference name, Geographic name, Name-title, Uniform title, Subject, Genre

### Browse filters
Authority source, References (exclude "See from"), Type of heading

### Browse result display
- The searched heading appears in context (alphabetical surroundings)
- **Authorized** headings shown normally
- **References** (See from / See also from) shown with indicator
- Clicking any heading opens the full MARC record

---

## Search Results List Columns
- **Authorized/Reference** — indicates if the heading is authorized or a reference
- **Heading/Reference** — the heading text (clickable → opens full record)
- **Type of heading** — from 1XX tag
- **Authority source** — which authority file
- **Number of titles** — count of linked bib records (clickable → opens list of linked bibs in Inventory)

---

## Exporting Authority Records

1. Search/filter for records → select checkboxes
2. Actions → **Export selected records (CSV/MARC)**
3. UUID .csv downloads immediately to browser
4. MARC .mrc file generated asynchronously → logged in Data Export app → download from there

Note: Selecting a **reference** checkbox in results counts toward the export but the underlying authority record is what gets exported (not the reference entry separately).

---

## Reporting

Two reports available via Actions menu, retrieved from **Export Manager**:

### MARC authority headings updates (CSV)
- Shows which authority records' **1XX fields** were updated in a date range
- Fields: date/time updated, original heading, new heading, original 1XX, new 1XX, authority source file, number of linked bibs, updater username
- Generate: Actions → MARC authority headings updates (CSV) → enter date range → Export → retrieve from Export Manager

### Failed updates: linked bibliographic fields (CSV)
- Shows when a 1XX or 010 $a update **failed to propagate** to linked bib fields
- Fields: date/time of failed link, bib title, bib UUID, MARC tag of failed bib field, linked authority identifier, reason for error
- Generate: Actions → Failed updates: linked bibliographic fields (CSV) → retrieve from Export Manager (no date range required)

---

## Authority Control (Linking to Bib Records)

### How linking works
- Bib fields eligible for authority control: 100, 110, 111, 130, 240, 600, 610, 611, 630, 650, 651, 655, etc.
- Link stored via:
  - **$0** — authority record identifier/URI (010 $a value from the authority record)
  - **$9** — internal FOLIO authority UUID (system-managed; never manually add)

### $9 subfield rule
**Never import or manually add $9** to bib fields. It is FOLIO's internal linking identifier; manually adding it causes broken linking behavior.

### Automatic propagation
When a 1XX heading is updated via quickMARC or Data Import:
1. All linked bib fields automatically update to match the new heading
2. If the change causes incompatibility (e.g. required subfield removed from 1XX), the bib field is **unlinked**

### 010 $a update propagation
When 010 $a (LCCN) is changed:
- The $0 subfield in all linked bib fields is updated to the new value
- This keeps the identifier link accurate

### Unlinking conditions
A bib field becomes unlinked when:
- The authority record is deleted
- A required subfield is removed from the authority 1XX (e.g. removing $t from 100 unlinks bib 240 which requires $t)
- Staff manually removes $0 or $9 from the bib field in quickMARC

---

## Version History

- Available on each authority record
- Click the **clock icon** (top right of the third pane) to open version history
- Each version shows: source (user who made the change), which fields were edited/added/deleted
- Click a version name → popup shows each action, field, and specific change
- Configuring number of cards per page: requires `Settings (MARC authority): Configure version history` capability

---

## Settings → MARC Authority → Manage Authority Files

Authority files define which authority sources are available in FOLIO.

### Authority file fields
- **Name** — display name of the authority file
- **Prefix** — prefix for HRIDs in this file; alphabetical only; max 25 chars; must be unique across all authority files
- **HRID starts with** — starting number for HRID sequence
- **Base URL** — URL of the external authority source (optional); used to construct $0 identifiers
- **Active/Inactive** — active files appear as options in search filters and during import; inactive files remain but are hidden from active selection

### Authority file CRUD
- Create: New button → fill fields → Save
- Edit: select file → edit → Save
- Delete: select file → Delete (only if no records are assigned to it)
- Cannot delete a file that has records assigned to it; must mark inactive instead

### Settings capabilities
| Action | Capability Set |
|---|---|
| Access MARC Authority settings pages | `Settings - Settings Marc-Authorities Enabled - View` |
| View authority files | `Settings - UI-Marc-Authorities Settings Authority-Files - View` |
| View, create, edit, delete authority files | `Settings - UI-Marc-Authorities Settings Authority-Files - Manage` |
| Configure version history display | `Settings - UI-Marc-Authorities Settings Version-History - View` |

---

## Required Capability Sets (Eureka — MARC Authority app)

| Action | Capability Set |
|---|---|
| View authority records | `Data - UI-Marc-Authorities Authority-Record - View` |
| Edit authority records | `Data - UI-Marc-Authorities Authority-Record - Edit` |
| Delete authority records | `Data - UI-Marc-Authorities Authority-Record - Delete` |
| Create authority records | `Data - UI-Marc-Authorities Authority-Record - Create` |
| View and edit in quickMARC | `Data - UI-Quick-Marc Quick-Marc-Authorities-Editor - Manage` |
| Create new record in quickMARC | `Data - UI-Quick-Marc Quick-Marc-Authorities-Editor - Create` |

---

## Key Business Rules for Test Cases

1. **1XX is required and non-repeatable** — saving/creating an authority record without a 1XX field returns a validation error in quickMARC
2. **001 and 999 ff cannot be edited or deleted** — system-supplied; attempts to modify them in quickMARC are blocked
3. **010 is non-repeatable and can only contain one $a** — adding a second $a to 010 is a validation error
4. **LDR position 06 must always = 'z'** — enforced in quickMARC; this is what classifies the record as an authority record
5. **Validation rules in quickMARC do NOT apply to Data Import** — records loaded via Data Import bypass quickMARC validations; invalid records may still import
6. **Deletion is soft then hard** — deleted record stays in SRS for ~7 days (LDR 05='d'); can be exported using Deleted authority export profile within this window
7. **After purge window: "Record not found"** — even the correct deleted export profile fails; record is permanently gone from SRS
8. **Default export profile rejects deleted records; Deleted export profile rejects non-deleted records** — using the wrong profile always results in per-record errors in the export log
9. **Deleting an authority record unlinks bib fields** — heading text stays in bib; $0 and $9 removed; bib record NOT deleted
10. **Never manually add $9 to bib fields** — it's FOLIO's internal UUID link; adding manually causes broken behavior
11. **1XX updates propagate automatically to linked bibs** — async process; may not appear immediately; check via "Failed updates" report if propagation is suspected to have failed
12. **010 $a change updates $0 in all linked bib fields** — important for test cases that verify $0 value after 010 update
13. **Number of titles = count of linked bibs** — if 0, safe to delete without affecting any bib; if > 0, deletion will unlink all those bibs
14. **"See from" references (4XX) appear in search results** — they show with a "Reference" indicator; selecting their checkbox for export counts toward the authority record export (not a separate record)
15. **Authority file prefix must be alphabetical only, max 25 chars, unique** — non-alphabetical characters or duplicate prefixes cause validation error when creating authority file
16. **Cannot delete an authority file that has records assigned to it** — mark as Inactive instead
17. **quickMARC saves are asynchronous** — toast confirms save but changes may not appear immediately in search results or linked bib records
18. **Reports (headings updates, failed updates) retrieved from Export Manager** — not downloadable directly from MARC Authority app; must go to Export Manager to download the CSV
