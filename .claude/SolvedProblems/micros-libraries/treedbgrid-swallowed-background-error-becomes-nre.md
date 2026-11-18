# TreeDBGrid swallows a failed background DB load and rethrows it as a misleading NullReferenceException

**Date:** 2026-08-26 · **Project:** Refers · **Component:** `Refers.Grid.TreeDBGrid` (any grid using `RefreshProcName` + `GetData(...)` — e.g. `HelpForm.MasterTGrid`)

## Symptom

User-facing dialog (global `CatchHelper` handler) on opening/refreshing a `TreeDBGrid`-based grid:

```
Ошибка приложения
Произошла ошибка. Обратитесь к разработчикам.
Ссылка на объект не указывает на экземпляр объекта.
```

i.e. a generic `NullReferenceException`, with no hint of what actually failed. Reproduced via `HelpForm` open (F1), but the faulty code is in the shared grid base, not Help-specific.

Running under the debugger (VS "Exception Settings" / debug output) shows the *real* sequence, e.g.:

```
Grpc.Core.RpcException: Status(StatusCode="Internal", Detail="FUNCTION refers.Get_m_content_In_Count does not exist ...")
   at ... Core.Client.CallService.CallProcedureDT(String procName, Dictionary`2 dc)
   at Refers.Grid.TreeDBGrid.bgW_DoWork(...) in TreeDBGrid.cs:line 633

System.NullReferenceException: Ссылка на объект не указывает на экземпляр объекта.
   at Refers.Grid.TreeDBGrid.DataToNodes(DataTable table) in TreeDBGrid.cs:line 793
```

The first exception (the actionable one) never reaches the user or the `.err` log — only the second does.

## Root cause

`TreeDBGrid.GetData(...)` loads tree data on a `BackgroundWorker` (`bgW`). `bgW_DoWork` does:

```csharp
this.dtable = CallService.CallProcedureDT(RefreshProcName, parameters);
```

`dtable` is a plain field (`DataTable dtable;`, default `null`, never reset before the async call starts). If `CallProcedureDT` throws — any DB/gRPC failure, e.g. a missing stored procedure/function server-side — the assignment never completes, so `dtable` stays `null` (or stale from a previous successful load).

`BackgroundWorker` semantics: `RunWorkerCompleted` **always** fires after `DoWork`, exception or not — the exception is captured into `RunWorkerCompletedEventArgs.Error`, not rethrown automatically. But `bgW_RunWorkerCompleted` never checked `e.Error`; it unconditionally started with `DataToNodes(dtable)`, which does `foreach (DataRow row in table.Rows)` with no null-check. With `dtable == null`, that's the `NullReferenceException` the user actually sees — the original `RpcException` (with the real, actionable message) is silently discarded by `BackgroundWorker` because nothing ever read `e.Error` or `e.Result`.

This is a bug in the shared grid component, not in any specific form — any form whose `RefreshProcName` procedure fails (missing procedure/function, bad params, DB down, timeout) hits the same masking NRE instead of a readable error.

## Fix

In `Refers\Grid\TreeDBGrid.cs`, `bgW_RunWorkerCompleted`: check `e.Error` first; if set, restore the UI (`BlockControls(false)`, `ShowLoadPic(false)`, `Enabled = true`) and rethrow the *original* exception via `ExceptionDispatchInfo.Capture(e.Error).Throw()` (preserves the original stack trace) **before** touching `dtable`/`DataToNodes`. The app's existing `Application.ThreadException` → `CatchHelper` pipeline then shows the real error (and for an `RpcException` specifically, `CatchHelper.HandleException` already special-cases it to show `rpcEx.Status.Detail`, e.g. "FUNCTION refers.Get_m_content_In_Count does not exist").

## Checklist / rule

- A `NullReferenceException` inside `TreeDBGrid.DataToNodes` (or immediately around a `GetData(...)` call) is almost never really a null-reference bug at that line — it means the background `CallProcedureDT(RefreshProcName, ...)` call failed and the real exception was swallowed. Check the debugger's exception history / VS output for an earlier `RpcException` (or other) right before it, don't just read the NRE's own stack.
- The `.err` log (`Errors\<machine>_<user>.err`) only ever captured whatever `CatchHelper.HandleException` was actually called with — before this fix, that was the downstream NRE, not the useful error. After this fix it captures the real exception.
- Fixing the C# side only surfaces the real error message — it does not fix the underlying cause. A missing DB procedure/function (see [CWATIS stored procedures live only in the live DB, never in the repo](cwatis-stored-procedures-not-in-repo.md) — same "not in the repo, can't grep it" constraint applies to the `refers` schema in this project) still needs to be fixed on the database side; ask the user rather than guessing at the function's body.

Precedent: `Refers\Help\HelpForm.cs` — `MasterTGrid` (`RefreshProcName = "GetHelpsTreeTable"`) failed because `refers.Get_m_content_In_Count` (a function the procedure calls internally) didn't exist on the connected DB; fix applied in `TreeDBGrid.cs` (`bgW_RunWorkerCompleted`) since the bug is generic to the base grid, confirmed via `Refers.csproj` Debug|x86 build (no new warnings/errors).
