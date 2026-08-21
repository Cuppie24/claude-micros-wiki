# SmartGrid Conventions (CWATIS)

Conventions for working with `MicrosSmartGrid.SmartGrid` (and `TreeFlexGrid`) in the CWATIS WinForms client. See also `SmartGridExtentions.cs` (`CWATIS\Extentions\SmartGridExtentions.cs`) for the extension methods referenced below, and the project's `SmartGridWiki.md` for general grid feature documentation (headers, context menu, tooltips, tree building, etc. — legacy/English, do not extend it; the active wiki is the `CwatisWiki` vault).

## Setting `DataSource`

Always assign the grid's `DataSource` via `C1DataCollectionBindingList`, not a raw `List<T>`/array — binding a plain list disables column-header sorting. Wrap the assignment in `BeginUpdate()`/`EndUpdate()` to suppress redraw flicker while the grid rebinds.

### Pattern

```csharp
masterGrid.BeginUpdate();
masterGrid.DataSource = new C1DataCollectionBindingList(contractTypes);
masterGrid.EndUpdate();
```

Precedent: `ContractDSForm.cs`, `PriceEditPanel.cs`, `CompletedWorksForm.cs`, `WareHouseCardForm.cs`, among others.

## CRUD Without a Full Refresh: Editing `C1DataCollectionBindingList` Directly

`C1DataCollectionBindingList` implements `System.Collections.IList` and `System.ComponentModel.IBindingList` directly (confirmed via reflection against `C1.DataCollection.BindingList.dll`, package `C1.DataCollection.BindingList` v1.0.20223.121) — `Insert(int, object)`, `Add(object)`, `RemoveAt(int)`, and the `this[int]` indexer **getter** all exist, work, and raise `ListChanged`, which the grid is already subscribed to. **The indexer *setter* is a different story — it's an unimplemented stub, see the Update section's gotcha below; reflection only proves a member's signature exists, not that its body does anything.** That means a create, update, or delete doesn't need a full re-fetch + `DataSource` reassignment (the "Setting `DataSource`" pattern above) — editing the already-bound list in place is enough, and is cheaper (no server round trip to re-list, no `BeginUpdate`/`EndUpdate` rebind, selection is never lost).

### Create: insert above the selected row

After the backend confirms the create (and hands back the new row's id), insert the new item into the *live bound* list at the position just above the currently selected row — not wherever it would actually belong once sorted/filtered. Getting the "real" position right isn't the goal here and isn't attempted — the next genuine refresh (`DataSource` reassigned from a fresh server fetch) re-sorts/re-filters everything back into place. Insert at index `0` when nothing is selected.

```csharp
public void InsertRow(SomeDto newRow)
{
    _allRows.Insert(0, newRow); // keep the refresh-time cache in sync too - see below

    var insertIndex = grid.Row >= grid.Rows.Fixed ? grid.Row - grid.Rows.Fixed : 0;

    var bindingList = (C1DataCollectionBindingList)grid.DataSource;
    bindingList.Insert(insertIndex, newRow);

    grid.Row = grid.Rows.Fixed + insertIndex;
}
```

### Update: replace the item at its current row, in place

Same idea for an edit: once the backend call succeeds, build (or receive back from the backend) the updated object and splice it into the bound list at the edited row's current index — don't reassign `DataSource` and don't call whatever `OnRefresh` normally does.

**Gotcha: don't use the indexer setter (`bindingList[i] = updated`) — it's an unimplemented C1 stub.** `C1DataCollectionBindingList.set_Item` exists (it has to, to satisfy `IList`) but its body just throws `System.NotImplementedException`, with `Source=C1.DataCollection.BindingList` — the exception comes from inside the C1 assembly itself, not from a mistake in the calling code. This shipped in an earlier revision of this doc because the claim above was based on reflection ("the member exists") rather than an actual call ("the member works") — confirmed broken via a real crash in production (`SubdivisionsForm.SetLocalId`, `NotImplementedException` at `C1.DataCollection.BindingList.C1DataCollectionBindingList.set_Item`, 2026-08-18). **Use `RemoveAt` + `Insert` at the same index instead** — both are already confirmed working (same primitives Create/Delete below use), so this needs no new API:

Find the row by searching the **bound list's own items**, not by going through `grid.FindModel<T>(Predicate<T>)`/`Rows[i].DataSource` (the "Restoring the Selected Row After a Refresh" technique above) — `FindModel` reads each row's `DataSource` off the *grid*, and a `SubdivisionsForm` case where a delete went through server-side but didn't show up in the grid until the next real refresh pointed at that path being unreliable for a list mutated in place via `Insert`/`RemoveAt` (not conclusively isolated — see the confirmed-vs-not breakdown below).

Searching the list directly needs its own unwrap, though — **`C1DataCollectionBindingList`'s own indexer getter (`this[int]`) does NOT return the raw item you stored.** It always comes back wrapped in `C1.DataCollection.BindingList.IDataCollectionBindingListItem`, the exact same wrapper the "`Rows[row].DataSource` is not always the bound model directly" gotcha below describes for the grid layer — confirmed via an instrumented headless test that talks to `C1.DataCollection.BindingList.dll` directly (`Insert`/`Add` an item, then read it back via the indexer: every read comes back as `DataCollectionBindingListItem`, unconditionally, not just for not-yet-selected rows the way the grid-level wrapping is). Unwrap it the same way `SmartGridExtentions.UnwrapModel<T>` does:

```csharp
private static int FindBindingListIndex(C1DataCollectionBindingList bindingList, int id)
{
    for (var i = 0; i < bindingList.Count; i++)
    {
        var raw = bindingList[i];
        var item = raw as SomeDto ?? (raw as IDataCollectionBindingListItem)?.Item as SomeDto;
        if (item != null && item.Id == id)
            return i;
    }
    return -1;
}

public void UpdateRow(SomeDto updated)
{
    var cacheIndex = _allRows.FindIndex(x => x.Id == updated.Id);
    if (cacheIndex >= 0) _allRows[cacheIndex] = updated;

    var bindingList = (C1DataCollectionBindingList)grid.DataSource;
    var listIndex = FindBindingListIndex(bindingList, updated.Id);
    if (listIndex < 0) return;

    bindingList.RemoveAt(listIndex);
    bindingList.Insert(listIndex, updated);
    grid.Row = grid.Rows.Fixed + listIndex; // RemoveAt+Insert is two ListChanged events, not one atomic replace - reassert selection explicitly rather than assume it survives both
}
```

**What's actually confirmed vs. still open**, since this convention went through two rounds of guessing before landing here — don't repeat either mistake:

- Confirmed by instrumented test (`C1.DataCollection.dll`/`C1.DataCollection.BindingList.dll` loaded directly, no grid, no license needed): the indexer-wrapping behavior above. Also confirmed: `new C1DataCollectionBindingList(someList)` copies `someList`'s items into internal storage at construction time — it does **not** keep a live reference to the `IEnumerable` you passed in, so mutating that original source list afterward has no effect on the bound list (and vice versa). Relevant if a form's cache list (`_allRows`) is ever passed straight through as the `filtered` argument without a defensive `.ToList()` first — check whether that's actually the same object before assuming a mutation needs to be applied to both, though the copy-on-construct behavior means it usually doesn't matter either way.
- Confirmed by real runtime crash, not just the headless test (which only ever exercised `Insert`/`Add`, never the setter): the indexer **setter** throws `NotImplementedException` unconditionally. Never call `bindingList[i] = x` — use `RemoveAt`+`Insert`.
- **Not confirmed:** whether `grid.FindModel`/`Rows[i].DataSource` genuinely becomes unreliable specifically *after a real refresh* has occurred, as opposed to some other variable in that test session. A `C1.Win.C1FlexGrid.C1FlexGrid`/`SmartGrid` can't be instantiated headlessly outside a licensed VS/build environment ("No license for the ComponentOne WinForms Edition can be found"), so this half of the picture is still diagnosed from a user-reported symptom, not from stepping through the grid. If a future case contradicts the "search the list directly" fix, this is the piece to re-open first.

### Delete: remove the item from the same live list

Same idea again, mirrored: once the backend confirms the delete, remove the item from the bound list by index — don't reassign `DataSource`, don't call `OnRefresh`. Same `FindBindingListIndex` lookup as Update, for the same reason.

```csharp
public void RemoveRow(int id)
{
    var cacheIndex = _allRows.FindIndex(x => x.Id == id);
    if (cacheIndex >= 0) _allRows.RemoveAt(cacheIndex);

    var bindingList = (C1DataCollectionBindingList)grid.DataSource;
    var listIndex = FindBindingListIndex(bindingList, id);
    if (listIndex < 0) return;

    bindingList.RemoveAt(listIndex);
}
```

### Keep the client-side cache in sync, not just the grid

Forms that re-filter client-side (a cached `_allX` list, re-sliced into a new `C1DataCollectionBindingList` on every filter-combo change, without hitting the server — see e.g. `SubdivisionsForm.ApplyLocalFilter`) must patch **both** the cache and the live bound list, in the same call. Miss the cache and the next client-side-only re-filter (not a real server refresh) silently reverts the row, since it rebuilds `DataSource` from the stale cache.

### Where the "updated object" comes from

Sometimes the backend call already returns enough to build it client-side (a create dialog already collected `Code`/`Name`; the new id is the only unknown, and the insert procedure returns it — just merge). Sometimes only *part* of the row changed and the new value is already known deterministically without asking the backend anything at all (e.g. "remove from local" always means "this FK becomes null" — no round trip needed to learn that). Don't guess a stored procedure's result-set shape to avoid an extra call — see `SolvedProblems/cwatis-stored-procedures-not-in-repo.md` (procedures aren't in the repo and can't be checked by reading code); confirm the actual column name with whoever owns the DB first.

Precedent: `SubdivisionsForm.cs` (`InsertRow` for `createCommand`/`OnAdd`; `SetLocalId` for `addToLocalCommand`/`removeFromLocalCommand` — the latter is a partial-field update, not a full-row edit, hence the narrower `SetLocalId(globalId, localId)` view method instead of a whole-row `UpdateRow`; `RemoveRow` for `deleteCommand`/`OnDelete`).

**Older precedent, different (pre-convention) list type:** `PriceSectionsForm.cs`, `ContractsSubTypeForm.cs`, `TemplDocsViewForm.cs` already do the same insert-above-selection / replace-in-place trick, but against a plain `System.ComponentModel.BindingList<T>` — the type the "Setting `DataSource`" rule above already steers new code away from. The list-editing technique they use is right; don't copy the list type.

## "Record Info" Context Menu Item

Many CWATIS list forms add a right-click context menu item on their grid that shows raw record info for the selected row.

### Pattern

```csharp
grid.AddItemToContextMenu("Информация о записи", Resources.Info, (_, _) =>
{
    // resolve the selected row's id (see below), then:
    Main.RecInfoView(id, "table_name");
});
```

`Main.RecInfoView(int id, string tableName)` lives in `CWATIS\HelperClasses\Main.cs`. The `tableName` string matches the underlying MySQL table (e.g. `"contracttypes"`, `"contracts"`, `"contractlines"`, `"pricelistrows"`).

### Getting the id

- If the grid has a real bound column for the key (e.g. `"ContractId"`, `"ActId"`, `"PreContrId"`), index the grid directly: `(int)grid[grid.Row, "ColumnName"]`.
- If the grid has **no bound id column** (`AutoGenerateColumns=false` and the id property isn't in `ColumnInfo`, or `IdName` is `null`), do **not** guess a column name (e.g. `"Id"`) — indexing by a name that isn't in `ColumnInfo` throws. Instead extract the id from the row's bound DTO:
  - Whole-model extraction: `grid.GetModelSelectedRow<TDto>()`, then read `.YourIdProperty`. Guard against a blank/default model when no row is selected (e.g. `selected == null || selected.YourIdProperty == 0`).
  - Or a direct cast when only the id is needed: `(grid.Rows[grid.Row].DataSource as TDto)?.YourIdProperty` (precedent: `TemplDocsViewForm.cs`).

## Restoring the Selected Row After a Refresh

After a re-search rebinds `DataSource` (new `C1DataCollectionBindingList`), the grid's selection is gone — `grid.Row` resets, so whatever the user had selected is lost even though the same underlying row (by key) is very likely still present in the new result set.

Use `grid.FindModel<TDto>(predicate)` (`SmartGridExtentions.cs`) — it already exists for exactly this ("find the row whose model matches a condition, return its index or -1"), and is the established idiom across the codebase (`OrgsGroupForm.cs`, `GoodsFSiteForm.cs`, `CompanyClientReportForm.cs`, `DeptSellReportForm.cs`, `RepManagerSalesList.cs`). Don't hand-roll the scan loop yourself — that's just reimplementing `FindModel` worse. `TDto` must implement `IEquatable<object>` (an `Equals(object obj)` override comparing the id field is enough — see `GroupModel.cs`/`BkCntrModel.cs` for the two-line pattern; the `Predicate<T>` overload doesn't call `Equals` itself but still requires the constraint).

```csharp
private async Task RaiseSearch(FilterDTO filter)
{
    int selectedId = grid.Row >= grid.Rows.Fixed ? grid.GetModelSelectedRow<TDto>().SomeIdProperty : 0;

    // ... rebind DataSource via the search ...

    if (selectedId != 0)
        grid.Row = grid.FindModel<TDto>(m => m.SomeIdProperty == selectedId);
}
```

Capture the key from the bound model *before* the refresh (same "Getting the id" technique as above, just applied at refresh time instead of for a context-menu lookup), then look it up again afterward with `FindModel` once the new data is bound.

**Gotcha (fixed as of 2026-07-17, but worth knowing):** `FindModel`'s two overloads used to cast `Row.DataSource` straight to `T` without unwrapping `IDataCollectionBindingListItem` — the exact same trap as the "`Rows[row].DataSource` is not always the bound model directly" gotcha below, just in a different method. That meant `FindModel` silently failed (returned `-1`) for any row that hadn't already been individually selected/focused on a `C1DataCollectionBindingList`-bound grid, i.e. it was unreliable for almost the exact case it's meant to handle. Both overloads now route through a shared `UnwrapModel<T>` helper (same unwrap `GetModelRow<T>` already did correctly) — no caller-side workaround needed anymore, just use `FindModel` normally.

Precedent: `BkCntrSelectForm.cs` (`RaiseSearch` restoring the selection by `ContrId`; `SelectLinkedRow` uses the same `FindModel` call with a different predicate, for jumping to the row linked to the current CWATIS contract rather than restoring a prior user selection).

## Handling Double-Click on a Specific Column

To react to a double-click on one particular column (not the whole row), wire `MouseDoubleClick`, not the plain `DoubleClick` event. `MouseDoubleClick` carries `MouseEventArgs` (`e.X`/`e.Y`), so the clicked cell can be resolved directly with `grid.HitTest(e.X, e.Y)` in the same handler — no need for a separate `MouseDown` handler to stash the hit-tested column first (that older pattern exists in `ContractsSubTypeForm.cs`'s `_detailGrid_MouseDown` + `_detailGrid_DoubleClick`; prefer `MouseDoubleClick` for new code).

```csharp
private void grid_MouseDoubleClick(object sender, MouseEventArgs e)
{
    var hit = grid.HitTest(e.X, e.Y);
    if (hit.Row < grid.Rows.Fixed || hit.Column != grid.Cols["SomeColumn"]?.Index) return;

    var selected = grid.GetModelRow<TDto>(hit.Row); // not GetModelSelectedRow - hit.Row may differ from grid.Row until the click is processed
    if (selected == null) return;
    // ...
}
```

Use `Cols["Name"]?.Index` for named columns instead of a hardcoded index (magic number) — see precedent `ProductSalesRepListForm.cs`, which does use a raw index (`hit.Column != 18`) and is the wrong part to copy.

Precedent: `ContractTypesForm.cs` (`masterGrid_MouseDoubleClick`, opens `JSONVIewForm` from the `HasJson` column).

## Displaying Images in Grid Cells

The established idiom for attaching an image to a grid cell is imperative, not declarative: call the native `SetCellImage(row, col, image)` method, or set `e.Image` inside the grid's `OwnerDrawCell` handler, then clear the text with `e.Text = string.Empty`. A declarative `Column.DataType = typeof(Image)` approach was tried once (`TemplDocsViewForm.cs`, now commented out) and abandoned in favor of this.

**Standing rule: every cell image must be center-aligned and scaled to fit the cell.** Set the column's `ImageAlign` to `C1.Win.C1FlexGrid.ImageAlignEnum.Scale` once at grid setup (constructor, right after `InitializeComponent()` — `Cols["Name"]` only resolves once `ColumnInfo` has been parsed, which happens as part of `InitializeComponent()`). `Scale` both centers the image and resizes it to fit the cell's current bounds while preserving aspect ratio (confirmed against the local `C1.Win.C1FlexGrid.4.8.xml` doc: "Image is scaled to fit the maximum area within the cell while preserving the original aspect ratio" — `ImageAlignEnum` is a plain enum, not flags, so there is no separate "center" setting to combine with it; `Scale` already implies centered). Use `ImageAlignEnum.Stretch` only if aspect-ratio distortion is acceptable — normally it isn't.

```csharp
public SomeForm()
{
    InitializeComponent();
    masterGrid.Cols[0].ImageAlign = C1.Win.C1FlexGrid.ImageAlignEnum.Scale;
    masterGrid.Cols["SomeFlagColumn"].ImageAlign = C1.Win.C1FlexGrid.ImageAlignEnum.Scale;
    ...
}
```

Precedent: `ContractTypesForm.cs` (approve checkmark column + `HasJson` column).

### Pattern — fixed icon

```csharp
private void grid_OwnerDrawCell(object sender, C1.Win.C1FlexGrid.OwnerDrawCellEventArgs e)
{
    if (e.Col == grid.Cols["SomeFlagColumn"]?.Index)
    {
        e.Image = Properties.Resources.check7; // or grid.SetCellImage(e.Row, e.Col, image);
        e.Text = string.Empty;
    }
}
```

Older precedent (`TemplDocsViewForm.cs`, `ContractsForm.cs`, `ContractsSubTypeForm.cs`, `ProductMoveRepListForm.cs`, `UserRightsForm.cs`) draws these at natural size without setting `ImageAlign` — that predates the standing rule above; don't copy that part when adding new icon columns, only the `OwnerDrawCell` wiring.

### Pattern — resizable image (photo/scan/attachment)

For anything larger than a fixed icon — a photo, a scanned document thumbnail, etc. — the `ImageAlign = Scale` step above still applies, plus:

1. `Scale` only fits the image inside the *existing* cell bounds; it does not grow them. Explicitly size the row height / column width for a thumbnail when populating the grid (e.g. `masterGrid.Rows[r].Height = 56;`), otherwise the image gets squeezed into a single text-line-height sliver.
2. Downscale the source image once when the row's data loads, rather than handing the grid a full-resolution bitmap per row — a DB-sourced photo/scan can be multi-megapixel, and keeping N full-size bitmaps in memory and rescaling them on every repaint is wasteful. Cache the thumbnail on the row's model rather than recomputing it inside `OwnerDrawCell` (which fires on every repaint). No reusable thumbnail helper exists yet in this codebase — add one when needed (e.g. an extension method `CreateThumbnail(this Image source, int maxWidth, int maxHeight)` in `SmartGridExtentions.cs` or a new `ImageExtensions.cs` under `CWATIS\Extentions\`), using `Graphics.DrawImage` with `InterpolationMode.HighQualityBicubic`.
3. Optional: since a thumbnail loses detail, pair it with a double-click handler on that column that opens a small modal (`PictureBox` with `SizeMode = Zoom`) showing the original full-resolution image. No such viewer exists yet in this codebase.

**Why:** researched across the whole project (all `OwnerDrawCell`/`SetCellImage`/`e.Image` usages, all `PictureBox` usages, all model/DTO image properties) — confirmed there was no prior art for resizing, no thumbnail helper, and no image viewer/lightbox anywhere in CWATIS as of 2026-07-10, when `ImageAlign` was first verified as a genuine API member (`C1.Win.C1FlexGrid.RowCol.ImageAlign`, inherited by `Column`; enum `C1.Win.C1FlexGrid.ImageAlignEnum`) against the local `C1.Win.C1FlexGrid.4.8.xml` doc for package version `4.8.20231.596` (ComponentOne WinForms Edition 2023 v1). `ContractTypesForm.cs` is the first form to actually apply it.

## Handling Click / Double-Click on a Cell

C1FlexGrid has **no** ready-made `CellClick(row, col)`/`CellDoubleClick(row, col)` event. You get the standard WinForms `MouseDown`/`Click`/`MouseDoubleClick`/`MouseMove` events (pixel `e.X`/`e.Y` only) and must resolve the cell yourself with `grid.HitTest(e.X, e.Y)`.

Cell-granular events do exist (`MouseEnterCell`/`MouseLeaveCell`/`MouseHoverCell`, `EnterCell`/`LeaveCell`, `CellButtonClick`, `CellChecked`) but none of them mean "user clicked this cell" — they're hover, keyboard/selection-driven current-cell change, the small in-cell edit-button glyph, and checkbox-toggle respectively. For actual click/double-click semantics, `HitTest` is the only path — confirmed against the local `C1.Win.C1FlexGrid.4.8.xml` doc and consistent with ~30 forms in this codebase that already use it. (`CellChecked` in particular looks like the obvious hook for a checkbox column but did not fire at all in a real test — see "Editing Cells" below for the event that actually works for checkbox columns.)

### `HitTestInfo` (returned by `grid.HitTest(x, y)`)

- `.Type` — `HitTestTypeEnum`: `Cell`, `ColumnHeader`, `RowHeader`, `ColumnResize`, `ColumnFreeze`, `RowResize`, `RowFreeze`, `Checkbox`, `EditButton`, `OutlineBar`, `OutlineTree`, `FilterIcon`, `ExpressionIcon`, `PinIcon`, `ColumnPickerIcon`, `None`.
- `.Row`, `.Column` (int — property is **`Column`**, not `Col` like elsewhere in the API) — both `-1` if the point isn't on a cell.
- `.Point`, `.X`, `.Y`, `.IsFooter`.

Fixed (header) rows/columns still report `Type == Cell` — check `hit.Row >= grid.Rows.Fixed` separately to exclude them.

### Pattern — double-click

Precedent: `CompletedWorksForm.cs` (`DetailGrid_MouseDoubleClick`), `ProductMoveRepListForm.cs` (`listOfReportsGrid_MouseDoubleClick`).

```csharp
private void grid_MouseDoubleClick(object sender, MouseEventArgs e)
{
    var hit = grid.HitTest(e.X, e.Y);

    if (hit.Type != C1.Win.C1FlexGrid.HitTestTypeEnum.Cell) return;
    if (hit.Row < grid.Rows.Fixed || hit.Row >= grid.Rows.Count) return;
    if (hit.Column != grid.Cols["SomeColumn"].Index) return;

    // handle it
}
```

### Pattern — single click (left/right)

Precedent: `GoodsTree4PriceCardsForm.cs` (`_detailGrid_MouseDown`).

```csharp
private void grid_MouseDown(object sender, MouseEventArgs e)
{
    var hit = grid.HitTest(e.X, e.Y);
    if (hit.Type != C1.Win.C1FlexGrid.HitTestTypeEnum.Cell) return;

    if (e.Button == MouseButtons.Left) { /* ... */ }
    else if (e.Button == MouseButtons.Right) { /* ... */ }
}
```

### Pattern — hover effect tied to a column

Precedent: `GoodsTree4PriceCardsForm.cs` (`_detailGrid_MouseMove`, hand cursor over a URL cell) — same `HitTest` call, wired to `MouseMove` instead of a click event.

## Per-cell tooltips: `grid.ToolTip` (a real `C1SuperTooltip`), not a custom tooltip

`SmartGrid` exposes a `ToolTip` property of type `C1.Win.C1SuperTooltip.C1SuperTooltip` directly (confirmed via reflection against `MicrosSmartGrid.V2.dll`) — this is a genuine ComponentOne tooltip control, not something to build by hand. There is **no per-column/per-cell tooltip property on `Column` or `C1FlexGridBase` itself** (checked via reflection — the only native tooltip-related members on the base grid are scroll tips, for the row-position hint shown while drag-scrolling, and error-icon tooltips via `ShowErrors`/`IDataErrorInfo` — neither is a general "static tooltip text for this column" mechanism). `SmartGrid.ToolTip` is what closes that gap.

Drive it from `MouseEnterCell` (`RowColEventHandler`, fires per cell as the mouse moves over the grid), reading the hovered cell's value via `grid.GetData(row, colName)` and calling `grid.ToolTip.SetToolTip(grid, text)` — pass `string.Empty` to clear it when the column/value doesn't warrant a tooltip. No `MouseLeaveCell` counterpart is needed in this codebase's precedent — entering a different cell just re-fires `MouseEnterCell` and overwrites the tooltip text (including clearing it via `string.Empty` for cells that shouldn't show one).

```csharp
private void grid_MouseEnterCell(object sender, C1.Win.C1FlexGrid.RowColEventArgs e)
{
    if (e.Row < grid.Rows.Fixed || e.Col != grid.Cols["SomeColumn"]?.Index) return;

    var value = grid.GetData(e.Row, "SomeColumn");
    grid.ToolTip.SetToolTip(grid, /* condition on value */ ? "Tooltip text" : string.Empty);
}
```

Precedent: `CompletedWorksForm.cs` (`MasterGrid_MouseEnterCell`/`DetailGrid_MouseEnterCell`, several columns including fixed-row header tooltips), `ContractTypesForm.cs` (`masterGrid_MouseEnterCell`, explains the `IsDefault` column's icon-only cell), and the same pattern repeated across `RepManagerSalesList.cs`, `ProductSalesRepListForm.cs`, `ProductMoveRepListForm.cs`, `DeptSellReportForm.cs`, `DepFactSellRepListForm.cs`, `CompanyClientReportForm.cs`.

## Gotcha: `Rows[row].DataSource` is not always the bound model directly

When a grid's `DataSource` is set via `C1DataCollectionBindingList` (the standing rule at the top of this doc), `grid.Rows[rowIndex].DataSource` does **not** reliably return the bound model (`TDto`) itself. For any row that hasn't been individually selected/focused yet, it can instead be `C1.DataCollection.BindingList.IDataCollectionBindingListItem` — an internal wrapper around the real item. A direct pattern match against the model type therefore silently fails for those rows:

```csharp
// WRONG — silently returns false/null for most rows, with no exception to signal the mismatch
if (!(grid.Rows[rowIndex].DataSource is TDto model)) return;
```

**Always go through `GetModelRow<T>(rowIndex)` / `GetModelSelectedRow<T>()`** (`SmartGridExtentions.cs`) instead — they already unwrap `IDataCollectionBindingListItem` (`(rawSource as IDataCollectionBindingListItem)?.Item as T`) before falling back to reading the row's live cell values:

```csharp
var model = grid.GetModelRow<TDto>(rowIndex);
if (model.SomeIdProperty == 0) return;   // not "== null" — see below
```

**Don't check the result for `null`** — `GetModelRow<T>`/`GetModelSelectedRow<T>` always return a `new T()` at minimum (their constraint is `where T : class, new()`), never `null`. Check a sentinel property instead (whatever's guaranteed non-default for a real row — usually the id, e.g. `model.ContractTypeId == 0`), matching the pattern already used elsewhere in this codebase (e.g. `ContractTypesForm.deleteCommand_Click`: `selected == null || selected.ContractTypeId == 0`).

**Why this matters more than it looks**: this bug doesn't throw, doesn't warn, and doesn't reliably reproduce on the *first* row you test (a row that's been individually selected via `AfterSelChange`-style handling tends to have its `DataSource` already resolved to the real model, so testing against "the currently selected row" can pass while the exact same code fails for every other row). It cost real debugging time on `ContractTypesForm`'s `"IsDefault"` checkbox column: three different event-firing theories (`BeforeEdit` vs `StartEdit` vs `CellChecked`, all discussed below) were investigated and two full reimplementations were done, before instrumented diagnostics proved every relevant event (`BeforeEdit`, `StartEdit`, `CellChecked`, `AfterEdit`) had been firing correctly all along — the actual defect was this single `Rows[row].DataSource as TDto` cast, silently returning `false` for any row other than the one already selected.

## Gotcha: `GetModelRow`/`GetModelSelectedRow` reconstruct a copy — read-only properties are silently skipped

`GetModelRow<T>(rowIndex)` / `GetModelSelectedRow<T>()` (`SmartGridExtentions.cs`) do not return the grid's live bound item. They build a fresh `new T()` and populate it property-by-property via reflection:

```csharp
foreach (PropertyInfo prop in typeof(T).GetProperties())
{
    if (grid.Cols.Contains(prop.Name) && prop.CanWrite)
        prop.SetValue(model, grid.Rows[rowIndex][prop.Name]);   // from the grid cell
    else if (!grid.Cols.Contains(prop.Name) && prop.CanWrite)
        prop.SetValue(model, prop.GetValue(source));            // from the unwrapped bound object
}
```

Consequences:

- **A property with no grid column still comes through.** It's copied straight from the row's real bound object — the same `T` instance originally handed to `C1DataCollectionBindingList`. The grid never trims objects down to what's displayed, so "not shown as a column" does not mean "lost."
- **A read-only property (no public setter) is silently skipped either way** — both branches gate on `prop.CanWrite`, so a computed/get-only property stays at its `new T()` default regardless of whether it has a column. No exception, no warning.
- **The result is a copy, not a reference.** Mutating the returned object does not mutate the row's actual bound item. To push edits back into the grid's bound data, use `UpdateSelectedGridRow<T>` instead (which also stashes the model on `Rows[grid.Row].UserData`).
- **If the row's `DataSource` can't be unwrapped to `T` at all** (`source == null` — e.g. a fixed/header row, or a row not actually bound to `T`), you get a fully blank `new T()`, not a partially-filled one.

## Editing Cells: `StartEdit` vs `BeforeEdit` vs `AfterEdit`

C1FlexGrid (and `SmartGrid`, which extends it directly — confirmed via reflection against `MicrosSmartGrid.V2.dll`: `SmartGrid`'s base type is `C1.Win.C1FlexGrid.C1FlexGrid`, and it declares no override of any edit-related method) exposes three cell-edit events that look related but are **not interchangeable**:

- **`BeforeEdit`** fires far more often than the name suggests. Per the C1FlexGrid docs: "This event doesn't fire only when the grid is about to enter edit mode. It also fires while painting cells, in order to support drawing the editor's drop-down buttons." Prefer `StartEdit` for guard/cancel logic that must run exactly once per genuine edit attempt.
- **`StartEdit`** fires *only* when the grid is actually about to enter edit mode. This is the correct event for validation/guard logic that needs to cancel an edit before it starts (`e.Cancel = true`) — confirmed firing reliably (once per click, decompiled and instrumented) even for checkbox columns under `DrawMode=OwnerDraw`, once the `Rows[row].DataSource` gotcha above is avoided.
- **`AfterEdit`** fires once an edit has been committed (does not fire if the user cancels with ESC). By this point the cell's bound model property already reflects the new value — use this event to persist it.
- **`CellChecked`** also genuinely fires for a checkbox toggle (instrumented: confirmed alongside `AfterEdit`) — but only *after* the value has already changed, same timing as `AfterEdit`, so prefer `StartEdit`/`AfterEdit` for symmetry (one event to guard, one to persist) rather than mixing in a third event that can't cancel anything useful by the time it runs.

### Pattern — guard + persist

```csharp
private void grid_StartEdit(object sender, C1.Win.C1FlexGrid.RowColEventArgs e)
{
    if (e.Col != grid.Cols["SomeColumn"]?.Index) return;
    if (e.Row < grid.Rows.Fixed || e.Row >= grid.Rows.Count) return;

    var model = grid.GetModelRow<TModel>(e.Row);   // NOT Rows[e.Row].DataSource as TModel - see gotcha above
    if (model.SomeIdProperty == 0) return;

    if (/* not allowed */)
    {
        MessageBox.Show("...");
        e.Cancel = true;
    }
}

private async void grid_AfterEdit(object sender, C1.Win.C1FlexGrid.RowColEventArgs e)
{
    if (e.Col != grid.Cols["SomeColumn"]?.Index) return;
    if (e.Row < grid.Rows.Fixed || e.Row >= grid.Rows.Count) return;

    // model already reflects the new (committed) value here - StartEdit already rejected
    // anything that shouldn't have gotten this far.
    var model = grid.GetModelRow<TModel>(e.Row);
    if (model.SomeIdProperty == 0) return;

    await (OnUpdate?.Invoke(model) ?? Task.CompletedTask);
}
```

Precedent: `userpermissionsui/RolesManagerForm.cs` (`RolesGrid_StartEdit`/`RolesGrid_AfterEdit`, and `RoleRightsGrid`'s `_roleRightsGrid_OnStartEdit`/`_roleRightsGrid_OnAfterEdit`). **Correction:** an earlier version of this doc also cited `ContractTypesForm.cs` here as a `StartEdit`/`AfterEdit` example — its actual current code (`masterGrid_BeforeEdit`/`masterGrid_AfterEdit`) uses `BeforeEdit`, not `StartEdit` (confirmed by reading the file directly, 2026-08-18). Whether that's a deliberate choice or drift from this doc's own recommendation above is unknown - don't cite it as `StartEdit` precedent without re-checking the file.

**Resolved — the `Cols[e.Col].Name` theory below was a dead end; see the `Footers`/aggregate gotcha further down instead.** A `SubdivisionsForm` implementation hit a real, repeatable `ArgumentOutOfRangeException` immediately after every committed cell edit. Three rounds of fixes targeting this form's own event handlers — guarding `e.Row`, then also guarding `e.Col` around `Cols[e.Col]`, then rewriting to avoid `Cols[e.Col]` entirely (switching `StartEdit`→`BeforeEdit` and comparing `e.Col` against `Cols["Name"]?.Index` instead, matching `ContractTypesForm`) — **produced the identical error every time.** A captured stack trace (via a temporary second `Application.ThreadException`/`AppDomain.UnhandledException` subscriber logging `ex.ToString()`, since the app's real handler's `CatchHelper` lives in an external package with no source in this repo) proved why: the throw was entirely inside `C1.Win.C1FlexGrid.C1FlexGridBase.DrawRow` → `RowCollection.get_Item` → `ArrayList.get_Item`, reached via a native `WM_PAINT` → `OnPaint`. **Nothing from the form's own C# code appears anywhere in that stack** — no amount of fixing `SubdivisionsForm`'s handlers could ever have helped, since the crash was never in them. The `Cols[e.Col].Name`-avoidance and `BeforeEdit` changes were kept anyway (harmless, and `BeforeEdit` does match the one other proven-working precedent in this codebase) but were not what fixed this.

## Gotcha: with a `Footers` aggregate present, committing an edit on the grid's *last* row makes `DrawRow` throw `ArgumentOutOfRangeException`

### Symptom

On a `SmartGrid` that has a `Footers.Descriptions` aggregate configured (e.g. a `Count` total), committing a cell edit (Enter/Tab/click-away) specifically on the **last data row** throws, reliably and repeatably (confirmed by retrying the exact same row twice — not a one-time/cold-start effect) — client-side as `System.ArgumentOutOfRangeException: Индекс за пределами диапазона...Имя параметра: index`, surfaced by the app's global unhandled-exception handler, not by any local `try`/`catch` (see below for why). The grid also visually fails to paint from this point on (diagonal red-hatch pattern instead of rows) — the same underlying paint failure repeating on every subsequent repaint, not a separate issue.

**Editing any row other than the last one is fine, footer and all** — the trigger is specifically last-row-edit-with-a-footer-present, confirmed via instrumented row-count logging (`Rows.Fixed` + the bound list's real `Count` matched `Rows.Count` exactly whether or not the crash happened - the crash correlates with *which row* was edited, not with any row-count mismatch).

**This is very likely a general defect in this `MicrosSmartGrid.V2`/`C1FlexGrid` version combination, not something specific to `SubdivisionsForm`.** An earlier version of this entry claimed `ContractTypesForm.cs`'s `masterGrid` — which has this *exact* configuration (`Footers.Fixed = true`, `Count` aggregate on `Column = 1`, `OwnerDraw`, `BeforeEdit`/`AfterEdit` editing) — "doesn't crash", based on it being cited as a working precedent elsewhere in this doc. **That claim was never actually tested and was wrong**: editing `ContractTypesForm`'s own last row reproduces the identical crash (confirmed directly by the user, 2026-08-18). That's the second time in this same investigation that trusting "cited as precedent" over actually testing produced a wrong conclusion (the first was the `StartEdit`-vs-`BeforeEdit` citation above) — don't repeat the pattern a third time: a form appearing elsewhere in this doc as a "working precedent" means its *general approach* was copied successfully, not that every edge case of its current behavior has been verified. If a grid has `Footers` + an aggregate + is editable, assume it can hit this until someone actually tests its own last row, regardless of what any doc (including this one) says about it.

Full stack trace (captured via a temporary second `Application.ThreadException` subscriber, since this app's real global handler's `CatchHelper` lives in an external package with no source in the repo):

```
System.ArgumentOutOfRangeException: Индекс за пределами диапазона...
Имя параметра: index
   в System.Collections.ArrayList.get_Item(Int32 index)
   в C1.Win.C1FlexGrid.RowCollection.get_Item(Int32 index)
   в C1.Win.C1FlexGrid.C1FlexGridBase.DrawRow(PaintEventArgs e, Int32 row, DrawColorMode colorMode)
   в C1.Win.C1FlexGrid.C1FlexGridBase.a(PaintEventArgs A_0, DrawColorMode A_1)
   в C1.Win.C1FlexGrid.C1FlexGridBase.OnPaint(PaintEventArgs e)
   в System.Windows.Forms.Control.PaintWithErrorHandling(PaintEventArgs e, Int16 layer)
   в System.Windows.Forms.Control.WmPaint(Message& m)
   ...WndProc (native WM_PAINT)
```

**The error message points at the wrong component entirely.** Nothing from the form's own C# code — no event handler, no repository call, no presenter method — appears anywhere in this stack. The exception is thrown from deep inside C1FlexGrid's own `OnPaint`, triggered by a native repaint message, not synchronously from any application code path. A local `try`/`catch` around the `AfterEdit` handler (even one wrapping an `await`ed save) cannot catch this — it's a different call stack entirely, which is also why it survived three separate rounds of fixes aimed at that handler's own `Row`/`Col` bounds-checking and indexing (none of which were wrong per se, just irrelevant to this crash).

### Root cause

**Not fully diagnosed at the C1FlexGrid-internals level, and closed without a full explanation** (no source access, public documentation/search turned up nothing specific to this combination, and two targeted mitigation attempts both failed - see below). What's confirmed empirically: (1) removing the grid's `Footers` aggregate entirely eliminates the crash, every time; (2) instrumenting `AfterEdit` to log `e.Row`/`Rows.Fixed`/`Rows.Count`/bound-list-`Count` showed the crash occurred exactly when `e.Row == Rows.Count - 1` (the last row) and did not occur for any other row; (3) an initial "cold start" theory (footer's internal slot not yet allocated on the *first* post-load edit) was ruled out by retrying the same last row a second time and getting the identical crash again.

Two follow-up theories about *why*, both tested and **both failed to prevent the crash while keeping the footer**:

- **Native "move to next row after Enter" landing on the footer.** Theory: `Footers.Fixed = true` positions the footer immediately after the last data row, and C1FlexGrid's own post-commit row-advance tries to select it as if it were a normal row. Tested fix: set `smartGrid1.Row = e.Row` as the first line of `AfterEdit`, synchronously, before any `await` - to snap the selection back before any repaint could happen on the bad position. **Did not stop the crash.**
- **`C1DataCollectionBindingList.RemoveAt` on the last row triggering the grid's own "pick a new current row" logic.** Theory (this codebase already does `RemoveAt`+`Insert` to replace a bound item in place - see the "Update" gotcha above - and separately does a plain `RemoveAt` for delete): `RemoveAt` fires `ListChanged`, and while the list is momentarily shorter, if the removed item was the currently-selected row, the grid needs to auto-pick a new current row - landing on the footer if that row was last. Tested fix: before every `RemoveAt` that might remove the current row, move `smartGrid1.Row` off it first (to the previous row), then restore it to the correct position afterward - applied to all three of this form's `RemoveAt` call sites (`SetLocalId`, `RemoveRow`, and `UpdateField`, which also covers `AfterEdit`'s revert-on-failure path since it's shared). **Did not stop the crash either.**

Given two independently-plausible, directly-targeted mitigations both failed, the true mechanism is something else inside C1FlexGrid/`MicrosSmartGrid.V2` that wasn't identified. **Don't re-attempt either of the two theories above without new evidence (e.g. a debugger break-on-exception session, since the stack trace by itself wasn't enough to distinguish between them or find the real cause).**

### Fix

**Closed as: remove the footer.** No workaround for editing the last row while keeping the footer was found, after three separate attempts (see Root cause). Given the trigger is "the user edits the actual last row" — an ordinary, unavoidable action, not an edge case that can be designed around in the UI — the fix is to remove the `Footers.Descriptions`/`AggregateDefinition` setup from the grid's `InitializeComponent()` (Designer.cs) entirely rather than leave a latent, guaranteed-eventually-reproducible crash in a shipped feature. This is the only confirmed-working fix; treat it as final unless someone finds the actual C1-internals mechanism. If a total count is genuinely needed, compute and display it separately (e.g. a label outside the grid, driven by the same data the grid is bound to) — that sidesteps this code path entirely rather than trying to keep `Footers` working around it.

```csharp
// Designer.cs - remove:
aggregateDefinition1.Aggregate = C1.Win.C1FlexGrid.AggregateEnum.Count;
aggregateDefinition1.Caption = "Всего: ";
aggregateDefinition1.Column = 1;
footerDescription1.Aggregates.Add(aggregateDefinition1);
this.smartGrid1.Footers.Descriptions.Add(footerDescription1);
this.smartGrid1.Footers.Fixed = true;
// ...and the now-unused footerDescription1/aggregateDefinition1 local declarations further up.
```

### Checklist

- If a footer-having grid crashes with `ArgumentOutOfRangeException` after a committed edit, check whether it's specifically the *last* row — that's the actual, narrow trigger, confirmed on two different forms (`SubdivisionsForm`, `ContractTypesForm`) now. It's not on-load, not on selection change, not row-count-mismatch related.
- **Treat "works fine" claims about other forms as unverified until someone actually edits that form's own last row.** This entry went through three rounds of a form being wrongly assumed safe just because it's cited elsewhere as a working precedent (first the `StartEdit` claim, then this same `ContractTypesForm` being assumed not to crash). "Cited as precedent" means the general approach was copied successfully; it says nothing about whether every specific edge case — including this one — was ever exercised on that form.
- Any other editable grid in this codebase with a `Footers` aggregate configured (grep `Footers.Descriptions.Add` alongside `BeforeEdit`/`StartEdit`/`AfterEdit` in the same Designer.cs/View pair) should be assumed to have this same latent crash until it's actually tested, not assumed safe by default. `ContractTypesForm.cs` is confirmed affected too (task flagged 2026-08-18 to fix it there, same way).
- **Don't re-attempt fixing this by touching the form's own row-selection code** — both "reset selection to the edited row before any repaint" and "move selection off the row before any `RemoveAt` that might remove it" were tried and neither stopped the crash. The only confirmed-working fix is removing the `Footers` aggregate entirely (see Fix above).
- When a crash's stack trace contains **zero** application code frames, that's a strong signal the fix belongs in what's *configured* on the control, not in a handler that merely triggered it - but here even correctly identifying the configuration culprit (`Footers`) and testing two specific, plausible mechanisms wasn't enough to find a fix that keeps it. Know when to stop: after three failed attempts, removing the feature was the right call over a fourth guess.
- If the app's own global exception handler doesn't show a usable stack trace (routes through an external package with no source, as `CatchHelper` does here), add a temporary second subscriber to `Application.ThreadException`/`AppDomain.UnhandledException` that shows `ex.ToString()` directly — multicast events support multiple subscribers, so this doesn't require touching or removing the existing handler. Remove it once diagnosed.

Found in CWATIS, 2026-08-18, while wiring inline grid-cell editing for `SubdivisionsForm` (`Code`/`Name` columns). Closed the same day after three failed mitigation attempts - footer removed, not fixed.

### Checkbox / Boolean columns (`Style:"DataType:System.Boolean;..."`)

A column styled with `DataType:System.Boolean` renders a checkbox glyph and toggles on click, going through the exact same `StartEdit`/`AfterEdit` lifecycle as a text column — no special handling needed beyond the guard + persist pattern above (and the `GetModelRow<T>` gotcha, which applies here just as much as to any other column). `AllowEditing` at the column level (default `true` unless set otherwise) controls whether the checkbox responds to clicks at all; there's no need to set it to `False` and drive the checkbox by hand via `MouseDown`/`HitTest` — that was tried as a workaround while the real bug (the `DataSource`-casting gotcha above) was still unidentified, and turned out to be unnecessary once the actual defect was fixed.

Working precedent for a guarded checkbox column: `userpermissionsui/RolesManagerForm.cs` — `RolesGrid` (also a `MicrosSmartGrid.SmartGrid`, also `DrawMode=OwnerDraw`) has a `DataType:System.Boolean` column literally named `"IsDefault"`, guarded in `RolesGrid_StartEdit`. Its own code comment states the `BeforeEdit`-vs-`StartEdit` reasoning directly: "Почему проверка происходит здесь а не в BeforeEdit? Потому что BeforeEdit вызывается не только при пользовательском редактировании, но и при рендеринге." **Correction:** this used to also cite `ContractTypesForm.cs`'s `masterGrid_StartEdit`/`masterGrid_AfterEdit` as following the same pattern for its own `"IsDefault"` column (single-default-type invariant, confirmation on replacing an existing default) — per the same `StartEdit`-citation error caught elsewhere in this doc, `ContractTypesForm.cs`'s Designer.cs only wires `masterGrid_BeforeEdit`/`masterGrid_AfterEdit`, no `StartEdit` at all. Whether the "IsDefault" enforcement logic itself still exists in that `BeforeEdit`/`AfterEdit` pair, or exists at all, wasn't re-verified — check the actual current file before citing this as precedent for anything.

## Gotcha: yellow "selected" row visible in the VS designer (`HighLight` default is `Always`)

C1FlexGrid's `HighLight` property defaults to `HighLightEnum.Always`, which paints the current row/cell with the `Highlight` style **even when the grid has no focus — including on the VS designer surface**. In CWATIS the shared `StyleInfo` defines `Highlight{BackColor:255, 255, 180;...}` (pale yellow), so a grid dropped on a form/UserControl shows a phantom yellow first data row in the designer.

The same flag controls both surfaces (the designer host is just an unfocused grid), so a single value can't give "clean designer + always-visible runtime selection". **Fix is two-step**: serialize `WithFocus` in the Designer.cs, then restore `Always` at runtime in the constructor, guarded by the design-time flag (see `SolvedProblems/winforms-usercontrol-design-time-safety.md` for `_isDesignTime`):

```csharp
// Designer.cs — keeps the design surface clean:
this._grid.HighLight = C1.Win.C1FlexGrid.HighLightEnum.WithFocus;

// ctor, after InitializeComponent() — runtime behaves like every other grid
// (selected row stays yellow even when the grid is unfocused):
if (!_isDesignTime)
    _grid.HighLight = C1.Win.C1FlexGrid.HighLightEnum.Always;
```

Plain `WithFocus` alone is only correct when the unfocused-selection highlight is genuinely unwanted at runtime too (`Forms/Telegram/Onboarding/View/OnboardingsUserControl.Designer.cs`, `UserOnboardingsUserControl.Designer.cs`). The two-step pattern's precedent: the ДС tab grids `ContractDSListControl`/`ContractDSLinesTabControl` (2026-07-27) — users expect the auto-selected first row to be visibly highlighted right after load, matching the app's other grids.

## Gotcha: designer serializes `SelectedRows` state junk

The VS designer serializes `SmartGrid.SelectedRows` into the Designer.cs (`this._grid.SelectedRows = ((List<int>)(resources.GetObject("_grid.SelectedRows")));`) with a binary blob in the `.resx` — this is captured *designer-session selection state*, not configuration. It pre-bakes a selection into every new instance. When found (typically after copying a grid between forms), delete both the Designer.cs line and the `<data name="_grid.SelectedRows" ...>` element in the `.resx`.
