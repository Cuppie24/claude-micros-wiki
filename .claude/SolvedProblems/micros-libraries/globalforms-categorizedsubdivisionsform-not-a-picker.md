# Gotcha: MC_GlobalForms CategorizedSubdivisionsForm - picker support is opt-in via SelectMode (added 2026-08-25)

**Current behavior (as of 2026-08-25)** — `MC_GlobalForms.Forms.CategorizedSubdivisions.View.CategorizedSubdivisionsForm` (source: `GlobalForms/Forms/CategorizedSubdivisions/View/CategorizedSubdivisionsForm.cs` in the `globalforms` sibling repo) supports two modes, selected by its `SelectMode` bool property (default `false`):

- **`SelectMode = false` (default)** — plain admin CRUD grid for the shared `global_db.categorized_subdivisions` table (add/delete subdivision-category links). `footerPanel` (the "Выбрать"/"Отмена" bar) is hidden. `ShowDialog()` never returns `OK` in this mode - `GetSelectedModel()`/`GetSelectedId()` are still just read-only accessors for the focused row, not picker API, exactly as before.
- **`SelectMode = true`** — acts as a modal picker. Shows `footerPanel` and confirms via the public `SelectedModels` (`List<CategorizedSubdivisionModel>`) property plus `DialogResult.OK`:
  - `selectButton` click → `SelectedModels` = `smartGrid1.GetSelectedRows()` (rows multi-selected via Ctrl/Shift), or the focused `grid.Row`'s model if nothing is multi-selected. Empty selection is a no-op - dialog stays open (same guard pattern as `DeleteSB_Click`).
  - Double-click on a grid row → `SelectedModels` = **only** the focused `grid.Row`'s model, even when other rows are multi-selected via `GetSelectedRows()` - double-click intentionally ignores the multi-selection, unlike `selectButton`.
  - `cancelButton` click → `DialogResult.Cancel`, `SelectedModels` left untouched.

```csharp
using var form = new CategorizedSubdivisionsForm { SelectMode = true, AllowedCategories = new List<int> { 3 } };
if (form.ShowDialog() != DialogResult.OK) return;
var selected = form.SelectedModels; // List<CategorizedSubdivisionModel>, 1+ items
```

`AllowedCategories` still works the same as before and composes with `SelectMode` (e.g. a picker scoped to one category's subdivisions).

**History** — before 2026-08-25 the form had no `SelectMode`/footer UI at all: nothing in its source ever set `this.DialogResult`, so `ShowDialog()` could only ever return `Cancel`. CWATIS's `SubdivisionPickerControl` (`Controls/Subdivisions/View/SubdivisionPickerControl.cs`) hit exactly this and worked around it by using `MSubDivisionsForm { OnlyView = true }` instead (see that form's `_detailGrid_DoubleClick`/`SelectedIds`/`SubDivId`) - that still works and is the right choice when the picker does **not** need category scoping. Prefer `CategorizedSubdivisionsForm { SelectMode = true }` when the picker does need `AllowedCategories`/`TypeFilterCB`-style category scoping.

**Still-valid general rule** — a WinForms `Form` only returns `DialogResult.OK` from `ShowDialog()` if something in its source explicitly sets `this.DialogResult = DialogResult.OK` (or closes via a button whose `DialogResult` is set to `OK`). A class name containing "Form" plus a public `GetSelected*()` method is still not sufficient evidence a form is a picker - grep the form's actual source for `DialogResult` (and, specifically for this form, check whether `SelectMode` exists/is set) before wiring anything up as "pick one and close".

**Checklist / rule**
- `global_db.categorized_subdivisions` (columns: `Id`, `SDId`, `TypeId`/`Type`, `Code`, `Name`) plus `global_db.subdivision_categories` (columns: `Id`, `Name`) is a shared cross-product mechanism (added to `MC_GlobalForms` ~Aug 2026) intended to replace each product's own local "subdivisions" table (e.g. CWATIS's now-retired `cwatis.subdivisions`/`SubdivisionsRepository`). A `SubdivisionCategory` is a named reference row, looked up via `new SubdivisionCategoriesRepository().GetAsync()` and matched by `Name` - there's no fixed/portable numeric id to hardcode, and nothing auto-creates these rows, so a category your product needs may simply not exist yet.
- Use `CategorizedSubdivisionsRepository`/`ICategorizedSubdivisionsRepository` (`GetAsync(search)` / `AddAsync(sdId, subdivDirectoryTypeId)` / `DeleteAsync(id)`) purely as the data layer - independent of `SelectMode`/picking UI.
- The `MC_GlobalForms` NuGet package is opaque, but its full source is checked out on this machine at `D:\projects\projects\globalforms` (sibling repo, not a submodule of any product) - read it directly there instead of decompiling `MC_GlobalForms.dll` or guessing from call sites.

Found in: CWATIS, refactoring `Controls/Subdivisions/View/SubdivisionPickerControl.cs` (2026-08-25, pre-`SelectMode`). `SelectMode`/`SelectedModels` added directly to `CategorizedSubdivisionsForm` in the `globalforms` repo the same day.

> **Update 2026-08-27:** the `new CategorizedSubdivisionsForm { SelectMode = true, ... }` snippets above (parameterless ctor) **throw at runtime** in the shipped package — see next section. Use the 2-arg `(ICategorizedSubdivisionsRepository, ISubdivisionCategoriesRepository)` ctor.

---

## Gotcha: CategorizedSubdivisionsForm's parameterless ctor THROWS at runtime in the shipped package - use the 2-arg (repository) ctor (confirmed 1.1.6.46, 2026-08-27)

**Symptom** — `new CategorizedSubdivisionsForm { SelectMode = true, AllowedCategories = ... }` compiles fine (the parameterless ctor is `public`), but at runtime the ctor throws `System.Exception`: `"Для работы формы требуется репозиторий ICategorizedEntityRepository - используйте конструктор CategorizedSubdivisionsForm(ICategorizedEntityRepository repository)."` — the modal picker never opens. CWATIS's committed `Controls/Subdivisions/View/SubdivisionPickerControl.cs` (`OpenModalPicker`, `using var form = new CategorizedSubdivisionsForm { SelectMode = true, AllowedCategories = SubdivisionCategories };`) does exactly this and is therefore **broken at runtime** against the `MC_GlobalForms` it pins (1.1.6.46) — the ⊞-button path throws.

**Root cause** — two ctors: `()` and `(ICategorizedSubdivisionsRepository, ISubdivisionCategoriesRepository)`. The parameterless one does `InitializeComponent()` then `if (!DesignMode) throw new Exception(...)` — it exists only so the VS designer can instantiate the form. Only the 2-arg ctor builds the presenter. `globalforms` source HEAD has the same guard with newer wording (`"...требуется репозиторий ICategorizedEntityRepository - используйте конструктор CategorizedSubdivisionsForm(ICategorizedEntityRepository repository)."`, commit `c64a1c0`); the shipped 1.1.6.46 DLL throws with the older message above. Either way: parameterless == throws.

**Why "read the globalforms source" (bullet above) isn't enough here** — `globalforms` `development` HEAD runs ahead of any pinned package and the ctor/interface story is mid-refactor (`ICategorizedEntityRepository` vs `ICategorizedSubdivisionsRepository`, DI-enforcement). To know what a *specific* `MC_GlobalForms.<ver>` does, inspect that DLL — PowerShell reflection + IL:
```powershell
$asm  = [Reflection.Assembly]::LoadFile("...\packages\MC_GlobalForms.1.1.6.46\lib\net48\MC_GlobalForms.dll")
$t    = $asm.GetType("MC_GlobalForms.Forms.CategorizedSubdivisions.View.CategorizedSubdivisionsForm")
$t.GetConstructors() | % { $_.ToString() }
$ctor0 = $t.GetConstructors() | ? { $_.GetParameters().Count -eq 0 }
$il = $ctor0.GetMethodBody().GetILAsByteArray()   # ~88 bytes => has logic, not a bare ctor
# scan $il for ldstr (0x72) + $asm.GetModules()[0].ResolveString($tok) => recovers the throw message
```

**Fix** — construct with the 2-arg ctor, passing the *consuming product's* repo impls (`<Product>/Services/CategorizedSubdivisions/Repositories/{CategorizedSubdivisions,SubdivisionCategories}Repository.cs`, hitting `<product>.CategorizedSubdivisions_Get` etc.):
```csharp
using (var form = new CategorizedSubdivisionsForm(
    new CategorizedSubdivisionsRepository(), new SubdivisionCategoriesRepository())
    { SelectMode = true, AllowedCategories = categoryCodes })
{
    if (form.ShowDialog() != DialogResult.OK) return;
    var picked = form.SelectedModels.FirstOrDefault();
}
```
`MainForm` menu wiring in both CWATIS and bookkeep already uses the 2-arg ctor; only CWATIS's `SubdivisionPickerControl` regressed to the parameterless form. The bookkeep port (`Bookkeep2/Controls/Subdivisions/View/SubdivisionPickerControl.cs`, 2026-08-27) uses the 2-arg ctor and is correct.

**Checklist / rule**
- A `public` parameterless WinForms `Form`/`UserControl` ctor is not proof it's runtime-usable — designer-only ctors that `throw` unless `DesignMode` are a common DI-enforcement pattern. Check the ctor body (or disassemble the pinned DLL) for `DesignMode` + `throw` before using object-initializer syntax on it.
- When a `globalforms` type misbehaves, cross-check the **pinned** `MC_GlobalForms.<version>` DLL, not just `D:\projects\projects\globalforms` HEAD — they drift, especially mid-refactor.

Found in: bookkeep, porting CWATIS's `SubdivisionPickerControl` into `Bookkeep2` and pointing `ContractAdd_EditFormNew`'s subdivision picker at it (2026-08-27). `MC_GlobalForms` 1.1.6.46 in both repos.
