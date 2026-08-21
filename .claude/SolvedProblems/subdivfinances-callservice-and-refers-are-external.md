# SubdivFinances: CallService.cs is dead code, and the `Refers` submodules are never checked out

**Date:** 2026-08-20 · **Project:** SubdivFinances (`D:\projects\subdivfinances`) · **Component:** anything touching `CallService.*` calls or the `Refers` namespace (`RfrSubDivisionsForm`, `CustomCombo`, `URefBox`, `TDBG`, etc.)

## Symptom

Asked to trace how `CallService.CallProcedureDT(...)` actually executes, or to find/edit the source of a `Refers`-namespace class (e.g. `RfrSubDivisionsForm`), in order to fix or understand a bug. Both leads look like they should resolve to in-repo source and don't:

- `SubdivFinances\CallService.cs` exists, is named exactly right, and defines `public static class CallService` with all the expected methods (`CallProcedureDT`, `CallProcedureNonQuery`, etc.) — but grepping/reading it shows the **entire file body is commented out** (every line prefixed `//`), including the `namespace`/`class` declarations themselves. It is legacy WCF-era code (`SFServiceClient`) fully disabled, not the active implementation, despite having the exact name you'd search for.
- `git submodule status` shows `submodules/refers`, `submodules/grp-service`, `submodules/dss` all prefixed `-` (uninitialized) and the directories are empty. There is no `.gitmodules`-cloned source for the `Refers` namespace anywhere in the working tree.

## Root cause

The calls that actually run (`CallService.CallProcedureDT(...)` etc., used throughout the `Main`/`Refers`/`RWSMain` forms) resolve through `SubdivFinances\GlobalUsings.cs`:

```csharp
global using GRP_Client = Core.Client;
global using Core.Client;
```

`CallService` is a type inside the `Core.Client` namespace, brought in by the second line — it is **not** the commented-out class in `CallService.cs`. The real implementation ships precompiled inside the `GRP-Client.V2`/`Core.Abstractions` NuGet packages (see `SubdivFinances.csproj` references, `HintPath` under `..\packages\GRP-Client.V2.*\lib\net48\` and `..\packages\WebServiceClient.*\lib\net48\Core.Abstractions.dll`). This matches the repo's recent commit "Refactor: Replace GRP_Client with CallService" — call sites were repointed from `GRP_Client.Xxx(...)` to `CallService.Xxx(...)`, both aliases into the same external `Core.Client` library; the in-repo `CallService.cs` was left behind, fully commented out, as dead weight.

Separately, `RfrSubDivisionsForm`, `RfrSubDivisionsF`, `Refers.CustomCombo`, `Refers.URefBox`, `Refers.TDBG`, etc. (referenced via `using Refers;`) come from the **`Refers.V2` NuGet package** (`..\packages\Refers.V2.*\lib\net48\Refers.V2.dll`), a completely different thing from the `submodules/refers` git submodule. The submodule points at `https://gitlab.micros.uz/micros-dvl/refers.git` — presumably that repo's source produces the `Refers.V2` package — but the submodule is never initialized in this working copy, so it's a dead end for exploration, not a shortcut to the source.

Net effect: **stored-procedure call mechanics and every `Refers`-namespace UI component are opaque external binaries from this repo's point of view.** There is no in-repo source to read, patch, or blame for either.

## Fix / rule

- Don't trust a filename match. If `SubdivFinances\CallService.cs` (or any file that looks like "the" implementation) doesn't compile-contribute anything, check whether it's fully commented out before assuming it's dead-end investigation — grep hits inside `//`-prefixed lines still count as a match but aren't live code.
- To find what a bare identifier like `CallService` actually resolves to, check `SubdivFinances\GlobalUsings.cs` first — this project uses `global using X = Y;` aliases, so the type may not share a name with any file in the repo at all.
- Don't try to `git submodule update --init` `submodules/refers`/`grp-service`/`dss` expecting to find `Refers`/`CallService` source — treat these three submodules as unused in this checkout; the actual dependency is the compiled `Refers.V2` / `GRP-Client.V2` / `Core.Abstractions` NuGet packages referenced in `SubdivFinances.csproj`.
- Practical consequence: this repo has **zero raw SQL, zero ORM entity classes**, and all DB access is dynamic `DataTable`/`DataRow` string-indexer access via `CallService.CallProcedureDT/CallProcedureNonQuery/GetTable`. When hunting for a bug caused by a DB schema change (e.g. a dropped column), the only place bugs can hide in *this* repo is a hardcoded column-name string literal (`dr["SomeColumn"]`, `SetDataBinding(dt, "Id", "SomeColumn", ...)`) on a `DataTable` sourced from the affected stored procedure — the procedure internals and the `Refers.V2` package internals are both out of reach and out of scope for an in-repo fix.

Precedent: fixing `SubdivFinances\Users\UserRightsForm_New.cs` (`SDIdsCombo.SetDataSource(SDIds, "LocId", "MCode")` → `"Name"`) after `bookkeep.subdivisions`/`global_db.rfr_subdivisions` dropped their `MCode` column — the stored procedures were already migrated (out of scope, and unreachable from this repo anyway), so the only fixable thing was this one hardcoded column-name literal, found by cross-referencing every consumer of `MainClass.SubDivDT`/`Accessed*SubDivDT` rather than by tracing the call mechanics.
