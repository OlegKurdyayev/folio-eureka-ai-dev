# FOLIO Agreements App — Business Logic Context

> This file is used by the `write-testrail-cases` skill to understand the business logic of the Agreements app before generating test cases.
> Sources: https://docs.folio.org/docs/erm/agreements/

---

## What is Agreements

The Agreements app is part of FOLIO's Electronic Resource Management (ERM) module. It manages contracts with content vendors for electronic resources — defining what content the library can access, under what terms, for what period, and linking to licensing, acquisitions, and usage data. Agreements connect to Licenses (terms of use), eHoldings (KB content), Orders (purchase order lines), and eUsage (usage statistics).

---

## Key Terms

| Term | Definition |
|---|---|
| **Agreement** | A contract record describing content access, linked to vendor, license, and acquisitions info |
| **Knowledge base (KB)** | Database of e-resource metadata (title lists, coverage dates, packages) |
| **Internal KB** | FOLIO's own knowledge base (Local KB admin) |
| **External KB** | Third-party KB like EBSCO, used via eHoldings integration |
| **Agreement line** | A single e-resource (package or title) covered by the agreement |
| **License** | Linked contract or terms of use from the Licenses app |
| **E-resource** | An electronic resource — can be a Package or a Title within a package |
| **Platform** | The delivery platform for e-resources (e.g. vendor website) |

---

## Agreement Record Structure (Accordions)

1. **Agreement information** — Name, Status, Periods, Description, Alternative names, Content type, Renewal priority, Is perpetual
2. **Agreement periods** — Date ranges when agreement is active
3. **Internal contacts** — Library staff contacts linked via Users app
4. **Agreement lines** — E-resources covered by this agreement (packages, titles)
5. **License information** — Linked licenses from Licenses app (controlling and supplemental)
6. **Organizations** — Vendors and other organizations linked to the agreement
7. **Supplementary properties** — Custom local metadata fields (configured in Settings)
8. **Supplementary documents** — Attached files and links (signed contract PDFs, etc.)
9. **Usage data** — eUsage accordion (requires eUsage permission); shows charts for titles in agreement
10. **Related agreements** — Links to predecessor, successor, or related agreements
11. **Notes** — Internal notes
12. **Tags** — Organizational tags

---

## Primary Fields

### Required fields
- **Name** — agreement title
- **Status** — pick list; configurable in Settings → Agreements → Pick list values

### Status values (configurable, common defaults)
- **Active** — currently in force
- **Closed** — terminated or expired; triggers **Reason for closure** field
- **Draft** — being prepared
- **In negotiation** — under discussion
- **Requested** — requested but not yet active

### Reason for closure
- Only visible/required when Status = Closed
- Values configurable in Settings → Agreements → Pick list values
- Examples: Cancelled, Ceased, Completed, Rejected, Superseded

### Other primary fields
| Field | Notes |
|---|---|
| **Description** | Free text summary |
| **Alternative names** | Other names the agreement is known by; adding one requires filling it in or deleting it |
| **Renewal priority** | Configurable pick list (e.g. For review, Definitely cancel, Definitely renew) |
| **Is perpetual** | Yes/No — whether agreement grants perpetual access after it ends |
| **Content type** | Configurable pick list (e.g. E-journals, E-books, Databases); multiple allowed; adding one requires selecting a value or deleting it |

---

## Agreement Periods

Every agreement must have at least one period. Periods define when the agreement is active.

### Period fields
- **Start date** (required per period)
- **End date** (optional; only one period in a set can have no end date — "open-ended")
- **Cancellation deadline** — date by which library must cancel to avoid renewal
- **Period note** — free text

### Period rules
- At least one period required to save an agreement
- Multiple periods allowed — their date ranges must **not overlap**
- The **earliest** period start date = agreement start date
- The **latest** period end date = agreement end date (or no end date if open-ended)
- If only one period: no period navigation UI shown
- If multiple periods: Previous / Current / Next navigation buttons appear
- "Open-ended" period = has start date but no end date; agreement assumed indefinite

---

## Agreement Lines

Agreement lines represent individual e-resources (packages or titles) covered by the agreement.

### Sources for agreement lines
1. **Internal KB** — search FOLIO's local knowledge base (via Local KB admin)
2. **eHoldings** — search EBSCO or other external KB via eHoldings integration
3. **Unlinked** — add manually without linking to a KB resource (for tracking purposes)

### Agreement line fields
- **Type** — set automatically based on source (Package, Title, Unlinked)
- **Active from / Active to** — date range this specific line is active (can differ from agreement period)
- **Suppress from discovery** — hide this line from discovery layer
- **Note** — internal note on the line
- **Linked POL** — link to a Purchase Order Line in Orders app (multiple POLs can be linked to one agreement line)
- **Documents** — attached documents specific to this line

### Agreement line types
| Type | Description |
|---|---|
| **Package** | Entire package from KB (e.g. "Springer E-books 2024") |
| **Title** | Individual title from a package |
| **Unlinked** | Manual entry not connected to any KB |

### Linking POLs to agreement lines
- An agreement line can have one or more POLs linked
- Linking connects acquisitions spend data to ERM content tracking
- Can be done at line creation or later via edit
- POL can be replaced or removed from the agreement line

---

## License Information

Agreements can be linked to license records from the Licenses app.

### License link types
- **Controlling** — the primary license governing this agreement
- **Historical** — previous licenses
- **Future** — planned future licenses
- **Supplemental** — additional licenses with specific terms

### License note
- Each linked license can have a note explaining its relevance

---

## Organizations

Organizations linked to an agreement record vendors and other relevant institutions.

### Organization roles (configurable in Settings)
- **Vendor** — the content provider
- **Licensor** — the party granting the license
- **Consortium** — a consortium facilitating the agreement
- Others as configured

### Linking an organization
- Uses Organization look-up (same organizations as in Orders/Finance)
- Role must be assigned when adding

---

## Supplementary Properties

Custom metadata fields configured per institution in **Settings → Agreements → Supplementary properties**. Used to track local information not covered by standard fields (e.g. subject area, library selector, cost).

Each property has: name, type (text, number, date, pick list), primary/secondary designation, and optionally a note.

---

## Supplementary Documents

File attachments and links associated with the agreement.

### Document fields
- **Name** (required)
- **Note**
- **Category** — configurable pick list (e.g. Consortium agreement, Draft, Final, License, Signed)
- **File** — uploaded document
- **URL** — external link

Download requires the `Agreements: File download` capability.

---

## Related Agreements

Links this agreement to other agreement records to capture relationships.

### Relationship types (configurable)
- **Has backfile in**
- **Has package in**
- **Superseded by**
- **Supersedes**
- Others as configured in Settings

---

## Search and Filters

### Key search filters
- **Status** — Active, Closed, Draft, etc.
- **Reason for closure**
- **Renewal priority**
- **Is perpetual** — Yes/No
- **Start date / End date / Cancellation deadline**
- **Organizations** + **Organizations role**
- **Internal contacts** + **Internal contacts role**
- **Content type**
- **Tags**
- **Supplementary properties**
- **Agreement content** — search by KB package/title
- **Documents**

### Agreement line filters
- **Agreement** (parent agreement)
- **Agreement line type** — Package, Title, Unlinked
- **Active from / Active to**
- **PO line**
- **Tags**
- **Documents**

---

## Export

### Export an agreement
- Actions → Export — downloads agreement data as JSON

### Export e-resources
- Actions → Export e-resources — downloads coverage data for all e-resources in the agreement (useful for KBART files, etc.)

---

## E-Resources (Internal KB)

Packages and titles in the internal knowledge base, managed via **Local KB admin** app.

### Package
- A collection of titles from a content provider
- Has: name, source, coverage, platform, content type
- Can be assigned to an agreement line directly

### Title
- Individual e-resource within a package
- Has: title, identifiers (ISBN, ISSN, etc.), coverage dates, platform

### Platforms
- Delivery platforms for e-resources
- Can be edited to add URL customizations (library-specific proxy prefixes, etc.)

---

## Integration with Other Apps

| App | Integration |
|---|---|
| **Licenses** | Link licenses to agreements; terms of use for content |
| **Orders** | Link POLs to agreement lines; connects spend to content |
| **eHoldings** | Source KB for agreement lines (external KB) |
| **eUsage** | Usage statistics displayed within agreement record |
| **Local KB admin** | Source KB for agreement lines (internal KB) |
| **Users** | Internal contacts must have Users app records |

---

## Required Capability Sets (Eureka)

| Action | Capability Set |
|---|---|
| View/search agreements | `Data - UI-Agreements Agreements - View` |
| Edit agreements | `Data - UI-Agreements Agreements - Edit` |
| Delete agreements | `Data - UI-Agreements Agreements - Delete` |
| Download documents | `Procedural - UI-Agreements Agreements File - Execute` |
| View/search e-resources (internal KB) | `Data - UI-Agreements Resources - View` |
| Edit e-resources | `Data - UI-Agreements Resources - Edit` |
| View platforms | `Data - UI-Agreements Platforms - View` |
| Edit platforms | `Data - UI-Agreements Platforms - Edit` |
| View eUsage charts in agreement | `Data - Plugin-Eusage-Reports Charts - View` |

---

## Key Business Rules for Test Cases

1. **Name and Status are the only required fields** — agreement can be saved with just these two + at least one period
2. **At least one period required** — agreement cannot be saved without an agreement period; each period requires a Start date
3. **Period dates must not overlap** — saving overlapping periods returns a validation error
4. **Only one open-ended period allowed** — multiple periods without end dates is not permitted
5. **Reason for closure only appears when Status = Closed** — field hidden for other statuses
6. **Content type: clicking Add requires a value** — clicking "Add content type" without selecting a value blocks save; must either fill or delete
7. **Alternative names: same rule** — clicking "Add alternative name" without filling it blocks save
8. **Adding internal contact requires a User record** — cannot add a free-text name; must link via Users app lookup
9. **Agreement lines can be unlinked** — not every line needs to link to a KB resource; manual tracking is allowed
10. **Multiple POLs can link to one agreement line** — one content item can be paid across multiple purchase orders
11. **POL linking is optional** — agreement lines don't require a linked POL
12. **License links are typed** — controlling vs. supplemental vs. historical vs. future; each type has different semantic meaning
13. **Delete agreement does not include edit** — Delete capability is separate from Edit; a user can have Delete without Edit
14. **File download is separate from upload** — requires its own `File download` capability even if user has Edit
15. **Export e-resources generates KBART-compatible coverage data** — used for sharing holdings with discovery systems and aggregators
16. **eUsage accordion requires separate eUsage permission** — not included in standard Agreements Edit capability
17. **Platform URL customization is per-institution** — proxy prefixes and URL templates configured per platform record; affects how links resolve in discovery
18. **Agreement status values are configurable** — institutions can add/remove status options in Settings → Agreements → Pick list values; agent should not assume a fixed list
19. **Internal KB vs external KB behavior differs** — instructions for adding agreement lines differ depending on whether the library uses FOLIO's internal KB or an external KB like EBSCO via eHoldings
20. **Related agreements require relationship type** — cannot link two agreements without selecting the relationship type; relationship types are configurable
