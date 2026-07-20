---
bc-version: [all]
domain: data-modeling
keywords: [audit, audit-trail, system-fields, systemcreatedat, systemcreatedby, systemmodifiedat, systemmodifiedby, created-by, created-at, modified-by, modified-at, systemid, systemrowversion, table-design, metadata, custom-field, reinvent]
technologies: [al]
countries: [w1]
application-area: [all]
---

# Use the platform system audit fields instead of hand-rolled created/modified fields

## Description

Every AL table in Business Central automatically carries a set of platform-managed **system fields**: `SystemId` (GUID), `SystemCreatedAt` (DateTime), `SystemCreatedBy` (GUID user security ID), `SystemModifiedAt` (DateTime), `SystemModifiedBy` (GUID), and `SystemRowVersion` (BigInteger). The platform populates and maintains `SystemCreatedAt/By` on insert and `SystemModifiedAt/By` on every modify — with no trigger code, no field declarations, and no risk of a developer forgetting to set them on one write path. They are always present, always consistent, and already indexed and exposed to the runtime.

Re-creating this behaviour with custom fields — for example declaring `"System Created By"` / `"System Created At"` and populating them in `OnInsert`/`OnModify` — reinvents platform functionality the wrong way. It costs extra fields (and object size), extra trigger code that must be kept correct across *every* insert/modify path, and it introduces subtle bugs: a `TRANSFERFIELDS`, a `MODIFYALL`, or a direct `Rec.Modify(false)` that skips the trigger will silently leave the custom audit fields stale, whereas the platform system fields are updated by the kernel regardless. Custom audit fields also store the plain `UserId` text, which is not stable across environments, while `SystemCreatedBy`/`SystemModifiedBy` store the user security GUID that joins cleanly to the User table.

## Best Practice

Rely on the built-in system fields for creation and modification audit. Do not declare your own created/modified fields and do not write trigger code to populate them.

- Read them directly on any record — they need no declaration: `Rec.SystemCreatedAt`, `Rec.SystemCreatedBy`, `Rec.SystemModifiedAt`, `Rec.SystemModifiedBy`.
- Surface them on a page or API by adding a field control bound to the system field (e.g. `field(CreatedAt; Rec.SystemCreatedAt)`), optionally with `Editable = false`.
- Resolve a system user GUID to a name via the `User` table (`"User Security ID"`) or `User Personalization` when you need a readable value.
- Only introduce a **custom** audit field when the requirement genuinely exceeds the platform's semantics — for example a domain-specific "Approved By/At", a full field-level change log (use Change Log or a dedicated history table), or a status-transition timestamp. Name and justify it as business data, not as a generic created/modified stamp.

## Anti Pattern

Declaring custom `"System Created By"` / `"System Created At"` / `"System Modified By"` / `"System Modified At"` fields (often at high IDs like 9998/9999) and populating them from `UserId` and `CurrentDateTime` in `OnInsert`/`OnModify` triggers:

```al
// Anti-pattern — reinvents SystemCreatedAt/By and SystemModifiedAt/By
field(9998; "System Created By"; Code[50]) { Editable = false; }
field(9999; "System Created At"; DateTime) { Editable = false; }

trigger OnInsert()
begin
    "System Created By" := CopyStr(UserId, 1, MaxStrLen("System Created By"));
    "System Created At" := CurrentDateTime;
end;
```

This duplicates platform behaviour, breaks on any write path that bypasses the trigger (`ModifyAll`, `Modify(false)`, `TransferFields`), stores an environment-unstable `UserId` string instead of the user security GUID, and adds object weight for nothing. Prefer the platform fields above.
