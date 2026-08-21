# CWATIS and bookkeep don't share id spaces for their own per-org reference tables

**Date:** 2026-07-28, updated 2026-07-28 · **Project:** CWATIS + bookkeep integration (BkLink feature) · **Component:** any bookkeep stored procedure call seeded from a CWATIS `ContractModel`

## Symptom

Creating a bookkeep contract from CWATIS data recorded the wrong subdivision — bookkeep showed code `'TM'` when the CWATIS contract's subdivision was `'AHG'`. No error, no exception — the procedure call succeeded and silently stored a real, valid-looking, but wrong subdivision.

## Root cause

CWATIS's `ContractModel.SdId` and bookkeep's `bookkeep.subdivisions.SubDivId` are **not the same id space**, even though both are plain small integers that look interchangeable. Confirmed in `Bookkeep2/ReferencesBox_Components/SubDivisionsRB.cs`: bookkeep's own subdivision lookup queries `bookkeep.subdivisions` **filtered by `Host`** — i.e. bookkeep's `SubDivId` is a **per-organization** sequence local to bookkeep's own schema, unrelated to whatever table CWATIS's own `SdId` comes from. Passing CWATIS's `SdId` straight through as bookkeep's `_SubDivId` just happens to resolve to *some* subdivision that shares that number in bookkeep's per-Host table for the given Host — a coincidental, wrong match, not a real error condition, which is why it stayed silent.

This same shape of bug is worth suspecting for **any** other CWATIS field passed into a bookkeep procedure that looks like a plain reference id (a "type", "kind", "category" id) unless there's a confirmed shared master table behind it — organization/person ids (`ContractSellerId`/`ExecuterId`/etc., backed by the shared `Refers`/MSubject infrastructure both apps use) are fine; anything backed by a **bookkeep-local, Host-scoped** table is not.

## First attempt (superseded) - matching by Code

The first fix matched by the human-readable subdivision code (`bookkeep.subdivisions.Code`, scoped by `Host`) instead of by id, on the theory that codes are the one thing both systems agree on. **This is not reliable either.** Live data across 8 organizations showed the code `"SWR"` mapping to `SDId = 46` for 7 of them, but `SDId = 256` for one (`Host = 7923`) — the same textual code can point at a genuinely different underlying subdivision depending on the organization. Matching by `Code` would silently pick the wrong one for that org, the same class of bug as matching by `SubDivId`, just rarer.

## Actual fix - match by the shared SDId

`bookkeep.subdivisions` has its own `SDId` column, a foreign key into `global_db.rfr_subdivisions.SDId` — a genuinely shared reference table. CWATIS's `ContractModel.SdId` references that same `global_db.rfr_subdivisions.SDId` space (confirmed against live data: CWATIS's own `SdId` for a contract matched the `SDId` bookkeep had recorded for the equivalent subdivision). So the reliable join is `bookkeep.subdivisions.SDId = cwatisContract.SdId AND bookkeep.subdivisions.Host = <host>` - not `SubDivId`, not `Code`.

```csharp
public async Task<int?> ResolveSubDivIdAsync(int? sdId, int hostId)
{
    if (!sdId.HasValue || sdId.Value <= 0) return null;
    return await Task.Run(() =>
    {
        var dt = CallService.GetTable("bookkeep.subdivisions",
            fields: "SubDivId AS ID",
            where: $"Host = {hostId} AND SDId = {sdId.Value}");
        return dt.Rows.Count > 0 ? (int?)MC_GlobalForms.HelperClasses.Numeric.Int(dt.Rows[0]["ID"]) : null;
    });
}
```

If no row matches, that means bookkeep genuinely has no subdivision registered for that org yet (confirmed live: a fair number of organizations are missing subdivisions other orgs already have under the same code) - this is real missing reference data, not a bug, and the calling code (`BkCntrCreatePresenter.CreateAndLinkAsync`) aborts the whole creation rather than silently continuing with an empty subdivision.

Precedent: `CWATIS/Forms/Contracts/BkLink/Repositories/BkCntrRepository.cs` (`ResolveSubDivIdAsync`), called from `CWATIS/Forms/Contracts/BkLink/Presenter/BkCntrCreatePresenter.cs` before calling `bookkeep.CreateOrUpdateContractNew`.

## Checklist / rule

- Never assume a plain-looking int id from a CWATIS model is valid in a bookkeep procedure call just because the parameter name matches — check whether the underlying bookkeep table is Host-scoped/bookkeep-local first.
- Don't assume a human-readable code is a safe fallback either once you've ruled out the id - a shared code string across two independently-administered tables can still map to different underlying rows. Look for an actual shared foreign-key reference table (here, `global_db.rfr_subdivisions`) before trusting a code match.
- When in doubt, get live data before switching the matching column - the `MCode`/`Code` mixup earlier in this same investigation (see `contractsmap-zero-instead-of-null.md`) and the `SDId=256` outlier here were both things a plausible-sounding naming argument would have gotten wrong.
- `MC_GlobalForms.HelperClasses.Numeric` and `Refers.Numeric` are two different classes with the same name - a file with both `using MC_GlobalForms.HelperClasses;` and `using Refers;` gets CS0104 on unqualified `Numeric.Int(...)`; qualify explicitly rather than dropping either using.

## Also seen entirely within bookkeep, no CWATIS involved (2026-08-21)

The same local-`SubDivId`-vs-global-`SDId` confusion turned out not to be specific to the CWATIS boundary — bookkeep is mid-migration, moving various tables' subdivision reference from the local `bookkeep.subdivisions.SubDivId` to the global `global_db.rfr_subdivisions.SDId` one table at a time (tracked in `Задача - Перенос SubDivId на global_db.rfr_subdivisions.md` in the bookkeep repo root, and GitLab issue `microsdvl/bookkeep#1`). `contracts.SubDivId` has already been migrated to the global id; `documents.SubDivId` has not yet. Any bookkeep C# code that copies a subdivision id from one of these tables' rows into a field bound to the other now silently mismatches, same shape of bug as the CWATIS case, no exception, just a wrong or missing match.

Two confirmed instances:
- `Bookkeep2/Operations/Sales/CabinetForDivForm.cs` — the subdivision filter combo (`FiltSDCombo`, bound to the local id) fed `_SubDivId` into `bookkeep.RefreshCabContracts` (which filters `contracts.SubDivId`, global) using the local id. Fixed by resolving the global id from the combo's bound row instead (`FiltSDCombo.Current.Row["LocId"]`) for the Master/contracts query only — the Detail/documents query correctly kept the local id.
- `Bookkeep2/Operations/BankOperations/DayBankStatement2F.cs` (`contractsRB1_ComplPar`) — copied a selected contract's `SubDivId` (global) straight into `subDivisionsRB1.RefId`, a control bound to the local id.

Fix: `Bookkeep2/HelpClasses/DocHelper.cs` → `DocHelper.ResolveLocalSubDivId(int globalSDId, int hostId)` (added 2026-08-21) — the bookkeep-internal counterpart to this file's `ResolveSubDivIdAsync` precedent, same query shape (`bookkeep.subdivisions` WHERE `SDId = ... AND Host = ...`). This consolidates what used to be a private copy of the same method living in `ContractAdd_EditFormNew.cs` (introduced in commit `1b71c69e`, "Refactor SubDivId handling to use global SDId") — that call site was updated to use the shared helper instead of keeping its own copy.

Rule: inside bookkeep's own forms, don't assume two same-named `SubDivId` columns share an id space just because they're both bookkeep tables — the migration is per-table and partial. Before copying a subdivision id between a contract-side value and a document-side (or other not-yet-migrated) field, check whether both sides have actually moved to the global `SDId`, and route through `DocHelper.ResolveLocalSubDivId` when one side is still local.
