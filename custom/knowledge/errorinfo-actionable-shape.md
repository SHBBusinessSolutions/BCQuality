---
bc-version: [bc28, all]
domain: error-handling
keywords: [errorinfo, error-info, actionable-error, collectible, verbosity, errortype, addaction, addnavigationaction, recordid, pageno, title, message, detailedmessage, severity, cause, create, static-factory, hallucination, invalid-member, non-existent-api, self-serve, navigation-action]
technologies: [al]
countries: [w1]
application-area: [all]
---

# ErrorInfo — the members that exist (and the ones models invent)

## Description

`ErrorInfo` is the AL type for raising a rich, **actionable** error: a message with context (the affected record), a chosen verbosity, and optional buttons that let the user fix the problem themselves. It is the target shape the `shb-better-errors` skill rewrites bare `Error('…')` calls into. This note is the **API-surface companion** to that skill: the skill covers *when and how* to author an actionable error; this file pins down *which members actually exist*, because a fast/cheap model frequently hallucinates a plausible-but-nonexistent member (e.g. `.Severity`, `.Cause`, or a fluent `.Create()` on an instance), the app fails to compile, and — worse — a panicked recovery strips the feature back to a plain `Error()` and quietly drops the actionable requirement.

The specific trap that burned a build: the model asserted that `ErrorInfo.Create()` "does not exist" and reached for `.Severity` / `.Cause` (which genuinely do not exist). The truth is the reverse — `Create()` is the real static factory, and the severity concept is spelled `Verbosity`. Getting this surface wrong in either direction costs a compile cycle and risks a lost requirement.

## Best Practice

Build the `ErrorInfo` from the real member surface below, then `Error(ErrInfo)`.

**Members that exist:**
- **Construction:** `ErrorInfo.Create(Message: Text)` and `ErrorInfo.Create(Message: Text; Collectible: Boolean)` — a **static factory qualified on the type**, or declare an `ErrorInfo` variable and set properties directly. Do not call `.Create()` as a fluent method on an already-declared instance.
- **Text:** `Title`, `Message`, `DetailedMessage`.
- **Classification:** `Verbosity` (`Verbosity::Error` …) — this is the "severity" concept — and `ErrorType` (`ErrorType::Client` / `ErrorType::Internal`).
- **Context:** `RecordId`, `TableId`, `FieldNo` / `FieldName`, `PageNo`, `Collectible`, `CustomDimensions`, `DataClassification`.
- **Actions:** `AddAction(Caption: Text; CodeunitId: Integer; MethodName: Text)` runs a codeunit method as a self-serve fix; `AddNavigationAction(Caption: Text)` adds a "go there" button — combine it with `PageNo` and/or `RecordId` so the platform knows where to navigate.

**Members that do NOT exist (do not use — they fail compile):**
- `.Severity` → use `Verbosity`.
- `.Cause` → there is no such property; put the cause in `Message` / `DetailedMessage`.

Recovery rule: if an actionable-error rewrite does not compile, **fix the member name — do not delete the requirement.** Falling back to a bare `Error()` to reach green silently drops the "actionable" acceptance criterion.

## Anti Pattern

Reaching for invented members (and mis-asserting that the real factory is missing), which fails to compile:

```al
// Anti-pattern — .Severity and .Cause do not exist; this does not compile.
ErrInfo.Severity := Severity::Error;   // no such member
ErrInfo.Cause := 'Vendor is blocked';  // no such member
Error(ErrInfo);
```

Use the real surface — the static factory plus `Verbosity`, context, and an actionable button:

```al
// Best practice — real members, actionable, self-serve navigation to the vendor.
var
    VendorBlockedErr: Label 'Vendor %1 is blocked and cannot be assigned skills.', Comment = '%1 = Vendor No.';
    ShowVendorLbl: Label 'Kreditor anzeigen';
    ErrInfo: ErrorInfo;
begin
    ErrInfo := ErrorInfo.Create(StrSubstNo(VendorBlockedErr, Vendor."No."), true);
    ErrInfo.Verbosity := Verbosity::Error;
    ErrInfo.ErrorType := ErrorType::Client;
    ErrInfo.RecordId := Vendor.RecordId();
    ErrInfo.PageNo := Page::"Vendor Card";
    ErrInfo.AddNavigationAction(ShowVendorLbl);
    Error(ErrInfo);
end;
```

See `.github/skills/shb-better-errors/SKILL.md` for when to raise an actionable error and the language rules (German user-facing message via `Label`, English identifiers).
