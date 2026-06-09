# FOLIO eHoldings App — Business Logic Context

> This file is used by the `write-testrail-cases` skill to understand the business logic of the eHoldings app before generating test cases.
> Sources: https://docs.folio.org/docs/erm/eholdings/

---

## What is eHoldings

The eHoldings app manages the library's electronic resource holdings via an external knowledge base (KB). It connects to EBSCO's HoldingsIQ API — currently the **only supported external KB**. Changes made in FOLIO eHoldings or EBSCO Holdings Management are **immediately reflected** in both interfaces (real-time sync via API).

eHoldings allows libraries to:
- Browse and search providers, packages, and titles from the EBSCO KB
- Select/deselect packages and titles for the library's holdings
- Edit coverage settings, proxy settings, and custom labels
- Create custom packages and titles
- Link e-resources to Agreements in the Agreements app
- View usage & analysis (cost-per-use) data

---

## Architecture: External KB

eHoldings does **not** store KB data in FOLIO — all data lives in EBSCO's knowledge base. FOLIO accesses it via the HoldingsIQ API. Requires:
- EBSCO HoldingsIQ API subscription
- API key configured in **Settings → eHoldings**
- EBSCO customer ID configured in Settings

Changes are real-time: no batch sync needed.

---

## Record Hierarchy

```
Provider (content vendor, e.g. Elsevier, Springer)
  └── Package (collection of titles, e.g. "Elsevier Freedom Collection")
        └── Title (individual e-resource, e.g. a journal or e-book)
              └── Resource (title in context of a specific package = title + package combination)
```

---

## Providers

### Provider record fields
- **Name** — provider name (from EBSCO KB)
- **Packages selected** — number of packages the library has selected
- **Total packages** — total packages available from this provider
- **Proxy settings** — proxy prefix for accessing this provider's content
- **Tags** — organizational tags
- **Notes** — internal notes

### What you can edit on a Provider
- Proxy settings
- Tags
- Notes

Providers themselves cannot be created or deleted — they come from EBSCO KB.

---

## Packages

### Package types
- **KB packages** — come from EBSCO KB; can be selected/deselected
- **Custom packages** — created locally in FOLIO eHoldings; fully editable; can be deleted

### Package record fields
- **Name** — package name
- **Provider** — which provider this package belongs to
- **Content type** — type of content (see Content Type list below)
- **Selected** — whether this library has selected this package
- **Hidden** — whether this package is hidden from discovery
- **Titles selected** — number of titles selected within this package
- **Total titles** — total titles in the package
- **Proxy** — proxy setting for this package (overrides provider proxy)
- **Coverage settings** — custom date ranges for coverage
- **Tags** — organizational tags
- **Notes** — internal notes
- **Agreements** — linked agreements from Agreements app

### Editing a package
- **Holding status** (Selected/Not Selected) — requires `Can select/unselect` capability
- **Package settings** — visibility, proxy, allow/disallow title customization
- **Coverage settings** — custom start/end dates for this package's coverage

### Custom packages
- Created by the library locally (not from EBSCO KB)
- Can contain custom titles
- Can be deleted (KB packages cannot be deleted)
- Require: Name, Content type

### Package Content Type values
- Aggregated Full Text
- Abstract and Index
- E-Book
- E-Journal
- Mixed Content
- Online Reference
- Print
- Streaming Media
- Unknown

### Package export
- Actions → Export — sends export job to Export Manager app
- Downloads package details as a file via Export Manager

---

## Titles

### Title types (Publication Type)
Audiobook, Book, Book Series, Database, Journal, Newsletter, Newspaper, Proceedings, Report, Streaming Audio, Streaming Video, Thesis & Dissertation, Website, Unspecified

### Title record fields
- **Name** — title name
- **Publisher** — publisher of the title
- **Publication type** — see list above
- **Identifiers** — ISSN, ISBN, etc.
- **Subjects** — subject headings
- **Peer reviewed** — Yes/No
- **Contributors** — authors/editors
- **Description** — description of the title
- **Packages** — list of packages this title appears in
- **Coverage** — coverage date ranges
- **Custom labels** — local custom metadata fields (up to 4, configured in Settings → eHoldings)
- **Tags** — organizational tags
- **Notes** — internal notes
- **Agreements** — linked agreements

### Adding a title to holdings
- User selects a title and clicks "Add to holdings" (or selects a package)
- Requires `Can select/unselect` capability

### Removing a title from holdings
- Available from the title's resource view within a package
- Requires `Can select/unselect` capability

### Custom titles
- Created locally by the library (not from EBSCO KB)
- Can be added to custom packages
- Can be deleted (KB titles cannot be deleted from FOLIO)
- Required fields: Title name, Publication type, Package (must link to a package when creating)

### Title search options
- Keyword (default)
- ISSN/ISBN
- Publisher
- Subject

---

## Resources (Title + Package combination)

A **Resource** is a title viewed in the context of a specific package. The same title can appear in multiple packages — each is a separate Resource record.

### Resource-level settings (editable per title-in-package)
- **Holdings status** — selected or not selected
- **Visibility** — show/hide from discovery
- **Proxy** — override proxy for this specific resource
- **Coverage** — custom coverage dates for this specific resource
- **Custom labels** — local metadata

---

## Selection Status

| Status | Meaning |
|---|---|
| **Selected** | Library has added this package/title to their holdings |
| **Not Selected** | Package/title exists in KB but not in library's holdings |

Filtering by selection status is available on both packages and titles search.

---

## Proxy Settings

Proxy settings can be configured at three levels (each overrides the one above):
1. **Provider level** — applies to all packages from this provider
2. **Package level** — overrides provider proxy for this package
3. **Resource level** — overrides package proxy for this specific title in this package

Proxy servers are configured in **Settings → eHoldings → Root proxy**.

---

## Coverage Settings

Coverage defines the date range for which content is available. Can be set at:
- **Package level** — applies to all titles in the package
- **Resource level** — overrides package coverage for a specific title

Coverage fields:
- **Start date** (Volume, Issue, Year can also be specified)
- **End date** (or left blank for "present")
- Multiple coverage ranges can be added

---

## Custom Labels

Libraries can define up to **4 custom label fields** per title record in **Settings → eHoldings → Custom labels**. Used for local metadata not available in the standard KB fields. Labels appear on title/resource records and are editable per resource.

---

## Agreements Integration

From any title or package record, user can:
- **Link an existing agreement** — search Agreements app and link
- **Create a new agreement** — opens New agreement form pre-populated with the resource
- **Remove an agreement** — unlink from the resource

Linked agreements appear in both the eHoldings record and the Agreements record.

---

## Usage & Analysis (Cost-Per-Use)

Requires separate permission: `eHoldings: Can view Usage & analysis data`.

Also requires **Usage Consolidation** to be configured in **Settings → eHoldings → Usage consolidation**:
- SUSHI credentials (for usage data)
- Currency settings
- Fiscal year settings

Displays cost-per-use metrics for packages and titles based on subscription cost vs. usage data from eUsage.

---

## Search and Filters

### Search tabs
- **Providers**
- **Packages**
- **Titles**

### Filters — Packages
- **Selection status**: All / Selected / Not Selected
- **Content type**: Aggregated Full Text, E-Book, E-Journal, Mixed Content, etc.
- **Tags**
- **Sort**: Relevance (default), Package name

### Filters — Titles
- **Selection status**: All / Selected / Not Selected
- **Publication type**: Book, Journal, Database, etc.
- **Packages**: filter titles by package
- **Tags**
- **Sort**: Relevance (default), Title name

### Filters — Providers
- **Tags**
- **Sort**: Relevance (default), Provider name

---

## Settings → eHoldings

- **KB credentials** — EBSCO API key and customer ID (required for any eHoldings functionality)
- **Root proxy** — default proxy server for all e-resources
- **Custom labels** — define up to 4 custom label fields for title records
- **Usage consolidation** — SUSHI credentials for cost-per-use analysis

---

## Required Capability Sets (Eureka)

| Action | Capability Set |
|---|---|
| View providers, packages, titles | *(included with app access)* |
| Edit providers, packages, titles | `Data - UI-Eholdings Records - Edit` |
| Select/unselect packages and titles | `Procedural - UI-Eholdings Package-Title Select-Unselect - Execute` |
| Create/delete custom packages and titles | `Data - UI-Eholdings Titles-Packages Create-Delete - Manage` |
| View usage & analysis (cost-per-use) | `Data - UI-Eholdings Costperuse - View` |
| Manage Usage Consolidation settings | `Settings - UI-Eholdings Settings Usage-Consolidation Create-Edit` |
| View Usage Consolidation settings | `Settings - UI-Eholdings Settings Usage-Consolidation` |
| Assign/unassign user from KB | `Settings - UI-Eholdings Settings AssignedUser - Edit` |
| Export packages/titles | `Data - UI-Export-Manager Export-Manager - Manage` |

---

## Key Business Rules for Test Cases

1. **EBSCO is the only supported external KB** — eHoldings requires EBSCO HoldingsIQ API setup; without credentials in Settings, the app shows no data
2. **Changes are real-time** — selecting/deselecting a package in FOLIO immediately reflects in EBSCO Holdings Management and vice versa; no sync delay
3. **Cannot delete KB packages or titles** — only custom packages and custom titles created in FOLIO can be deleted; KB content is read-only for deletion
4. **Select/unselect requires a separate capability** — Edit capability alone doesn't allow adding/removing holdings; needs `select-unselect` capability
5. **Custom packages require Name + Content type** — these are the only required fields when creating a custom package
6. **Custom titles must be linked to a package** — a standalone title cannot exist without being in a package; required when creating
7. **Proxy hierarchy: Resource overrides Package overrides Provider** — the most specific proxy setting wins
8. **Coverage can be set at package and resource levels** — resource-level coverage overrides package-level coverage for that specific title
9. **Custom labels are institution-specific** — up to 4 labels configured in Settings; labels appear on all title records but values are per resource
10. **Usage & analysis requires Usage Consolidation configured** — having the permission alone is not enough; SUSHI credentials must be set up in Settings
11. **Agreement linking is bidirectional** — linking from eHoldings appears in Agreements app and vice versa
12. **Tags can be removed but not deleted** — removing a tag from a record only unlinks it; the tag remains in the system as a selectable option
13. **Search defaults to Relevance sort** — other sort options (alphabetical by Provider/Package/Title) must be explicitly selected
14. **Selection status filter default is "All"** — showing both selected and unselected by default; must filter to see only library holdings
15. **Multiple packages per title** — the same title (e.g. a journal) can appear in multiple packages from different providers; each is a separate Resource record that can be independently selected/configured
16. **Provider records are not editable beyond proxy/tags/notes** — provider names, package lists, and titles all come from EBSCO KB and cannot be changed in FOLIO
17. **Export goes through Export Manager** — package/title exports are processed asynchronously; files downloaded from the Export Manager app, not directly from eHoldings
