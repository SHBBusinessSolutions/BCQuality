---
bc-version: [all]
domain: data-modeling
keywords: [insert, modify, delete, insert-true, modify-true, delete-true, runtrigger, run-trigger, oninsert, onmodify, ondelete, onvalidate, validate, init, direct-assignment, table-trigger, referential-integrity, junction-table, m-n, many-to-many, orphan, orphaned-records, cascade, cleanup, tablerelation, validatetablerelation, stale-buffer, record-buffer]
technologies: [al]
countries: [w1]
application-area: [all]
---

# Write records the safe way: `Init` + `Validate` + `Insert/Modify/Delete(true)`, and clean up junction rows

## Description

A fast/cheap execution model, asked to "add a record", reaches for the shortest path that compiles: set the fields with `:=`, then call bare `Insert()`. That path silently skips three platform mechanisms that exist precisely to keep data correct — table triggers, field validation, and buffer initialization — and it compiles clean, so nothing flags it. The result looks right in a demo and rots later: the first time someone adds an `OnInsert` trigger, an `OnValidate`, or a `TableRelation` rule, every one of these hand-written insert sites bypasses it, and no compiler error points back to them.

Three distinct things get skipped, and they are easy to conflate:

1. **`Insert()` / `Modify()` / `Delete()` without the boolean run the *no-trigger* overload.** `Insert(true)` fires the table's `OnInsert` trigger; bare `Insert()` deliberately bypasses it (same for `Modify`/`Delete` and `On…`). If the table has no trigger *today*, `true` is a harmless no-op — but writing `false`-by-omission bakes in a bypass that turns into a silent data bug the moment trigger logic (yours or an extension's) appears.
2. **`:=` bypasses `OnValidate` and the `TableRelation` check.** `ValidateTableRelation = true` and any `OnValidate` logic only run when the field is **validated** (`Rec.Validate(FieldNo, Value)` or UI entry), *not* on a direct `:=` assignment. Assigning a FK with `:=` stores whatever you give it, valid or not — the relation is not enforced.
3. **No `Init()` means a stale record buffer.** A failed `Rec.Get(...)` guard leaves the buffer populated with the values it carried, and skips `InitValue` field defaults. Calling `Insert()` straight after a failed `Get` can therefore carry stale non-key values and miss defaults. Always `Init()` a fresh record before populating it for insert.

A separate but related modeling gap shows up on **junction tables** (an M:N link such as `Vendor Skill` between `Vendor` and `Skill Master`): if neither parent has an `OnDelete` cleanup, deleting a parent leaves the junction rows behind as **orphans** pointing at a key that no longer exists. `TableRelation` does not cascade deletes — it only validates on write.

## Best Practice

Populate through `Validate`, initialize with `Init`, and persist with the trigger-running overload.

```al
// Insert a fresh, validated, trigger-running record
if VendorSkill.Get(VendorNo, SkillCode) then begin
    VendorSkill.Validate("Proficiency Level", ProficiencyLevel);
    VendorSkill.Modify(true);
end else begin
    VendorSkill.Init();                              // fresh buffer + InitValue defaults
    VendorSkill.Validate("Vendor No.", VendorNo);    // runs OnValidate + TableRelation check
    VendorSkill.Validate("Skill Code", SkillCode);
    VendorSkill.Validate("Proficiency Level", ProficiencyLevel);
    VendorSkill.Insert(true);                        // fires OnInsert
end;
```

- **`Insert(true)` / `Modify(true)` / `Delete(true)`** by default. Only pass `false` (or bare) when you have a *documented* reason to bypass triggers (e.g. a controlled data upgrade), and say so in a comment.
- **`Validate(FieldNo, Value)`** for any field that carries a `TableRelation`, an `OnValidate`, or a business rule — that is what enforces referential integrity and field logic. Reserve bare `:=` for the primary-key fields of a brand-new record (where there is nothing to validate yet) or a genuinely rule-free field.
- **`Init()`** before populating a record for insert, especially after a failed `Get`/`FindFirst`, so the buffer is clean and `InitValue` defaults apply.
- **Junction / child tables:** protect each parent with an `OnDelete` that clears the children, so no orphan rows survive:
  ```al
  // tableextension on Vendor (and another on Skill Master)
  trigger OnAfterDelete()  // via an event subscriber or tableextension OnDelete
  var
      VendorSkill: Record "Vendor Skill PIL-SHB";
  begin
      VendorSkill.SetRange("Vendor No.", Rec."No.");
      VendorSkill.DeleteAll(true);
  end;
  ```
  Add a secondary key on each filtered FK field so the cleanup `SetRange`/`DeleteAll` is index-backed, not a full scan.

## Anti Pattern

Direct assignment plus a bare, no-trigger `Insert()` straight after a failed `Get` — skips validation, skips triggers, and can carry a stale buffer:

```al
// Anti-pattern — := bypasses OnValidate/TableRelation, no Init() (stale buffer),
// bare Insert() skips OnInsert. Compiles clean; rots the day a trigger/validation is added.
if not VendorSkill.Get(VendorNo, SkillCode) then begin
    VendorSkill."Vendor No." := VendorNo;          // FK not validated
    VendorSkill."Skill Code" := SkillCode;         // FK not validated
    VendorSkill."Proficiency Level" := ProficiencyLevel;
    VendorSkill.Insert();                           // OnInsert bypassed, no Init()
end;
```

And a junction table whose parents have **no** `OnDelete` cleanup — deleting a `Vendor` or `Skill Master` orphans every linked `Vendor Skill` row. `TableRelation` validates writes; it does **not** cascade deletes, so the cleanup must be explicit.

Note on the guard pattern: manual `Vendor.Get(...)` / `SkillMaster.Get(...)` existence checks before the insert *do* cover referential integrity for those specific keys — but they duplicate what `Validate` against the `TableRelation` would do for free, and they do not help any other field or any future rule. Prefer `Validate`; keep an explicit `Get` guard only when you need a *custom* error/action beyond the stock relation error (e.g. an actionable `ErrorInfo` with a navigation button).
