# MySQL procedure: `IN` parameter mistaken for an output channel

## Symptom

A C# repository method calls an insert-or-update stored procedure via `CallService.CallProcedureNonQuery(name, dic, out error)`, treats the returned `int` as "the new row's id," and stores it (e.g. into a DTO field used later to update/delete that same row). No exception is thrown, `error` is empty, the insert visibly succeeds — but the "new id" the C# side ends up with is wrong/stale, so a later operation keyed on it (a follow-up `UPDATE`/`DELETE` using that id) silently fails or no-ops (`if (id == 0) return;`-style guards swallow it without any visible error).

Confirmed in `CWATIS.Forms.Subdivisions.Repository.SubdivisionsRepository.AddGlobalAsync` calling `global_db.InsUpd_RfrSubDivisions` (2026-08-12) — a newly-created subdivision's `GlobalId` came back wrong, so `deleteCommand_Click` on that same just-created row was a no-op.

## Root cause

The procedure declares the id parameter as `IN` when the developer's intent was clearly `OUT`/`INOUT`:

```sql
create
    definer = bookkeep@`%` procedure global_db.InsUpd_RfrSubDivisions(IN _SubDivId int, IN _Name varchar(255), IN _Code varchar(5))
BEGIN
  ...
  IF IFNULL(_SubDivId, 0) = 0 THEN
    INSERT INTO global_db.rfr_subdivisions (`Name`, `Code`) VALUES (_Name, _Code);
    SELECT LAST_INSERT_ID() INTO _SubDivId;   -- assigns a *local copy* only - IN params can't carry a value back out
  ELSE
    UPDATE global_db.rfr_subdivisions SET `Name` = _Name, `Code` = _Code WHERE SDId = _SubDivId;
  END IF;
END;
```

`SELECT ... INTO _SubDivId` against an `IN` parameter is **valid MySQL syntax** — it doesn't error, doesn't warn, and reads back fine for the rest of the procedure body. But since the parameter is `IN`, that assignment only ever touches the callee's local copy; nothing propagates back to the caller. This is a plain MySQL footgun, not a client-side (`CallService`/ADO.NET) bug — decompiling or instrumenting the C# call layer will not find it, since the C# side genuinely receives *something* (whatever `CallProcedureNonQuery` treats as "the" return value when the target parameter isn't actually `OUT`/`INOUT` — observed as unusable/wrong, not an exception).

**Matching precedent, root cause not independently re-confirmed:** `ContractSubTypeRepository.cs` (`cwatis.ContractSubtypeAddEdit` add/edit methods) has dead, commented-out code where a previous developer tried the exact same `CallProcedureNonQuery(...) → treat return as new id` pattern, abandoned it, and switched to `CallProcedureDT` + reading the new id from a trailing `SELECT` result set instead:
```csharp
// CallService.CallProcedureNonQuery("cwatis.ContractSubtypeAddEdit", _dic, out var res);
var dt = CallService.CallProcedureDT("cwatis.ContractSubtypeAddEdit", _dic);
```
Strongly suggestive of the same class of bug (an `IN` param standing in for an output), but that procedure's own definition wasn't pulled to confirm it directly — treat as a second data point, not proof.

## Fix

Change the parameter direction, nothing else — the body's existing "compute + assign to the id param" logic is usually already correct once it can actually reach the caller:

```sql
create
    definer = bookkeep@`%` procedure global_db.InsUpd_RfrSubDivisions(INOUT _SubDivId int, IN _Name varchar(255), IN _Code varchar(5))
```

`cwatis.subdivisions_add` (the sibling "add to local" procedure in the same codebase) sidesteps the whole issue with a different, equally valid pattern: keep the id param `IN`-only, and instead end the procedure with `SELECT LAST_INSERT_ID() AS Id;` — a real result set, read back via `CallProcedureDT` + `dt.Rows[0]["Id"]` (see `SubdivisionsRepository.AddToLocalAsync`) rather than trying to mutate a parameter. Either fix works; which one applies depends on whether the call site already depends on `CallProcedureNonQuery`'s `out error`/`SIGNAL SQLSTATE` validation-message path (switching to `CallProcedureDT` loses that unless the procedure is also restructured to signal errors some other way) — `InsUpd_RfrSubDivisions` has a real `SIGNAL SQLSTATE '45000'` duplicate-code check wired to `out error`, so the `INOUT`-param fix was preferred there specifically to keep that path intact.

## Checklist

- If a "create/insert" repository method's returned id is used immediately afterward (to update/delete/select the same just-created row) without a full server refresh in between, and that follow-up silently no-ops or fails: suspect this class of bug before anything else — it reproduces 100% of the time, silently, with no exception anywhere in the C# stack.
- Don't trust that a procedure "returns the new id" just because a C# call site's variable is named `newId`/`resultId` and gets returned/assigned somewhere — that only reflects the *developer's intent*, not a verified fact. Get the actual `SHOW CREATE PROCEDURE` text (procedures aren't in the repo — see [cwatis-stored-procedures-not-in-repo.md](cwatis-stored-procedures-not-in-repo.md)) and check the parameter directions and/or trailing `SELECT` yourself.
- `CallProcedureNonQuery`'s `out error` parameter can carry real, user-facing validation messages (duplicate-key checks via `SIGNAL SQLSTATE`, etc.) — don't switch a call site to `CallProcedureDT` (or any other mechanism) to fix an id-return bug without checking whether the procedure also relies on `SIGNAL`/`out error` for validation; that path has no equivalent under `CallProcedureDT`.
