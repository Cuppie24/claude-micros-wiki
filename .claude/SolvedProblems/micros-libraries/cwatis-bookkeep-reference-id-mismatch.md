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
- When in doubt, get live data before switching the matching column - the `MCode`/`Code` mixup earlier in this same investigation (see `../../../projects/CWATIS/.claude/docs/SolvedProblems/contractsmap-zero-instead-of-null.md`) and the `SDId=256` outlier here were both things a plausible-sounding naming argument would have gotten wrong.
- `MC_GlobalForms.HelperClasses.Numeric` and `Refers.Numeric` are two different classes with the same name - a file with both `using MC_GlobalForms.HelperClasses;` and `using Refers;` gets CS0104 on unqualified `Numeric.Int(...)`; qualify explicitly rather than dropping either using.

## Also seen entirely within bookkeep, no CWATIS involved (2026-08-21)

The same local-`SubDivId`-vs-global-`SDId` confusion turned out not to be specific to the CWATIS boundary — bookkeep is mid-migration, moving various tables' subdivision reference from the local `bookkeep.subdivisions.SubDivId` to the global `global_db.rfr_subdivisions.SDId` one table at a time (tracked in `Задача - Перенос SubDivId на global_db.rfr_subdivisions.md` in the bookkeep repo root, and GitLab issue `microsdvl/bookkeep#1`). `contracts.SubDivId` has already been migrated to the global id; `documents.SubDivId` has not yet. Any bookkeep C# code that copies a subdivision id from one of these tables' rows into a field bound to the other now silently mismatches, same shape of bug as the CWATIS case, no exception, just a wrong or missing match.

Two confirmed instances:
- `Bookkeep2/Operations/Sales/CabinetForDivForm.cs` — the subdivision filter combo (`FiltSDCombo`, bound to the local id) fed `_SubDivId` into `bookkeep.RefreshCabContracts` (which filters `contracts.SubDivId`, global) using the local id. Fixed by resolving the global id from the combo's bound row instead (`FiltSDCombo.Current.Row["LocId"]`) for the Master/contracts query only — the Detail/documents query correctly kept the local id.
- `Bookkeep2/Operations/BankOperations/DayBankStatement2F.cs` (`contractsRB1_ComplPar`) — copied a selected contract's `SubDivId` (global) straight into `subDivisionsRB1.RefId`, a control bound to the local id.

Fix: `Bookkeep2/HelpClasses/DocHelper.cs` → `DocHelper.ResolveLocalSubDivId(int globalSDId, int hostId)` (added 2026-08-21) — the bookkeep-internal counterpart to this file's `ResolveSubDivIdAsync` precedent, same query shape (`bookkeep.subdivisions` WHERE `SDId = ... AND Host = ...`). This consolidates what used to be a private copy of the same method living in `ContractAdd_EditFormNew.cs` (introduced in commit `1b71c69e`, "Refactor SubDivId handling to use global SDId") — that call site was updated to use the shared helper instead of keeping its own copy.

Rule: inside bookkeep's own forms, don't assume two same-named `SubDivId` columns share an id space just because they're both bookkeep tables — the migration is per-table and partial. Before copying a subdivision id between a contract-side value and a document-side (or other not-yet-migrated) field, check whether both sides have actually moved to the global `SDId`, and route through `DocHelper.ResolveLocalSubDivId` when one side is still local.

## Third instance: a *report* table that denormalizes both sides into one column (2026-09-02)

Found in bookkeep's `RepDebCredOrgsForm_New3` (отчёт Дебиторы-Кредиторы). Symptom: subdivision blank or
wrong on contract rows, subdivision filter drops them; document rows fine; **only in reports recalculated
after** the `contracts.SubDivId` migration - older reports still render correctly, which makes it look
like a UI regression rather than a data one.

The new shape here: the report table `bookkeep.repdebcredcontr3` is **polymorphic** (`BlockType`: 0=contract,
1=document, 2=GTD, 3=taxes; `BlockDocId` points at `contracts.ContrId` / `documents.DocId` /
`realistypes2level.Rt2lId` accordingly, no FK), and its `SubDivId` column is filled by
`bookkeep.RepDebCredMakerC3` with

```sql
,CASE v.BlockType WHEN 0 then c.SubDivId WHEN 1 THEN d.SubDivId END
```

i.e. **one column holding two different id spaces** - global from `contracts`, local from `documents` -
while the reader (`RefreshRepDebcredContracts3`, both `_ID=0` and `_ID=100` branches) resolves it purely as
local (`subdivisions sb ON sb.SubDivId=ro.SubDivId` → `rfr_subdivisions rs ON rs.SDId=sb.SDId`) and filters
on `TRIM(rs.Code)=TRIM(_SubDivId)`. Duplicated verbatim in `RepDebCredMakerC4`/`RefreshRepDebcredContracts4`.

Two traps that make the "obvious" minimal fix wrong:

- **`contracts.SubDivId_Local` and `documents.SubDivId_Global` are one-off migration snapshots, not
  maintained.** Verified on the live DB: no trigger on `contracts` at all, and
  `information_schema.ROUTINES WHERE ROUTINE_DEFINITION LIKE '%SubDivId_Local%'` / `'%SubDivId_Global%'`
  return **zero** procedures; no `.cs` file mentions either name. `documents.SubDivId_Global` was already
  missing for 2 of the last 5000 documents. So neither column may be *read* as an authoritative value -
  resolve live through `subdivisions` (`SubDivId` ↔ `SDId`, scoped by `Host`) instead.
- The `X_Local`/`X_Global` companion column existing (and even being 100% in sync today) is **not** evidence
  it is kept in sync. Check for a trigger/procedure that writes it before depending on it.

Diagnosing "which id space is in this row": don't guess from magnitude. Two reliable tests, both confirmed
here - (a) `contracts.SubDivId` never numerically equals `SubDivId_Local` (0 of 25 694 rows), so a per-row
match against both columns classifies unambiguously; (b) the report's own recalculation timestamp -
`repdebcredlist3.GenTime`, set by `UPDATE repdebcredlist3 SET GenTime = NOW()` at the **end of the maker**,
so it is last-calc time, not creation time - splits pre/post migration cleanly.

Repair of already-computed data: normalize the report column to the **global** id (the migration's direction),
converting historical local values with `JOIN subdivisions s ON s.SubDivId=r.SubDivId AND s.Host=rl.Host
SET r.SubDivId=s.SDId`. The `Host` scoping is mandatory. Procedure change and backfill must land in the same
maintenance window, or the display is wrong for converted rows in between. Full plan with counts and
verification queries: `Задача - Перенос SubDivId в repdebcredcontr3.md` in the bookkeep repo root.

Note for future sessions: bookkeep's stored procedures are **not** in the repo (zero `.sql` files), but they
are readable read-only from the dev server `Mars.micros.uz:3306` - compile a tiny C# runner against
`packages/MySql.Data.*/lib/net48/MySql.Data.dll` with `csc.exe` and run it from the folder holding the DLL
(PowerShell 5.1 `Add-Type` on that DLL blows up with a `ReflectionTypeLoadException`, then a
`StackOverflowException` in the `AssemblyResolve` handler - not worth fighting). Also: the client calls the
maker as `RepDebCredMaker3`, but the actual procedure is `bookkeep.RepDebCredMakerC3` - a plain
`SHOW CREATE PROCEDURE` on the client-side name returns "does not exist"; search
`information_schema.ROUTINES` by `LIKE` instead of trusting the name in the C# call.

**Scope check before you plan the repair - is the mixed-id column actually load-bearing?** Here it was not.
`SubDivId` appears in `RepDebCredMakerC3` only in the INSERT column list, in that one `CASE`, and (for the
sibling operations table) in a code-resolution join - never in a `WHERE`, a `JOIN` condition, or a `GROUP BY`;
the temp table every sum is aggregated from does not even carry a subdivision column, and the reader has no
outer `GROUP BY`. So no money column was corrupted and nothing had to be recomputed for correctness - it was
a display defect. That collapsed the repair from "four procedures plus a 104k-row backfill" to "two
procedures, recalculate one report". Always enumerate the column's occurrences in the writer *and* check the
GROUP BY lists before sizing the fix; a denormalized reference column in a report table is usually just a
label, and the migration-direction-purist fix is not automatically the right one.

The one thing that keeps it from being purely cosmetic: the same column is the backing store for a *filter*
(`TRIM(rs.Code)=TRIM(_SubDivId)`), and WinForms grids typically compute footers and derived rows client-side
over whatever the filtered query returned (`MasterTGrid.CalcFooter` → `SetTable4ContrSaldo()`). So a wrong
label still produces wrong on-screen totals whenever the user filters by it - check for client-side
`CalcFooter`/footer-derived rows before calling a label bug harmless.

**`bookkeep.subdivisions` is not unique on `(SDId, Host)` - a lookup JOIN there can silently multiply sums.**
Confirmed 2026-09-02: 8 duplicate `(SDId, Host)` pairs, worst `Host = 6877, SDId = 24 (KRZ)` with **18**
local rows, plus `Host = 7923, SDId = 5 (ADM)` with three - both orgs run the report. So the natural-looking
fix "add `LEFT JOIN subdivisions csb ON csb.SDId = c.SubDivId AND csb.Host = THost` and read `csb.SubDivId`"
is a real defect when it lands inside an `INSERT ... SELECT ... GROUP BY` whose projection is all `SUM(...)`:
the duplicate matches fan out the source rows and inflate the money by up to 18x, with no error. Use a
scalar subquery instead - it cannot multiply rows:

```sql
(SELECT MIN(s.SubDivId) FROM subdivisions s WHERE s.SDId = c.SubDivId AND s.Host = THost)
```

`MIN()` rather than `LIMIT 1` because `LIMIT` without `ORDER BY` is not stable across runs; the duplicates
all point at the same `SDId` anyway, so any candidate renders the same code. The mirror-image join
(`ON dsb.SubDivId = d.SubDivId`, i.e. global-from-local) is safe because it hits the PRIMARY KEY and is
strictly 1:1 - direction matters.

General rule: before adding a reference-lookup JOIN into an aggregating statement, verify the join key is
actually unique on the target. "It's a small reference table" is not the same as "it's a key". Also note
`bookkeep.subdivisions` no longer carries `Code`/`MCode` (removed in `d6d3a146`) - it is now purely a
`(SubDivId, Host, SDId)` link table, so every human-readable code comes from `global_db.rfr_subdivisions`.
