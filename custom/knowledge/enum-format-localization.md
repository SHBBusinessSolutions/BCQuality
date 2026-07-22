---
bc-version: [all]
domain: integration
keywords: [format, enum, option, asinteger, localization, localisation, caption, translation, culture, locale, language, de-de, json, api, apipage, odata, integration, payload, serialize, serialization, deserialize, contract, machine-boundary, invariant, stable-token, non-deterministic, round-trip, webhook, edi, marshalling]
technologies: [al]
countries: [w1]
application-area: [all]
---

# Never Format(Enum) across a machine boundary — serialize the ordinal, not the caption

## Description

`Format()` applied to an `Enum` (or `Option`) value returns the **localized caption** of the member, resolved against the *current session's language*. That is correct for something a human reads on screen, but it is a defect the moment the result crosses a machine boundary — a JSON/API/OData payload, an integration message, a webhook body, an EDI file, a stored key, or anything another system parses. The same code then emits `"Basic"` for an `en-US` session and `"Grundlagen"` for a `de-DE` session, so the contract silently depends on who happens to trigger it. A consumer that keyed on `"Basic"` breaks under a German service account with no compile error and no runtime error — the payload is just quietly wrong.

The trap is invisible in review because the code looks innocent (`Format(Rec."Proficiency Level")`), compiles clean, and works in the developer's own language. It surfaces only in production under a different locale, which is exactly the kind of non-deterministic i18n bug that is expensive to trace. A `de-DE`-primary shop (where the running session is frequently German) is especially exposed, because the "wrong" branch is the *common* case, not the edge case.

The boundary is what matters, not the type. Typing the field as an `Enum` (see `prefer-enum-over-option.md`) fixes the *contract at the parameter level*; it does **not** fix serialization. An `Enum`-typed field still emits a localized caption if you `Format()` it into JSON. The two rules are complementary: use an `Enum` for the field/interface/API signature, **and** serialize its ordinal (or a fixed invariant token) when writing the payload.

## Best Practice

When turning an enum value into text that a machine will read, emit a **stable, culture-independent** representation — never the localized caption.

- **Preferred — serialize the ordinal:** `JObject.Add('proficiencyLevel', Rec."Proficiency Level".AsInteger());`. The ordinal is defined by the enum object and never changes with language. This matches how enums round-trip on OData/API pages natively.
- **If you must emit a string token** (e.g. an external contract expects `"Basic"`), map the enum to a fixed literal explicitly via a `case` — so the token is owned by your code and reviewed, not accidentally supplied by the translation layer:
  ```al
  case Rec."Proficiency Level" of
      Rec."Proficiency Level"::Basic:
          Token := 'Basic';
      Rec."Proficiency Level"::Advanced:
          Token := 'Advanced';
      Rec."Proficiency Level"::Expert:
          Token := 'Expert';
  end;
  ```
- **On API pages**, expose the enum field directly and let the platform serialize it — do not build the JSON by hand with `Format()`.
- **Keep `Format(Enum)` for the UI only** — messages, report text, a `Text` control the user reads — where the localized caption is the whole point.
- **Deserialize symmetrically:** read the ordinal back with `Enum::"…".FromInteger(...)` or assign the integer to the enum field; never parse a localized caption back into a value.
- **Align the documented contract with the wire format.** If the interface/API doc says `"proficiencyLevel": 1`, the code must emit the integer — not a string, and certainly not a translated one.

## Anti Pattern

Building an integration/API payload with `Format()` on the enum, so the emitted value follows the session language:

```al
// Anti-pattern — Format() emits the LOCALIZED caption into a machine-read payload.
// en-US session -> "proficiencyLevel":"Basic"
// de-DE session -> "proficiencyLevel":"Grundlagen"   <-- same code, different contract
procedure GetVendorSkills(VendorNo: Code[20]): Text
var
    VendorSkill: Record "Vendor Skill PIL-SHB";
    JObject: JsonObject;
    JArray: JsonArray;
    Result: Text;
begin
    VendorSkill.SetRange("Vendor No.", VendorNo);
    if VendorSkill.FindSet() then
        repeat
            Clear(JObject);
            JObject.Add('skillCode', VendorSkill."Skill Code");
            JObject.Add('proficiencyLevel', Format(VendorSkill."Proficiency Level")); // defect
            JArray.Add(JObject);
        until VendorSkill.Next() = 0;
    JArray.WriteTo(Result);
    exit(Result);
end;
```

Serialize the ordinal instead, giving a stable contract independent of the running language:

```al
// Best practice — AsInteger() is culture-independent and matches the documented contract.
JObject.Add('proficiencyLevel', VendorSkill."Proficiency Level".AsInteger()); // 0 / 10 / 20
```
