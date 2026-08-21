Two related tables held duplicate/dead data. Schema was just cleaned up directly in the DB:

**bookkeep.subdivisions** (local, per-organization)
- Before: SubDivId, Name, MCode, Code, Host, ParaId, SDId
- After: SubDivId, Host, SDId — **removed MCode, Code, ParaId, Name**

**global_db.rfr_subdivisions** (global reference directory)
- Before: SDId, Name, MCode, Code
- After: SDId, Name, Code — **removed MCode**

bookkeep.subdivisions.SDId is a FK into global_db.rfr_subdivisions.SDId.

## Already done

All stored procedures that used to return bookkeep.subdivisions.Code or bookkeep.subdivisions.Name have been updated to internally JOIN global_db.rfr_subdivisions ON SDId and return Code/Name from there instead. **Callers going through stored procedures need no changes.**

## What's broken / needs fixing

Any application code that reads bookkeep.subdivisions (or rfr_subdivisions) **directly** — raw SQL, ORM entities/models, query builders — bypassing the stored procedures, since it may reference now-nonexistent columns.

Specifically look for and fix:

1. **MCode** — column is gone entirely from both tables. Find every usage (queries, ORM properties, DTOs, UI bindings, reports) and remove it. It was ~90% duplicate of Name, so if it's genuinely needed somewhere for display, consider substituting Name.
2. **ParaId** (bookkeep.subdivisions only) — was fully unused; remove all references (should be low/no risk, but verify).
3. **Code** (bookkeep.subdivisions only) — if accessed via direct query (not through the already-updated procedures), the query must be changed to JOIN global_db.rfr_subdivisions rs ON bs.SDId = rs.SDId and pull rs.Code instead of bs.Code.
4. **Name** (bookkeep.subdivisions only) — same fix pattern as Code: any direct query pulling bs.Name must instead JOIN global_db.rfr_subdivisions rs ON bs.SDId = rs.SDId and pull rs.Name.

## Steps for Claude Code

1. Search the codebase for all references to subdivisions / rfr_subdivisions (table names, ORM model/entity class names, migration files, raw SQL strings).
2. Classify each usage: goes through a stored procedure (skip — already fixed) vs. direct table access (needs review).
3. For direct-access code, check for use of .MCode, .Code, .ParaId, .Name fields (on bookkeep.subdivisions) and apply the fixes above.
4. Update any ORM entity/model/DTO class definitions that mirror the old bookkeep.subdivisions or global_db.rfr_subdivisions schema so their fields match the new columns.
5. Update any local migration scripts / schema docs / seed data that still reference the dropped columns.
6. Build and run tests to catch compile-time or runtime failures from the removed fields.

## Out of scope

- Stored procedures — already migrated, don't touch unless a bug is found.

---

## Related but separate: local SubDivId vs global SDId (2026-08-21)

Besides the Code/MCode/Name column cleanup above, a **second, related** migration is in progress: various tables' `SubDivId` are being switched from referencing the local `bookkeep.subdivisions.SubDivId` to referencing `global_db.rfr_subdivisions.SDId` directly - table by table, not all at once. Tracked in the bookkeep repo's own `Задача - Перенос SubDivId на global_db.rfr_subdivisions.md`, and GitLab issue [microsdvl/bookkeep#1](https://gitlab.micros.uz/microsdvl/bookkeep/-/work_items/1).

**Status per table (as of 2026-08-21):**

Finished:
- `contracts.SubDivId` - already migrated to the global `SDId`.

Not finished - current scope of work:
- `bookkeep.documents.SubDivId`
- `bookkeep.depcards.SubDivId`
- `bookkeep.paymentorders.SubDivId`

These three are the active target: moving their references off local `bookkeep.subdivisions` and onto `global_db.rfr_subdivisions`. Don't treat any of the three as done until explicitly confirmed here - update this list per-table as each one actually completes, rather than describing the group as a single blended state.

### The "contract -> document" bug pattern

Any bookkeep C# code that copies a subdivision id from a `contracts`-sourced row (global `SDId`) straight into a UI control or field bound to the local id space (`bookkeep.subdivisions.SubDivId` - what every subdivision picker combo/reference-box in the app uses, since local ids are what the per-user rights table is keyed to) silently ends up with the wrong subdivision selected - no SQL error, just wrong or empty data. Same shape of bug, opposite direction, when a local-bound combo's value is sent into a query that filters a column holding the global id.

Fix: `DocHelper.ResolveLocalSubDivId(int globalSDId, int hostId)` in `bookkeep/Bookkeep2/HelpClasses/DocHelper.cs` - resolves the local `bookkeep.subdivisions.SubDivId` from a global `SDId` + `Host` (`bookkeep.subdivisions` is Host-scoped, so the lookup needs both). The companion direction (local -> global) is `ContractAdd_EditFormNew.ResolveGlobalSDId`, via `bookkeep.RefreshSubDivisions`.

Confirmed instances fixed 2026-08-21, all in the `bookkeep` repo:
- `Bookkeep2/Operations/Sales/CabinetForDivForm.cs` - the subdivision filter combo was sending the local id into the Master/`contracts` query (`bookkeep.RefreshCabContracts`, which filters on the global id) - fixed to resolve local -> global for that one query only; the Detail/`documents` query correctly kept the local id as-is.
- `Bookkeep2/Operations/BankOperations/DayBankStatement2F.cs` (`contractsRB1_ComplPar`) - a selected contract's global `SubDivId` was copied straight into a local-bound reference box.
- `Bookkeep2/Operations/Sales/SaleUZSF.cs` (~line 186 and ~294) - same pattern, both the "new document from contract" path and an edit-mode fallback (`CSubDivId` = raw `cont.SubDivId`, unresolved).
- `Bookkeep2/Operations/BuyUZSNew/Views/BuyUZSForm.cs` (`B_ContractRB_ComplPar`) - same pattern as `DayBankStatement2F`, a selected contract's global `SubDivId` copied straight into `B_subDivisionsRB`.
- `Bookkeep2/Operations/Invoice/ImportInvoiceForm.cs:1814` (`InvContractRB_ValueChangedByUser`) - same pattern, `subDivisionsRB1` (the actual document subdivision picker, feeds `documents.SubDivId` on save) filled from the selected contract's global id. Same file also has three read-only, display-only `ContrSubDiviRB`/`ContrSubDiviRB2` writes (lines 1825, 3716, 3825) with the identical unresolved assignment, but their `RefId` is never read back anywhere - left alone, cosmetic only.
- `Bookkeep2/Operations/Contract/View/ContractAdd_EditFormNew.cs` - had a private copy of the same resolve logic; deduped to call the shared `DocHelper` method instead.

Still outstanding, once `documents.SubDivId` itself migrates to global (currently correct *because* it's still local - full list in the GitLab issue above): the save-side lines in `SaleUZSF.cs`/`DayBankStatement2F.cs` that pass the local combo value straight into `documents.SubDivId`, and two places that populate the UI from a *document's own* (not the contract's) `SubDivId`.

### Correction to "Already done" above

That claim doesn't fully hold. Two procedures crashed at runtime with `Unknown column 'sb.Code' in field list` this session - the exact symptom of a procedure never updated to JOIN `global_db.rfr_subdivisions` after `bookkeep.subdivisions.Code` was dropped:
- `bookkeep.RefreshCabSaleWork` - confirmed (pasted, inspected, fixed server-side by the user, 2026-08-21).
- `bookkeep.RefreshPaymentOrdersNew` - same error text reproduced in `PaymentOrdersForm` on 2026-08-21, strongly suspected same cause, **not yet confirmed or fixed** (procedure source not yet checked).
- `bookkeep.RefreshImportInvoice` - `ImportInvoiceForm`'s sole master-grid procedure, fires unconditionally on open the same way the other two did; not yet reproduced/confirmed as the actual source of a matching error, just the leading suspect by the same pattern.

Don't assume "already migrated" for a given procedure until it's actually been exercised or its source checked - verify per-procedure rather than trusting this note.