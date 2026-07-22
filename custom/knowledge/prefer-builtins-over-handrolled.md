---
bc-version: [all]
domain: style
keywords: [builtin, built-in, platform, reinvent, hand-roll, handroll, text-split, split, list, dictionary, asinteger, setloadfields, string-parsing, comma-separated, csv, loop, repeat-until-false, foreach, invalid-al, boilerplate, helper, wheel, standard-library, methods]
technologies: [al]
countries: [w1]
application-area: [all]
---

# Reach for the built-in before hand-rolling a helper

## Description

The AL runtime ships a large surface of data-type methods — `Text.Split`, `Text.Contains`/`IndexOf`/`Substring`/`Replace`, the `List of [T]` and `Dictionary of [K, V]` collections, `Enum.AsInteger`/`FromInteger`, `SetLoadFields`, `Record.CalcFields`, and many more. These are tested, fast, and instantly recognisable to any reviewer. A recurring failure mode — especially from a fast/cheap execution model that doesn't recall the exact method exists — is to *re-implement* one of them by hand: a manual character-by-character parser instead of `Text.Split(',')`, a running index loop instead of a `foreach` over a `List`, a bespoke lookup instead of a `Dictionary`. The hand-rolled version is longer, carries its own off-by-one and empty-input bugs, obscures intent, and gives a reviewer a page of boilerplate to verify instead of a one-liner they already trust. This is the same "don't reinvent the platform" principle as the system audit fields (see `system-audit-fields.md`) — applied to the standard library rather than to table metadata.

A second, sharper hazard: when the model reaches for a construct it half-remembers, it often reaches for one that **isn't valid AL at all**. AL has no `loop` keyword, no `while (…) { }` C-style form, and `foreach` requires a **pre-declared** loop variable — `foreach var Item in …` does not compile (there is no inline `var` in a `foreach`). Hand-rolled control flow is where these non-AL constructs creep in; a built-in call sidesteps the whole class of error.

## Best Practice

Before writing a helper of more than a few lines, check whether a platform method already does it, and prefer that method.

- **Split a delimited string** with `Text.Split`, which returns a `List of [Text]`:
  ```al
  SkillCodeList := SkillCodes.Split(',');
  foreach SkillCode in SkillCodeList do
      SkillCode := SkillCode.Trim();
  ```
- **Iterate a collection** with `foreach <PreDeclaredVar> in <List>` — declare the loop variable in the `var` section, never inline.
- **Look values up** with a `Dictionary of [K, V]` (`.ContainsKey`, `.Get`, `.Set`) instead of parallel arrays or a scan.
- **Convert an enum to/from its ordinal** with `.AsInteger()` / `Enum::"…".FromInteger()` rather than a `case` ladder (except when you deliberately need a fixed external token — see `enum-format-localization.md`).
- **Trim, search, replace** with `.Trim()`, `.IndexOf()`, `.Contains()`, `.Replace()` rather than manual character walks.
- Only write a custom helper when no built-in fits — and when you do, keep it small and give it a name that states intent.

## Anti Pattern

Re-implementing `Text.Split` with a manual index walk — longer, bug-prone on empty/trailing input, and prone to slipping in non-AL control flow (`loop`, `repeat until false`, inline `foreach var`):

```al
// Anti-pattern — hand-rolled comma splitter that duplicates Text.Split(',')
local procedure ParseCommaSeparatedList(CommaSeparatedText: Text; var ResultList: List of [Text])
var
    StartIndex: Integer;
    CommaIndex: Integer;
    Part: Text;
begin
    ResultList.RemoveRange(1, ResultList.Count());
    if CommaSeparatedText = '' then
        exit;
    StartIndex := 1;
    repeat // 20 lines of IndexOf/Substring to re-derive a one-line built-in
        CommaIndex := CommaSeparatedText.IndexOf(',', StartIndex);
        if CommaIndex = 0 then begin
            Part := CommaSeparatedText.Substring(StartIndex).Trim();
            if Part <> '' then
                ResultList.Add(Part);
            exit;
        end else begin
            Part := CommaSeparatedText.Substring(StartIndex, CommaIndex - StartIndex).Trim();
            if Part <> '' then
                ResultList.Add(Part);
            StartIndex := CommaIndex + 1;
        end;
    until false;
end;
```

Use the built-in and drop the helper entirely:

```al
// Best practice — one line, no helper, no off-by-one surface
SkillCodeList := SkillCodes.Split(',');
// (Text.Split keeps empty entries; skip blanks at the point of use if needed.)
```
