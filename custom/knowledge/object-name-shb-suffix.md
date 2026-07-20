---
bc-version: [all]
domain: style
keywords: [object-name, suffix, -SHB, shb, naming, custom-object, kuerzel, kürzel, product-code, reserved-suffix, appsource, certified, caption, label, tooltip, user-facing]
technologies: [al]
countries: [w1]
application-area: [all]
---

# Suffix every SHB-authored object name with an "-SHB" suffix

## Description

At SHB Business Solutions, every AL object that SHB authors — table and page extensions, codeunits, enums, interfaces, permission sets, and new tables/pages — carries a suffix that ends in `-SHB` at the end of its object name. The suffix makes SHB's own objects instantly distinguishable from Microsoft base objects and from third-party apps in dependency lists, event subscriber pickers, and error stacks, and it keeps the suffix consistent across every customer PTE so a reviewer moving between projects never has to relearn the convention. Objects without the suffix blur the boundary between SHB code and everyone else's, which slows review and hand-over and makes ownership ambiguous when a customer or an external developer later reads the extension.

The suffix is not always a bare `-SHB`. In most solutions it is a compound of a short **Kürzel** (product or customer code) followed by `-SHB`, for example `CMG-SHB` (Contract Management). A small set of **certified / AppSource** solutions is the deliberate exception: they carry a reserved suffix *without* `-SHB` (for example `PMG`, `PPM`, `ARQ`, `CST`, `PPP`, `SDS`).

The authoritative source for these codes is the internal workbook `Nummerkreise_SHB_intern.xlsx` (sheets *IDs* and *SHB Interne Kürzel*). The tables below mirror the product and certified codes from that workbook; when the workbook changes, update this file. Customer-specific codes are intentionally not listed here — they are resolved separately (a future API will map a customer to its suffix).

## Best Practice

End the object name with `-SHB`, preferably as `<Kürzel>-SHB`, keeping the descriptive base within the 30-character budget so the suffix fits — for example `pageextension 55000 "Customer List Ext-SHB"`, `codeunit 74750 "Contract Mgt.-SHB"`, `enum 74850 "Doc. Text Status DXT-SHB"`. Plan the descriptive part around the suffix from the start rather than discovering the length limit at publish time. Apply the suffix uniformly to every object SHB creates.

- **General / ad-hoc customer PTE work:** end the name with `-SHB`, or `<CustomerKürzel>-SHB` when the work belongs to a specific customer solution. The customer's Kürzel is resolved separately, not from this file.
- **SHB product apps:** use the product's registered `<ProductKürzel>-SHB` (see *Product / app codes*), e.g. `PWS-SHB`, `RFS-SHB`, `ND2-SHB`.
- **Certified / AppSource solutions:** use the reserved suffix *without* `-SHB` (see *Reserved certified suffixes*), e.g. objects in the certified project-management solution use `PMG`, not `PMG-SHB`.

### Captions carry the name, not the suffix

The suffix belongs to the **technical object name only** — the developer-facing identifier used in dependency lists, event subscriber pickers, and error stacks. A **caption** is user-facing text shown in the client, so it MUST NOT repeat the suffix. Strip the `-SHB` / `<Kürzel>-SHB` / reserved code from every user-visible string: object `Caption`, field `Caption`, action `Caption`, `ToolTip`, `Label`, and any message text. A user sees *"Vendor Skill"*, never *"Vendor Skill PIL-SHB"*.

- `table 55003 "Vendor Skill PIL-SHB"` → `Caption = 'Vendor Skill'`
- `page 55004 "Skill Master List PIL-SHB"` → `Caption = 'Skill Master List'`
- `field(...; "Proficiency Level"; ...) { Caption = 'Proficiency Level'; }` — field captions were never suffixed and stay clean.

The same applies to added actions and controls on extensions: the object name gets the suffix; the caption the user reads does not.

### Reserved certified suffixes (no `-SHB`)

These suffixes are registered for certified/AppSource solutions and are used **without** the `-SHB` ending. Do not append `-SHB` to them, and do not reuse them for non-certified work.

| Suffix | Solution |
| --- | --- |
| `ARQ` | Abwesenheitsanträge/-registrierung (absence requests) |
| `CST` | Provisionsabrechnung (commission settlement) |
| `PMG` | Projektmanagement Lösung (project management) |
| `PPM` | SHB4Prepayments |
| `PPP` | SHB4ProjectPrepayments |
| `SDS` | SHB Standard Belegset (standard document set) |
| `SHB` | SHB company suffix (general reserved code) |

### Product / app codes (`<code>-SHB`)

Suffixes for SHB's own (non-certified) products. Objects in these apps end with the listed `-SHB` code.

| Suffix | Product |
| --- | --- |
| `AQI-SHB` | API Queue Interface |
| `CFB-SHB` | Comment Factboxes |
| `CMG-SHB` | Contract Management |
| `COP-SHB` | Copilot Apps |
| `CRM-COP-SHB` | CRM Copilot |
| `CUS-SHB` | Continuous Update Helper |
| `DXT-SHB` | Belegtexte (document texts) |
| `EDT-SHB` | Text Editor |
| `GLR-SHB` | G/L Account Rename |
| `L-PMG-DXT-SHB` | L-PMG Belegtexte Integration |
| `ND2-SHB` | Name 2 & Description 2 |
| `PBI-SHB` | Power BI Connector |
| `PIL-SHB` | Pilot Applications |
| `PMG-DXT-SHB` | PMG Belegtexte Integration |
| `PMG-PBI-SHB` | Power BI Connector PMG |
| `PPM-PBI-SHB` | Power BI Connector PPM |
| `PWS-SHB` | Password Safe |
| `RFS-SHB` | Remote File Storage |
| `SAV-SHB` | Shipment Address VAT |
| `TCS-SHB` | CUS Test App |

### Customer codes

Customer-specific suffixes are **not** stored in this file. A customer solution still ends in `<CustomerKürzel>-SHB`, but the mapping from customer to Kürzel is resolved elsewhere (a future API). Legacy C/SIDE-era customers predate the `-SHB` convention and may use a bare code without the suffix on existing objects; keep those as-is, and use `<code>-SHB` for any new AL work.

## Anti Pattern

Object names with no suffix (`pageextension 55001 "Vendor List Ext"`), an inconsistent or ad-hoc marker (`"Vendor List Ext SHB"`, `"SHB_VendorListExt"`, a leading `SHB` prefix), a name padded to 30 characters that leaves no room for the suffix and forces a rename during hand-over, appending `-SHB` to a reserved certified suffix (`"... PMG-SHB"` for the certified project-management solution instead of `PMG`), or inventing a new Kürzel when a registered product code already exists in the workbook. **Leaking the suffix into user-facing text** — `Caption = 'Vendor Skill PIL-SHB'`, a `ToolTip` or `Label` that ends in `-SHB` — is equally wrong: the suffix is for the technical name, never for what the user reads.
