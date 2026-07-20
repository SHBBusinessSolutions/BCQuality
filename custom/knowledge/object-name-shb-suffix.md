---
bc-version: [all]
domain: style
keywords: [object-name, suffix, affix, -SHB, prefix, shb, naming, custom-object]
technologies: [al]
countries: [w1]
application-area: [all]
---

# Suffix every SHB-authored object name with "-SHB"

## Description

At SHB Business Solutions, every AL object that SHB authors — table and page extensions, codeunits, enums, interfaces, permission sets, and new tables/pages — carries the affix `-SHB` at the end of its object name. The suffix makes SHB's own objects instantly distinguishable from Microsoft base objects and from third-party apps in dependency lists, event subscriber pickers, and error stacks, and it keeps the affix consistent across every customer PTE so a reviewer moving between projects never has to relearn the convention. Objects without the suffix blur the boundary between SHB code and everyone else's, which slows review and hand-over and makes ownership ambiguous when a customer or an external developer later reads the extension.

## Best Practice

End the object name with `-SHB`, keeping the descriptive base within the 30-character budget so the suffix fits — for example `pageextension 55000 "Customer List Ext-SHB"`, `codeunit 55010 "Sales Posting Mgt.-SHB"`, `enum 55020 "Delivery Status-SHB"`. Plan the descriptive part around the affix from the start rather than discovering the length limit at publish time. Apply the suffix uniformly to every object SHB creates, regardless of type or customer.

## Anti Pattern

Object names with no affix (`pageextension 55001 "Vendor List Ext"`), an inconsistent or ad-hoc marker (`"Vendor List Ext SHB"`, `"SHB_VendorListExt"`, a leading `SHB` prefix), or a name padded to 30 characters that leaves no room for `-SHB` and forces a rename during hand-over.
