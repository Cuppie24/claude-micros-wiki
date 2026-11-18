# GlobalForms's SmartGrid_Extentions.cs had no way to correctly read a bound model off a C1DataCollectionBindingList-backed row

**Date:** 2026-08-24 · **Project:** GlobalForms (`D:\projects\projects\globalforms`). **Component:** `GlobalForms\HelperClasses\Common\Extentions\SmartGrid_Extentions.cs` (`MicrosSmartGrid.SmartGrid` extension methods).

This is the GlobalForms-side instance of the same underlying `MicrosSmartGrid`/`C1DataCollectionBindingList` gotcha documented for CWATIS in `../../Topics/SmartGridConventions.md` ("Gotcha: `Rows[row].DataSource` is not always the bound model directly") — same root cause, but GlobalForms has its own separate extension-methods file and, until now, no fix for it. Read that entry too if working on grid code in either project; this entry is GlobalForms-specific (its own file, its own missing helper).

## Symptom

Code written against a `SmartGrid` whose `DataSource` is bound via `new C1DataCollectionBindingList(...)` (the standard binding pattern in this project — see `UnitsForm.LoadMasterGrid`, and every `smartGrid1.DataSource = new C1DataCollectionBindingList(...)` call) needs the model object actually bound to a given row (not just its visible-column values) — typically to read an id/foreign-key property that isn't rendered as its own grid column. There was no working local method for this: `SmartGrid_Extentions.cs` only had `GetValueRow<T>`/`GetValues<T>`/`GetValueRowByIndex<T>`, all of which either (a) reconstruct `T` by reading `grid.Rows[index][prop.Name]` for *every* property of `T` by name — which throws/fails for any property that has no matching grid column (e.g. a hidden `Id`, or a joined-in foreign key like `TypeId` that's carried on the model but never rendered), or (b) attempt `grid.Rows[rowIndex].DataSource as T` directly, which silently returns `null`/a blank `new T()` for any row that hasn't been individually focused yet — because C1DataCollectionBindingList wraps not-yet-resolved rows in `C1.DataCollection.BindingList.IDataCollectionBindingListItem`, not the raw model.

Ported-in code that assumed a `GetSelectedRowModel<T>()` extension existed (it doesn't, anywhere in this project's own source) failed to compile once an unrelated transitive package that happened to provide same-named types was removed (see [../build-and-tooling/globalforms-refersv2-self-referential-mc-globalforms-package.md](../build-and-tooling/globalforms-refersv2-self-referential-mc-globalforms-package.md)) — that package's removal was what exposed this gap, not a separate bug.

## Root cause

Confirmed via reflection against the referenced `packages\C1.DataCollection.BindingList.1.0.20223.121\lib\net452\C1.DataCollection.BindingList.dll`: `C1.DataCollection.BindingList.IDataCollectionBindingListItem` is a real public interface with exactly one member, `object Item { get; }`. None of the three pre-existing `SmartGrid_Extentions.cs` methods ever check for or unwrap this interface — they either bypass `DataSource` entirely (reading grid cells instead) or cast `DataSource` straight to `T`, which only succeeds once C1's binding-list has already resolved/cached that particular row (e.g. after it's been individually selected/focused at least once).

## Fix / rule

Added two methods to `SmartGrid_Extentions.cs`, matching the pattern CWATIS's `SmartGridExtentions.cs` already uses under the names `GetModelRow<T>`/`GetModelSelectedRow<T>` (see `../../Topics/SmartGridConventions.md`'s "Getting the id" / gotcha sections — same interface, same unwrap, just a second independent implementation since the two projects don't share this helper file):

```csharp
public static T GetModelRow<T>(this SmartGrid grid, int index) where T : class, new()
{
    if (index < grid.Rows.Fixed || index >= grid.Rows.Count) return new T();

    var rawSource = grid.Rows[index].DataSource;
    if (rawSource is T direct) return direct;
    if (rawSource is IDataCollectionBindingListItem wrapper && wrapper.Item is T unwrapped) return unwrapped;

    return grid.GetRowDataByIndex<T>(index); // existing per-cell fallback for already-resolved/edited rows
}

public static T GetModelSelectedRow<T>(this SmartGrid grid) where T : class, new()
    => grid.GetModelRow<T>(grid.Row);
```

Requires `using C1.DataCollection.BindingList;` in the extensions file. Callers also need `using MC_GlobalForms.HelperClasses;` explicitly — unlike plain type lookup (which finds `MainClass`/`GFUserSingleton` automatically from a nested `MC_GlobalForms.Forms.X.Y` namespace because `MC_GlobalForms` is a lexical ancestor), C# extension-method lookup additionally requires the extension class's *own* namespace to be `using`-imported for the `grid.GetModelSelectedRow<T>()` call syntax to bind, even though `MC_GlobalForms.HelperClasses` and `MC_GlobalForms.Forms.X.Y` are namespace siblings, not ancestor/descendant.

**Rule:** in this project, never call `grid.Rows[index].DataSource as T` (or reflect over `T`'s properties reading `grid.Rows[index][prop.Name]` for properties that aren't real grid columns) directly in form code for a `C1DataCollectionBindingList`-bound grid — use `GetModelRow<T>`/`GetModelSelectedRow<T>` instead. The existing `GetValueRow<T>`/`GetValues<T>` remain correct for the narrower case where every property of `T` genuinely has a same-named grid column.
