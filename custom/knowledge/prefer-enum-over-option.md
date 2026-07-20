---
bc-version: [all]
domain: data-modeling
keywords: [enum, option, optionmembers, optioncaptions, interface, extensible, enum-extension, enumextension, type-safety, proficiency, status, state, magic-number, ordinal, ordinal-spacing, spacing, gap, reserved-range, initvalue, default-value, negative-ordinal, data-type, field-type, parameter-type, method-signature]
technologies: [al]
countries: [w1]
application-area: [all]
---

# Prefer Enum over Option — and require Enum on interface and API signatures

## Description

AL offers two ways to model a small fixed set of choices: the legacy `Option` type (an in-place list of `OptionMembers` on a single field or variable) and the first-class `Enum` object. An `Enum` is a named, reusable, optionally **extensible** type with its own object ID, per-value `Caption`, and explicit ordinal values. An `Option` is an anonymous, per-declaration list whose members must be repeated (and kept in sync) everywhere the option is used, and whose values are positional and easy to desynchronise.

`Enum` is the modern default. It gives compile-time type safety (you cannot assign an unrelated ordinal), a single source of truth reused across tables, pages, variables and method parameters, real captions that translate through the normal XLF pipeline, and — via `enumextension` — a supported extension point so other apps (or later waves) can add values without touching the base object. `Option` gives none of this: every consumer restates the members, translations live in scattered `OptionCaptions`, and an extra or reordered member silently shifts the meaning of stored ordinals.

The gap becomes a hard blocker at a **contract boundary**. Interfaces and API/OData signatures cannot express an `Option` member set — the parameter degrades to a bare `Integer`, so the caller passes a magic number (0/1/2), the compiler stops checking, and the implementing codeunit has to validate and convert back to the field's option. Typing the parameter as an `Enum` removes the whole marshalling layer: the field, the interface method, and every caller share one type the compiler enforces end to end.

## Best Practice

Model any fixed set of choices as a dedicated `Enum` object, and reference that enum from the table field, the interface method, the API, and the code that consumes it.

- Create an `enum` object (suffixed per SHB naming, e.g. `enum 55001 "Proficiency Level PIL-SHB"`) with explicit ordinal `value(...)` entries and a `Caption` on each value.
- Type the **table field** as `field(3; "Proficiency Level"; Enum "Proficiency Level PIL-SHB")` — drop the inline `OptionMembers`/`OptionCaptions`.
- Type **interface and API parameters/returns** as the enum, never as `Integer` standing in for an option. This is what lets the field, the contract, and the caller share one compiler-checked type.
- Mark the enum `Extensible = true` when other apps or later waves may need to add values; extend it with an `enumextension` rather than editing the base enum. `Extensible` — not ordinal spacing — is what actually permits an extension; without it no `enumextension` can target the enum at all.
- **Space the ordinals** (e.g. `0, 10, 20`) for any enum that represents an ordered scale or is likely to grow — the leading gap lets a later value slot *between* two existing ones and keep the correct display/sort order (enums render and sort by ordinal). Do this only where an in-between value is plausible; a truly closed set (e.g. `No`/`Yes`) can stay contiguous. Spacing does **not** prevent collisions between extensions — that is handled by each extension using ordinals from its own reserved band (typically tied to its object-ID range).
- **Ordinals must be non-negative** — AL enum values are `0 … ~2,000,000,000`; `value(-10; …)` does not compile. The only way to reserve headroom *below* the first member is to offset the whole scale up (e.g. `10, 20, 30`) so ordinal `0` stays free — not to go negative.
- **Decouple the field default from the ordinal with `InitValue`.** Ordinal `0` is the raw storage default of an enum field, so which member sits at `0` otherwise decides the default. Setting `InitValue = <Member>` on the field makes the default explicit and independent of numbering, so you can offset the scale and still default correctly. Caveat: `InitValue` only applies on `Rec.Init` and new-record-via-UI — a bare `Rec.Insert()` without `Init`, a `TransferFields`, or a direct buffer write still lands on ordinal `0`. So an offset scale whose `0` is unnamed can still produce a member-less value on those paths; keeping a meaningful member at `0` (and adding `InitValue` for clarity) is the robust default.
- When migrating an existing `Option` field to an `Enum`, keep the ordinal values identical so stored data stays valid. Choose the final spacing **before** any data is written; re-numbering a populated enum requires an upgrade codeunit to remap stored ordinals.

## Anti Pattern

Declaring an inline `Option` on the field and then being forced to a bare `Integer` at the interface boundary — the caller passes a magic number and the implementer converts it back:

```al
// Anti-pattern — inline Option on the field...
field(3; "Proficiency Level"; Option)
{
    OptionMembers = Basic,Advanced,Expert;
    OptionCaptions = 'Basic', 'Advanced', 'Expert';
}

// ...forces the interface to degrade to Integer (no type safety, magic numbers):
interface "Vendor Skill Management PIL-SHB"
{
    // caller must "know" 0=Basic, 1=Advanced, 2=Expert; compiler checks nothing
    procedure AddSkill(VendorNo: Code[20]; SkillCode: Code[20]; ProficiencyLevel: Integer);
}
```

Prefer one shared `Enum` used by both the field and the contract:

```al
// Best practice — one reusable, type-safe, extensible enum with spaced ordinals
enum 55001 "Proficiency Level PIL-SHB"
{
    Extensible = true;

    value(0; Basic) { Caption = 'Basic'; }
    value(10; Advanced) { Caption = 'Advanced'; }
    value(20; Expert) { Caption = 'Expert'; }
    // an extension can later add value(5; Intermediate) between Basic and Advanced
}

// field and interface share the same compiler-checked type — no conversion layer
field(3; "Proficiency Level"; Enum "Proficiency Level PIL-SHB") { }

procedure AddSkill(VendorNo: Code[20]; SkillCode: Code[20]; ProficiencyLevel: Enum "Proficiency Level PIL-SHB");
```

Reserve `Option` for a genuinely local, throwaway, non-persisted, non-exposed toggle. Anything stored on a table, shown to a user, translated, or crossing an interface/API boundary should be an `Enum`.
