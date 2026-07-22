---
bc-version: [all]
domain: data-modeling
keywords: [validate, direct-assignment, onvalidate, field-validation, reference-field, tablerelation, business-logic, programmatic-write, insert, modify, testfield, dependent-field, referential-integrity, get-guard, skip-validation, silent-skip, future-onvalidate]
technologies: [al]
countries: [w1]
application-area: [all]
---

# Validate reference and dependent fields when writing a record in business logic

## Description

When code populates a record programmatically — building a row to `Insert` or updating one to `Modify` — AL offers two ways to set a field: direct assignment (`Rec."Field" := Value`) and `Rec.Validate("Field", Value)`. They are not equivalent. `Validate` runs the field's `OnValidate` trigger and its table-relation check; direct assignment does neither. It writes the raw value into the in-memory record and moves on.

For a field that today has no `OnValidate` and no `TableRelation`, the two forms produce the same stored result, so direct assignment looks harmless. The hazard is not the present — it is the future. The moment someone adds `OnValidate` logic to that field (default a dependent field from it, copy a description, recompute a total, block a value), every direct assignment that set it **silently skips the new logic**. No analyzer fires, the code still compiles, and the dependent data quietly drifts out of sync. A cheap/fast model reaches for `:=` because it is one token shorter and "works" in the test it just wrote — exactly the reasoning that plants the trap.

The same silent-skip is why `DataTransfer` and `CopyFields` carry an explicit warning (see `datatransfer-skips-triggers-and-subscribers.md`). This note is the everyday, row-at-a-time counterpart: not a bulk-upgrade concern, just ordinary business logic that creates or edits a record.

## Best Practice

When writing a record from code, use `Validate` for any field that has (or could plausibly grow) an `OnValidate` trigger or a `TableRelation` — reference fields above all. Reserve direct assignment for fields you are certain carry no validation semantics and never will (the primary-key parts you are about to `Insert`, a scratch flag, a value you have *already* validated by another path).

- **Reference fields** (`TableRelation` to a master/setup) → `Validate`, so the relation is checked and any lookup-driven defaulting fires:
  ```al
  VendorSkill.Init();
  VendorSkill.Validate("Vendor No.", VendorNo);
  VendorSkill.Validate("Skill Code", SkillCode);
  VendorSkill.Validate("Proficiency Level", ProficiencyLevel);
  VendorSkill.Insert(true);
  ```
- **Fields with dependent logic** (one field defaults or recomputes another) → `Validate`, in the same order the UI would.
- **Explicit `Get` guards are an acceptable substitute for the *relation* check — not for `OnValidate`.** If the procedure already proves the reference exists with a guarded `Get` that raises a richer, actionable `ErrorInfo` (better than the platform's generic "does not exist in table" message), that guard covers referential integrity. But it does **not** run the field's `OnValidate`; if the field has (or gains) dependent logic, still `Validate` it. Do not treat a `Get` guard as a blanket licence to `:=` every field.
- **Primary-key fields before `Insert`** are the normal exception — assigning the PK parts directly and then `Insert(true)` is idiomatic; `Insert` itself runs `OnInsert`.

## Anti Pattern

Populating a reference field by direct assignment in business logic, so a future `OnValidate` on that field is silently bypassed:

```al
// Anti-pattern — direct assignment skips OnValidate and the table-relation check.
// If "Skill Code" later gains an OnValidate that defaults "Proficiency Level"
// from the skill master, this code silently stops honoring it — no warning.
VendorSkill."Vendor No." := VendorNo;
VendorSkill."Skill Code" := SkillCode;
VendorSkill."Proficiency Level" := ProficiencyLevel;
VendorSkill.Insert(true);
```

Prefer `Validate`, keeping any actionable `Get` guards for the error-message quality they add:

```al
// Best practice — guards give a rich actionable error; Validate runs field logic.
if not Vendor.Get(VendorNo) then
    Error(ErrInfo);              // actionable ErrorInfo (see errorinfo-actionable-shape.md)

VendorSkill.Init();
VendorSkill.Validate("Vendor No.", VendorNo);
VendorSkill.Validate("Skill Code", SkillCode);
VendorSkill.Validate("Proficiency Level", ProficiencyLevel);
VendorSkill.Insert(true);
```

Do not over-apply the rule: `Validate` on a field that genuinely has no validation semantics and no relation is harmless but noise. The signal to reach for `Validate` is a `TableRelation`, an existing `OnValidate`, or a field whose value another field depends on. This is the row-level twin of `datatransfer-skips-triggers-and-subscribers.md`, which covers the same silent-skip for set-based `DataTransfer`/`CopyFields`.
