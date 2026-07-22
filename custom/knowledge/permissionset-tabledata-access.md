---
bc-version: [all]
domain: security
keywords: [permissionset, permission-set, tabledata, table-permission, rimd, missing-matching-permission-set, PTE0004, ptesecurity, indirectpermission, direct-permission, read, insert, modify, delete, execute, x, assignable, table-vs-tabledata]
technologies: [al]
countries: [w1]
application-area: [all]
---

# Grant table access with `tabledata … = RIMD`, not `table … = X`

## Description

When a PTE declares its own tables, the compiler (with `PerTenantExtensionCop`) requires every table to be reachable through a **matching permission set** — otherwise it raises *"Table 5500x '…' is missing a matching permission set."* A fast/cheap model, prompted to "add a permission set," frequently writes the table line as `table "My Table-SHB" = X` and moves on, because `= X` is the correct form for *executable* objects (codeunits, pages, reports, queries, xmlports). The build then **still fails with the same "missing matching permission set" error**, and — because the message points at the *table*, not the permission set — a panicked recovery often edits the table, adds a second permission set, or suppresses the cop, none of which fix it.

The distinction the model misses: a **`table`** permission (`X`) grants access to the table *object* (its metadata/design), while a **`tabledata`** permission grants access to the *records* — Read, Insert, Modify, Delete. The "matching permission set" the cop demands is a **`tabledata`** grant, not a `table` grant. `table … = X` does not satisfy it.

## Best Practice

For every table the extension owns, add a **`tabledata`** entry with the record-level rights the app needs — normally full `RIMD` for the app's own master/transaction tables:

```al
permissionset 55000 "Vend. Skill PIL-SHB"
{
    Assignable = true;
    Caption = 'Vendor Skills';

    Permissions =
        tabledata "Skill Master PIL-SHB" = RIMD,
        tabledata "Vendor Skill PIL-SHB" = RIMD,
        codeunit "Vendor Skill Mgmt PIL-SHB" = X;
}
```

Rules of thumb:
- **`tabledata "…" = RIMD`** — record rights (Read/Insert/Modify/Delete). This is what clears "missing a matching permission set". Narrow to `R` (read-only lookup) or `RM` where the app genuinely never inserts/deletes, but default to `RIMD` for tables the extension itself maintains.
- **`table "…" = X`** — object-level access; rarely needed on its own and does **not** substitute for `tabledata`. Do not reach for it to silence the table cop.
- **`= X` is for executable objects** — `codeunit`, `page`, `report`, `query`, `xmlport`. Enum, interface, and permissionset objects are not listed.
- Keep the permission set `Assignable = true` so it can be granted to users/other sets, and remember the object name is capped at **20 characters** (keep the `-SHB` suffix, abbreviate the descriptive part), with the file name matching the object name minus special characters.

## Anti Pattern

Granting `table … = X` to satisfy the "missing matching permission set" cop — it compiles the permission set but leaves the table error unresolved:

```al
// Anti-pattern — object-level grant does NOT clear "missing a matching permission set",
// and hands out no record rights (R/I/M/D) at runtime.
permissionset 55000 "Vend. Skill PIL-SHB"
{
    Permissions =
        table "Skill Master PIL-SHB" = X,   // wrong: still "missing matching permission set"
        table "Vendor Skill PIL-SHB" = X,   // wrong
        codeunit "Vendor Skill Mgmt PIL-SHB" = X;
}
```

Use `tabledata … = RIMD` for the records, and keep `= X` only for the codeunit:

```al
Permissions =
    tabledata "Skill Master PIL-SHB" = RIMD,
    tabledata "Vendor Skill PIL-SHB" = RIMD,
    codeunit "Vendor Skill Mgmt PIL-SHB" = X;
```

Recovery rule: when "Table … is missing a matching permission set" persists *after* you added a permission set, the fix is almost always **change `table` → `tabledata` (with `RIMD`)** — not editing the table, adding another permission set, or suppressing the cop.
