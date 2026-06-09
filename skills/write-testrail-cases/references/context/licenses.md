# FOLIO Licenses App — Business Logic Context

> This file is used by the `write-testrail-cases` skill to understand the business logic of the Licenses app before generating test cases.
> Sources: https://docs.folio.org/docs/erm/licenses/

---

## What is Licenses

The Licenses app manages the library's license records — the legal contracts and terms of use governing what the library can and cannot do with licensed content. Licenses link to Agreements (which define *what* content is accessible) and to Organizations (licensors, consortia).

---

## Key Terms

| Term | Definition |
|---|---|
| **License** | The contract or Terms of Use governing access to content |
| **Agreement** | Defines what content is accessible; license explains the permitted uses |
| **Term** | A specific condition of the license (e.g. ILL allowed, number of simultaneous users) |
| **Amendment** | A modification to an existing license; tracked as a sub-record of the license |
| **Core document** | The primary document for the license (e.g. signed contract PDF) |
| **Supplementary document** | Any additional document supporting the license record |

---

## License Record Structure (Accordions)

1. **License information** — Name, Type, Status, Start date, End date, Open ended, Description, Alternative names
2. **Internal contacts** — Library staff linked via Users app
3. **Organizations** — Licensors, consortia, etc. linked via Organizations app
4. **Core documents** — Primary license documents (file upload, physical location, or URL)
5. **Terms** — License terms (permitted uses, restrictions, etc.)
6. **Supplementary documents** — Additional supporting documents
7. **Amendments** — Modifications to the license (sub-records)
8. **Notes** — Internal notes
9. **Tags** — Organizational tags

---

## Primary Fields

### Required fields
- **Name** — license record title (required)
- **Type** — pick list (e.g. Local, Consortial); configurable in Settings → Licenses → Pick list values
- **Status** — pick list (e.g. Active, Expired); configurable in Settings → Licenses → Pick list values

### Key optional fields
| Field | Notes |
|---|---|
| **Start date** | When the license begins |
| **End date** | When the license terminates; mutually exclusive with Open ended |
| **Open ended** | Checkbox; license has no defined end date; cannot be set alongside an End date |
| **Description** | Free text summary |
| **Alternative names** | Other names/abbreviations; all are searchable; clicking Add requires a value or delete |

### Status values (configurable, common defaults)
- **Active** — currently in use
- **Expired** — no longer in use
- Others configurable in Settings

### Type values (configurable, common defaults)
- **Local** — specific to this library
- **Consortial** — entered through a library consortium
- Others configurable in Settings

---

## Internal Contacts

Library staff associated with the license (e.g. ERM librarian, authorized signatory).

- Must have a **Users app record** — cannot be added as free text
- Multiple contacts per license allowed
- Each contact must have a **Role** assigned (configurable in Settings → Licenses → Pick list values)
- Contact can be **replaced** (Replace user button) without removing and re-adding

---

## Organizations

Institutions associated with the license (typically the licensor or consortium).

- Must have an **Organizations app record** — cannot be added as free text
- Multiple organizations per license allowed
- One organization can be set as **primary** via "Set as primary organization" checkbox
- Each organization must have a **Role** assigned (configurable in Settings)
- Note field available per organization
- To view organization records and internal contacts, user also needs **Organizations: View** and **Users: Can view user profile** permissions

---

## Core Documents

The central document(s) for the license (e.g. the signed contract).

### Adding a core document requires at least one of:
- File upload (drag & drop or choose file)
- Physical location (text field)
- URL

### Name is required when adding a core document.

A **copy icon** appears next to core documents in the record view — clicking it copies the Reference value to clipboard.

### File download
Requires the `Licenses: File download` capability — separate from Edit capability.

---

## Terms

Terms define the specific conditions of the license — what the library can and cannot do with the content.

### Term types (configured in Settings → Licenses → Terms)
- **Primary terms** — appear on every license record by default; cannot be removed; Value can be left blank or "Not set"
- **Optional terms** — not shown by default; must be added individually per license; can be removed

### Term value types (configured per term in Settings)
- Text input
- Integer (with up/down arrows)
- Pick list (dropdown selection)

### Term fields per entry
- **Value** — the answer/definition for this term condition
- **Internal note** — visible only to FOLIO staff
- **Visibility** — Internal (staff only) or External (public-facing, e.g. catalog)
- **Public note** — visible externally

### Common term examples (institution-specific)
- Interlibrary Loan (ILL) — Permitted / Not permitted / Permitted with restrictions
- Course reserves — Yes / No
- Authorized users — definition of who can access content
- Simultaneous users — maximum number
- Perpetual access — Yes / No

---

## Supplementary Documents

Additional documents beyond the core license document.

### Adding a supplementary document requires at least one of:
- File upload
- Physical location
- URL

### Required fields: Name + Category
- **Category** — pick list configurable in Settings → Licenses → Pick lists (e.g. Correspondence, Draft, Counter-signature)
- Note field available

Copy icon appears for reference copying, same as core documents.

---

## Amendments

Amendments track modifications to an existing license. They are sub-records of the license.

### Amendment record fields
- **Name** (required)
- **Status** — pick list (same values as license status)
- **Start date / End date / Open ended**
- **Description**
- **Alternative names**
- **Internal contacts**
- **Organizations**
- **Core documents**
- **Terms** (can override or supplement parent license terms)
- **Supplementary documents**

### Amendment lifecycle
- Can be **created** from the license record (Actions → New amendment)
- Can be **duplicated** — creates a copy of the amendment
- Can be **edited** and **deleted** independently of the parent license
- Amendments appear in the Amendments accordion on the license record
- Can be searched independently via the Amendments search toggle

### When terms on amendments vs parent license conflict
- Amendment terms **supplement** the parent license — they do not automatically override
- Staff must manually review and reconcile conflicting terms

---

## Search and Filters

### Search targets
- **Licenses** — default search tab
- **Amendments** — toggle to search amendments independently

### License search fields
Keyword search across: Name, Alternative names, Description

### License filters
- **Status** — Active, Expired, etc.
- **Type** — Local, Consortial, etc.
- **Organizations** — filter by linked organization
- **Organization role** — filter by the role of the linked organization
- **Tags**
- **Start date / End date**
- **Terms** — filter by term name and value
- **Supplementary documents** — filter by document name
- **Core documents** — filter by document name
- **License content** — search within license content fields

---

## Linking Licenses to Agreements

From the **Agreements app**, a license is linked to an agreement with a relationship type:
- **Controlling** — the primary governing license
- **Historical** — past licenses
- **Future** — planned future licenses
- **Supplemental** — additional supporting licenses

From the **Licenses app**, linked agreements are visible in the license record but the link itself is managed from the Agreements side.

---

## Duplicating a License

Actions → Duplicate — creates a copy of the license record including all fields and terms. Useful for renewing licenses with similar terms under a new period.

---

## Exporting a License

Actions → Export — downloads the license data as a JSON file.

---

## Settings → Licenses

Key configurations:
- **Pick lists** — Status values, Type values, Organization role values, Internal contact role values, Document category values
- **Pick list values** — individual values within each pick list
- **Terms** — define available license terms: name, type (text/integer/pick list), primary vs optional, default visibility

Managing pick lists requires `Settings (Licenses): Manage pick lists and values` capability.
Managing terms requires `Settings (Licenses): Manage license terms` capability.

---

## Required Capability Sets (Eureka)

| Action | Capability Set |
|---|---|
| View/search licenses | `Data - UI-Licenses Licenses - View` |
| Edit licenses | `Data - UI-Licenses Licenses - Edit` |
| Delete licenses | `Data - UI-Licenses Licenses - Delete` |
| Download documents | `Procedural - UI-Licenses Licenses File - Execute` |
| Manage pick lists and values | `Data - UI-Licenses Picklists - Manage` |
| Manage license terms | `Data - UI-Licenses Terms - Manage` |
| View linked organizations | `Data - UI-Organizations - View` |
| View user profiles (for internal contacts) | `Data - UI-Users - View` |

---

## Key Business Rules for Test Cases

1. **Name is the only truly required field** — Type and Status also appear required in UX but have defaults; Name without a value blocks save
2. **Open ended and End date are mutually exclusive** — selecting Open ended disables/clears End date; entering End date clears Open ended checkbox
3. **Alternative names: click Add requires value or delete** — same pattern as Agreements; blank entry blocks save
4. **Internal contacts must be Users app records** — cannot type a name; must use Link user lookup
5. **Organizations must be Organizations app records** — cannot type a name; must use Link organization lookup
6. **Viewing linked organizations requires Organizations: View permission** — without it, the Organizations section may be hidden or show restricted data
7. **Viewing internal contacts requires Users: Can view user profile** — same pattern; without it, the internal contacts section is inaccessible
8. **Core document requires Name + at least one of: file / physical location / URL** — all three fields optional but at least one must be filled
9. **Supplementary document requires Name + Category + at least one of: file / physical location / URL** — Category is required (unlike core documents)
10. **File download is separate from Edit capability** — having Edit alone does not allow downloading attached files
11. **Primary terms cannot be removed from license records** — they always appear; Value can be blank or "Not set" but the term row stays
12. **Optional terms must be explicitly added** — not shown by default; each optional term must be individually added per license
13. **Term visibility controls internal vs external display** — Internal = staff only; External = public-facing (discovery, catalog); each term has its own visibility setting
14. **Amendments are sub-records of licenses** — deleting a license does not automatically cascade-delete amendments; amendments can be searched independently
15. **Amendment terms supplement, not override, parent license terms** — no automatic conflict resolution; staff must manually reconcile
16. **Delete license capability is separate from Edit** — same pattern as Agreements; a user can delete without being able to edit
17. **Status and Type values are configurable** — institutions define their own pick list values; agent should not assume fixed set of statuses/types
18. **Duplicate creates full copy** — all terms, documents, contacts, organizations are copied; useful for renewals
19. **Search covers alternative names** — alternative names are indexed and searchable; important for preconditions where you search by non-primary name
20. **Amendments have independent search toggle** — switching to Amendments search in the filter pane searches only amendment records, not parent licenses
