# FOLIO OAI-PMH — Business Logic Context

> This file is used by the `write-testrail-cases` skill to understand the business logic of the OAI-PMH settings before generating test cases.
> Sources: https://docs.folio.org/docs/settings/settings_oaipmh/settings_oaipmh/

---

## What is OAI-PMH in FOLIO

OAI-PMH (Open Archives Initiative Protocol for Metadata Harvesting) is a standard protocol that allows external systems (discovery layers, union catalogs, aggregators) to harvest bibliographic metadata from FOLIO. All configuration lives in **Settings → OAI-PMH** — there is no separate OAI-PMH app in FOLIO's navigation.

FOLIO's OAI-PMH feed exposes **Instance records** to harvesters. The feed is accessed via HTTP using standard OAI-PMH verbs.

---

## Supported Metadata Formats

FOLIO currently supports three metadata formats, discoverable via the `ListMetadataFormats` OAI-PMH verb:

| Format | Description |
|---|---|
| **marc21** | Full MARC 21 bibliographic record |
| **oai_dc** | Dublin Core (simplified metadata) |
| **marc21_withholdings** | MARC 21 record including holdings and item location data mapped to MARC fields |

---

## Settings Structure

OAI-PMH settings are organized into four tabs:

1. **General** — identity and basic service configuration
2. **Technical** — response pagination, validation, formatting
3. **Behavior** — deleted records, suppressed records, error handling, record source
4. **Logs** — harvest history and error download

---

## Settings → OAI-PMH → General

Configures the **Identify** verb response (the OAI-PMH handshake that harvesters use to learn about the repository).

| Setting | Description | Notes |
|---|---|---|
| **Enable OAI service** | Checkbox — globally enables or disables the OAI-PMH feed | If disabled, all OAI-PMH requests return an error |
| **Repository name** | Sets `repositoryName` XML element in Identify response | Human-readable name for the repository |
| **Base URL** | Sets `baseURL` XML element in Identify response; also appears in `response` element of ListRecords | Must be the correct public URL of the FOLIO OAI-PMH endpoint |
| **Time granularity** | Sets `granularity` XML element — determines date precision in selective harvesting | Options: YYYY-MM-DD (day) or YYYY-MM-DDThh:mm:ssZ (second) |
| **Administrator email(s)** | Sets `adminEmail` XML element | Can be multiple; comma-separated |

---

## Settings → OAI-PMH → Technical

| Setting | Description | Notes |
|---|---|---|
| **Max records per response** | Number of records returned per page of ListRecords/ListIdentifiers response | Uses resumptionToken for pagination when results exceed this limit |
| **Enable validation** | Whether response XML is validated against XSD schema | If enabled, malformed responses are caught; if disabled, faster but no schema check |
| **Formatted output** | Whether XML output includes line feeds and indentation (pretty-print) | Enabled = human-readable; Disabled = compact, smaller payload |

---

## Settings → OAI-PMH → Behavior

### Deleted records support

Controls how FOLIO reports deleted records to harvesters.

| Option | Behavior |
|---|---|
| **No** | No information about deleted records is included in OAI-PMH responses |
| **Persistent** | Records with MARC Leader 05 = `d`, `s`, or `x` are listed as deleted in ListRecords responses; guarantee that deleted records are kept indefinitely |
| **Transient** | Same behavior as Persistent (LDR 05 = d/s/x reported as deleted), but **no guarantee** deleted records are kept indefinitely in FOLIO |

Setting value appears in the `deletedRecord` XML element of the Identify response.

**FOLIO implementation detail:** Deleted records are detected via MARC LDR 05 values. An instance "set for deletion" (LDR 05 = 'd') will appear as deleted in the OAI-PMH feed if Persistent or Transient is selected.

### Suppressed records processing

Controls how records with "Suppress from discovery" flag are handled.

| Option | Behavior |
|---|---|
| **Transfer suppressed records with discovery flag value** | Suppressed records ARE included in the feed; a MARC subfield `$t = 1` is added to: field 999 (Instances), field 852 (Holdings), field 952 (Items) to signal suppression to the harvester |
| **Skip suppressed from discovery records** | Instances marked "Suppress from Discovery" are **excluded** from the OAI-PMH feed entirely |

### OAI-PMH errors processing

Controls HTTP status codes returned with OAI-PMH error responses.

| Option | Behavior |
|---|---|
| **Associate with HTTP status 200** | All responses (including errors) return HTTP 200; error details in XML body |
| **Associate with HTTP error statuses** | Errors return appropriate HTTP error codes (400, 404, 500, etc.) |

### Record source

Determines which FOLIO storage layer provides records for the harvest.

| Option | Behavior |
|---|---|
| **Source records storage** | Only MARC Instance records from SRS are harvested; FOLIO-native instances (no MARC) are excluded |
| **Inventory** | All Instance records harvested from Inventory; all records generated on-the-fly; no SRS dependency |
| **Source records storage and inventory** | MARC instances from SRS; FOLIO-native instances from Inventory (generated on-the-fly). Mixed mode — combines both sources |

---

## Settings → OAI-PMH → Logs

- Displays a list of **completed harvests**
- Log entries kept for **30 days**
- If a harvest contains records that could not be processed: a **Download** link appears in the rightmost column
- Clicking Download → downloads a spreadsheet with:
  - UUID of the affected instance
  - Error message explaining why it failed
- No logs for individual OAI-PMH verb requests (only completed harvest jobs)

---

## OAI-PMH Verbs (Protocol Reference)

Standard OAI-PMH verbs supported by FOLIO:

| Verb | Purpose |
|---|---|
| **Identify** | Returns repository information (name, base URL, granularity, admin email, deletedRecord policy) |
| **ListMetadataFormats** | Returns list of supported metadata formats (marc21, oai_dc, marc21_withholdings) |
| **ListSets** | Returns list of available sets (if sets are configured) |
| **ListIdentifiers** | Returns list of record identifiers (with optional date range filtering) |
| **ListRecords** | Returns full records (with optional date range filtering and set selection) |
| **GetRecord** | Returns a single record by identifier |

Selective harvesting (filtering by date range) uses the `from` and `until` parameters; precision depends on Time granularity setting.

---

## Field Mappings for marc21_withholdings

The `marc21_withholdings` format maps Inventory (Holdings and Item) data into MARC fields so that discovery systems can filter by location and call number:

| Inventory data | MARC field in feed |
|---|---|
| Holdings location | 852 |
| Item location | 952 |
| Suppression flag (Holdings) | 852 $t = 1 |
| Suppression flag (Items) | 952 $t = 1 |
| Suppression flag (Instances) | 999 $t = 1 |

Full field mapping documented on the FOLIO wiki: https://folio-org.atlassian.net/wiki/spaces/FOLIOtips/pages/5673827/Harvested+Inventory+Data

---

## Required Capability Sets (Eureka)

| Action | Capability Set |
|---|---|
| View OAI-PMH settings pages (navigation) | `Settings - Settings Oai-Pmh Enabled - View` |
| View all OAI-PMH settings (read-only, Save disabled) | `Settings - UI-Oai-Pmh Settings - View` |
| View and edit OAI-PMH settings | `Settings - UI-Oai-Pmh Settings - Edit` |
| View OAI-PMH logs | `Settings - UI-Oai-Pmh Settings Logs - View` |

**Important:** View and Edit are separate capabilities. A user with View only sees the settings but cannot save changes (Save button is disabled).

---

## Key Business Rules for Test Cases

1. **Enable OAI service checkbox is a global on/off switch** — when unchecked, all OAI-PMH requests return an error regardless of other settings
2. **View and Edit are separate capabilities** — user with View only sees settings but Save button is disabled; user with Edit can both view and save
3. **Logs capability is separate from settings capabilities** — a user with Edit cannot see Logs without the Logs capability
4. **Three metadata formats supported** — marc21, oai_dc, marc21_withholdings; discoverable via ListMetadataFormats verb
5. **Deleted records: No means harvesters see nothing about deleted items** — Persistent and Transient both expose deleted records; difference is the retention guarantee
6. **Deleted record detection uses MARC LDR 05** — only records with LDR 05 = 'd', 's', or 'x' are reported as deleted; FOLIO-native deletions without MARC may not appear
7. **Suppressed records: two opposite behaviors** — "Transfer suppressed" includes them with $t=1 flag; "Skip suppressed" excludes them entirely; affects what harvesters receive
8. **$t = 1 suppression subfield is added to 999, 852, 952 fields** — only when "Transfer suppressed records with discovery flag value" is selected
9. **Record source determines which instances appear in the feed** — "Source records storage" excludes FOLIO-native (non-MARC) instances entirely; "Inventory" includes all but generates records on-the-fly; "Source records storage and inventory" is the mixed mode
10. **Max records per response controls pagination** — harvesters use resumptionToken to fetch subsequent pages; setting affects performance and harvester behavior
11. **Enable validation adds XSD schema checking** — if enabled and response is malformed, error returned; if disabled, faster responses but no schema enforcement
12. **Formatted output affects XML readability only** — functionally equivalent; compact output is smaller payload
13. **HTTP error status setting affects harvester compatibility** — some harvesters expect HTTP 200 for all OAI-PMH responses (including errors); others expect proper HTTP error codes
14. **Logs kept for 30 days only** — historical harvests older than 30 days are deleted automatically
15. **Error download in logs contains instance UUIDs** — useful for diagnosing which specific records failed during harvest; download only appears when errors exist
16. **Time granularity affects selective harvesting precision** — day-level granularity means `from`/`until` parameters only accept YYYY-MM-DD; second-level accepts full timestamp
17. **Base URL must match the actual OAI-PMH endpoint URL** — incorrect Base URL setting causes harvesters to use the wrong URL in resumptionTokens, breaking pagination
