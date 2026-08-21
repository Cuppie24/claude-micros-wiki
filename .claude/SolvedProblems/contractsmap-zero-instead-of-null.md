# ContractsMap maps an absent DB value to 0, not null, on several nullable ContractModel fields

**Date:** 2026-07-28 · **Project:** CWATIS · **Component:** `CWATIS.Forms.Contracts.Mappers.ContractsMap` / `ContractModel`

## Symptom

Downstream code (e.g. a stored procedure call on another system) fails with a foreign-key error like:
```
Cannot add or update a child row: a foreign key constraint fails (...CONSTRAINT `FK_contracts_m_subject_ResponsibleId` FOREIGN KEY (`ResponsibleId`) REFERENCES `global_db`.`m_subject` (`ID_M_SUBJ`))
```
even though the field was believed to be optional/nullable and a null-check (`model.SomeId == null`) had already been applied to guard it. The check passes (doesn't flag it as missing) but the value sent downstream is wrong, not absent.

## Root cause

`ContractsMap.Map()` (`CWATIS/Forms/Contracts/Mappers/ContractsMap.cs`) uses this pattern for many `int?` properties on `ContractModel`:
```csharp
contract.ExecuterId = Convert.IsDBNull(entity["ExecuterId"]) ? 0 : Numeric.Int(entity["ExecuterId"]);
```
A genuine SQL `NULL` becomes the C# value **`0`**, not `null` — despite the property being declared `int?`, which reads as "null means absent." Confirmed present for (at least): `ContractSellerId`, `ContractBuyerId`, `ContractShipperId`, `ContractConsigneeId`, `CurrencyId`, `CurrencyPaymentId`, `InitiatorId`, `ExecuterId`, `SdId` (line numbers ~21-46, ~60 as of 2026-07-28). By contrast, `KindId` (line 38) and `BkContrId` (line 79) in the *same* mapper correctly use `(int?)null`, so the pattern isn't consistent even within the one file — don't assume all `int?` fields behave the same way; check each one.

Anything that does `model.SomeId == null` to check "is this set" will silently pass a mapped `0` through as if it were a real, meaningful id. If that `0` is then forwarded to something with a foreign-key constraint on the equivalent real table, MySQL correctly rejects it (`0` is not `NULL` and doesn't reference a real row) — the FK error is doing its job; the bug is upstream, in trusting the null-check.

## Fix

When consuming any of these fields for something that treats "unset" as meaningful (validation, or forwarding to another system with FK constraints), check `<= 0`, not `== null`:

```csharp
private static bool IsMissing(int? id) => !id.HasValue || id.Value <= 0;
private static int? NormalizeId(int? id) => IsMissing(id) ? null : id;
```

Use `IsMissing` for required-field validation, and `NormalizeId` before forwarding an optional id downstream (so a mapped `0` becomes a real `NULL` at the boundary, instead of a false "valid-looking" id).

Precedent: `CWATIS/Forms/Contracts/BkLink/Presenter/BkCntrCreatePresenter.cs` (`GetMissingRequiredFields`, `BuildCreateDto`) — building the parameters for `bookkeep.CreateOrUpdateContractNew`, which has real FK constraints on `Host`/`OrgId`/`CurrencyId`/`ResponsibleId`.

## Checklist / rule

- Never trust `ContractModel.SomeId == null` alone as an "is this set" check - verify against `ContractsMap.cs` first, since the mapper may have already collapsed null into 0 for that specific field.
- When forwarding a `ContractModel` id field to anything with a foreign-key constraint on the real target table, normalize `0`/negative to `null` first rather than passing it through - the FK constraint will otherwise catch it at the worst possible time (a cryptic constraint-name error surfaced to the end user) instead of your own clear validation message.
- This is a mapper inconsistency, not a deliberate convention - don't assume it holds for fields not explicitly checked here; grep `ContractsMap.cs` for the field name before relying on it.

## Don't over-apply this - only fields backed by an actual `int?` are affected

Wrongly applied this exact "0 means missing" check to `ContractModel.CurrencyTypeId` and it broke on real data. `CurrencyTypeId` is declared as a plain non-nullable `int` in `ContractModel` (not `int?`), and its target table `global_db.cntr_currencytypes` legitimately uses `0` as a real row ("Базовая"/base currency) - confirmed both by a user report of live data and by `ContractsBigForm.cs`'s commented reference (`AddCustomItem(0, "Базовая")`). For a non-nullable `int` mapped as `Convert.IsDBNull(...) ? 0 : ...`, `0` could mean "was null" **or** "was genuinely zero" and the two are indistinguishable from the C# value alone - so `<= 0` is not a safe "missing" signal there. Only apply `IsMissing`/`NormalizeId` to fields that are themselves `int?` in `ContractModel` (i.e. where the mapper is *throwing away* information the type could have represented) - check the property's declared type before assuming the same fix applies.
