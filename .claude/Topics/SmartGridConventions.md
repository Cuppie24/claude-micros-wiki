# SmartGrid Conventions (CWATIS)

Conventions for working with `MicrosSmartGrid.SmartGrid` (and `TreeFlexGrid`) in the CWATIS WinForms client. See also `SmartGridExtentions.cs` (`CWATIS\Extentions\SmartGridExtentions.cs`) for the extension methods referenced below, and the project's `SmartGridReference.md` for general grid feature documentation (headers, context menu, tooltips, tree building, etc. — legacy/English, do not extend it; the active wiki is the `CwatisWiki` vault).

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

### Rule: a single-row save patches the bound list — no re-fetch

This is **the standard way** to show a saved row in **every** WinForms form and grid, not something you opt into. When the user saves (creates, edits or deletes) **one row**, and the save logic changes nothing except that row, the view puts the client-side model into the live `C1DataCollectionBindingList` (Update: `RemoveAt` + `Insert`, see below). Don't make a second DB call to re-read the row or the list, don't call the form's `OnRefresh`/`LoadXAsync`, and don't reassign `DataSource`.

- **Wrong:** `await _repository.SaveAsync(model); await LoadDocumentsAsync();` — a full re-list just to show one row the client already has. It costs a server round trip, rebinds the grid, can lose the selection and scroll position, and fires selection events again.
- **Wrong:** writing the new values cell by cell (`grid[row, "Col"] = model.Col` for each column, as the older `CompletedWorksForm.UpdateDetailRow` does). The bound item stays stale, so the next sort, `GetModelRow<T>` or client-side filter shows the old values, and every new column needs another line.
- **Right:** the presenter saves, and only after success calls `_view.UpdateXRow(model)`. The view finds the item's index in the binding list (unwrapping `IDataCollectionBindingListItem`), calls `RemoveAt` + `Insert` at that index, and selects the row again.

**When a real re-fetch IS required** (refresh the affected data instead of patching): the save has server-side side effects that the client can't reproduce. Examples: a procedure or trigger that recalculates totals, statuses or amounts on this row or on other rows, a header whose total depends on its lines, a renumbering, or a cascade into other rows or other grids. Also re-fetch when the grid shows server-derived display columns (joined names, computed fields) that the model you have doesn't fill and you can't fill cheaply on the client. Fill a joined name from data the form already holds (e.g. the `_warehouses` list a combo is bound to) when you can — that still needs no DB call. Re-fetch only when you really can't.

Precedents: `SubdivisionsForm.cs` (`InsertRow`/`SetLocalId`/`RemoveRow`), `WhMovementControl.UpdateDocumentRow` (Warehouse → Movement; saving a document header).

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

Sometimes the backend call already returns enough to build it client-side (a create dialog already collected `Code`/`Name`; the new id is the only unknown, and the insert procedure returns it — just merge). Sometimes only *part* of the row changed and the new value is already known deterministically without asking the backend anything at all (e.g. "remove from local" always means "this FK becomes null" — no round trip needed to learn that). Don't guess a stored procedure's result-set shape to avoid an extra call — see `../SolvedProblems/micros-libraries/cwatis-stored-procedures-not-in-repo.md` (procedures aren't in the repo and can't be checked by reading code); confirm the actual column name with whoever owns the DB first.

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

**Tables outside `cwatis`:** the two-argument overload always opens the `cwatis` log (`log_db.rec_info_cwatis`). Each schema's
`add_rec_info` writes to its own log table, and `global_db.add_rec_info` writes to **`log_db.rec_info_refers`** (not
`rec_info_global`, which stays empty — verified on mars 2026-11-10). For a `global_db` table call
`Main.RecInfoView(id, table, "")`: an empty `MainDBName` makes `RecInfoForm` open the refers log. Passing `"refers"` explicitly
works too, but adds a second item with the same code to the form's database list. The log only shows something if the
procedures that write the table call `add_rec_info`; `CallService.DeleteLines` and plain table writes don't.

**Footer rows:** check that the current row is a data row before reading the model — reading a model on a footer row throws
(see the footer gotcha below). Warehouse precedent that does both: `Features/Warehouse/Shared/View/WhRecordInfo.Attach(grid,
schema, table, getCurrentId)` (2026-11-10).

### Getting the id

- If the grid has a real bound column for the key (e.g. `"ContractId"`, `"ActId"`, `"PreContrId"`), index the grid directly: `(int)grid[grid.Row, "ColumnName"]`.
- If the grid has **no bound id column** (`AutoGenerateColumns=false` and the id property isn't in `ColumnInfo`, or `IdName` is `null`), do **not** guess a column name (e.g. `"Id"`) — indexing by a name that isn't in `ColumnInfo` throws. Instead extract the id from the row's bound DTO:
  - Whole-model extraction: `grid.GetModelSelectedRow<TDto>()`, then read `.YourIdProperty`. Guard against a blank/default model when no row is selected (e.g. `selected == null || selected.YourIdProperty == 0`).
  - Or a direct cast when only the id is needed: `(grid.Rows[grid.Row].DataSource as TDto)?.YourIdProperty` (precedent: `TemplDocsViewForm.cs`).

## Empty Grid Before the First Load: Collapse to Fixed Rows

A grid dropped in the designer is saved with a placeholder row count (`Rows.Count = 51`, etc.). On form open the user would see ~50 empty rows flash for the moment between the form appearing and the async fetch finishing, and then the real 3 rows replace them (visible flicker).

**Rule:** on form load, before the first data fetch, collapse every grid to its fixed rows (plus fixed footer rows, if any). Do it right after `InitializeComponent()` in the form constructor — it only touches `Rows`, so it is safe there (unlike `Cols[...]` access, see `../SolvedProblems/winforms/winforms-child-control-setup-constructor-vs-load.md`):

```csharp
public MyForm()
{
    InitializeComponent();
    ClearGrid(masterGrid);
    ClearGrid(detailGrid);
    ...
}

private static void ClearGrid(SmartGrid grid)
{
    grid.Rows.Count = grid.Rows.Fixed + grid.Footers.Descriptions.Count;
}
```

`Footers.Descriptions.Count` is the number of footer rows (0 when the grid has none); leaving it out would delete the footer row. Binding data later through `DataSource = new C1DataCollectionBindingList(...)` resets the row count anyway, so nothing else needs to change.

Precedent: `Forms/WhInventory/View/WhInventoryActsForm.cs` (`ClearGrid`).

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

**GlobalForms note:** GlobalForms' own `SmartGrid_Extentions.cs` doesn't have a `FindModel` counterpart yet (only `GetModelRow`/`GetModelSelectedRow` - see the `../SolvedProblems/micros-libraries/globalforms-smartgrid-missing-model-unwrap-helper.md` gotcha). For a single call site there, a plain inline `_allData.FindIndex(x => x.Id == focusedId)` + `grid.Row = grid.Rows.Fixed + index` is fine - the "don't hand-roll the scan loop" warning above is about not reimplementing a helper that already exists *in the same codebase*; it just doesn't exist to reuse yet in GlobalForms. Port `FindModel` over only once a second call site actually needs it.

**Gotcha: `BeginUpdate()`/`EndUpdate()` around a `DataSource` reassignment does NOT by itself preserve the focused row.** They only suspend repaint/layout while the new `C1DataCollectionBindingList` is swapped in - `grid.Row` still resets (typically to the first data row) once a brand-new `DataSource` is assigned, Begin/EndUpdate or not. Confirmed in GlobalForms: `SubdivisionCategoriesForm.RebindGrid` already wrapped its `DataSource = new C1DataCollectionBindingList(...)` in `BeginUpdate()/EndUpdate()` from day one (for the usual flicker/perf reason), yet the focused row still jumped to the first row on every refresh, because nothing ever restored `grid.Row` afterward. Fixed with this section's main technique: capture the focused row's id *before* the rebind, then relocate and restore `grid.Row` *after* `EndUpdate()`. Begin/EndUpdate and the id-based restore are two independent, both-needed pieces, not alternatives to each other.

Found in GlobalForms, 2026-08-26, in `SubdivisionCategoriesForm.RebindGrid` (shared by the create/delete/refresh and revert-on-invalid-edit paths for the subdivision-categories grid) and, once known to look for it, the same pre-existing pattern in the sibling `CategorizedSubdivisionsForm.ApplyTypeFilter` - there the previous code unconditionally forced `grid.Row = grid.Rows.Fixed` after every rebind (add/delete/approve **and** an actual category-filter change), so approving/unapproving a row always bounced focus to the top. Fixed the same way, with one addition needed there specifically: since `ApplyTypeFilter` is also the handler for a genuine filter-category change (where the previously-focused row usually won't exist in the new filtered set at all), the id-restore has to fall back to the first row when the id isn't found, rather than assuming it's always present like the simpler `RebindGrid` case.

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

**Standing rule: every cell image must be center-aligned and scaled to fit the cell.** Set the column's `ImageAlign` to `C1.Win.C1FlexGrid.ImageAlignEnum.Scale` once at grid setup (constructor, right after `InitializeComponent()` — `Cols["Name"]` only resolves once `ColumnInfo` has been parsed, which happens as part of `InitializeComponent()`). `Scale` both centers the image and resizes it to fit the cell's current bounds while preserving aspect ratio (confirmed against the local `C1.Win.C1FlexGrid.4.8.xml` doc: "Image is scaled to fit the maximum area within the cell while preserving the original aspect ratio" — `ImageAlignEnum` is a plain enum, not flags, so there is no separate "center" setting to combine with it; `Scale` already implies centered). Use `ImageAlignEnum.Stretch` only if aspect-ratio distortion is acceptable — normally it isn't. When the image must have a **specific pixel size** rather than "as big as the cell", `Scale` is the wrong tool — see "Pattern — exact pixel size, centered" below.

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

### Pattern — exact pixel size, centered (e.g. "always 16×16 in the middle of the cell")

`ImageAlign` has **no size setting**. `Scale` sizes the image *to the cell* (a 16px icon in a 40px-tall row becomes 40px; in a short row it shrinks), and every positional value (`CenterCenter`, `LeftCenter`, …) draws the image at its **natural pixel size**, clipped if it's bigger than the cell. So "a specific size" = control the bitmap's size yourself, then let `CenterCenter` center it. Two ways:

**A (preferred) — resize the bitmap once, align with `CenterCenter`.** Build the sized bitmap once and cache it in a static field (never inside `OwnerDrawCell`, which fires on every repaint and during autosize):

```csharp
// grid setup (Load, or right after InitializeComponent()) — or in the Designer column Style string: "ImageAlign:CenterCenter;"
grid.Cols["HasPCDiff"].ImageAlign = C1.Win.C1FlexGrid.ImageAlignEnum.CenterCenter;

private static Image _diffIcon16;
private static Image DiffIcon16 => _diffIcon16 ?? (_diffIcon16 = Resize(Properties.Resources.check7, 16, 16));

private static Image Resize(Image src, int w, int h)
{
    var bmp = new Bitmap(w, h);
    using (var g = Graphics.FromImage(bmp))
    {
        g.InterpolationMode = System.Drawing.Drawing2D.InterpolationMode.HighQualityBicubic;
        g.PixelOffsetMode = System.Drawing.Drawing2D.PixelOffsetMode.HighQuality;
        g.DrawImage(src, new Rectangle(0, 0, w, h));
    }
    return bmp;
}

private void grid_OwnerDrawCell(object sender, OwnerDrawCellEventArgs e)
{
    if (e.Row < grid.Rows.Fixed) return;
    if (e.Col == grid.Cols["HasPCDiff"]?.Index)
    {
        e.Text = string.Empty;
        e.Image = flag ? DiffIcon16 : null;
    }
}
```

The bitmap must keep the target size even if the row/column is later made larger — that's the point; with `Scale` it wouldn't. Make sure the row is at least as tall as the icon (+ a couple of px), otherwise `CenterCenter` clips it.

**B — draw it yourself** (when the size must depend on the cell, e.g. "16px but never more than row height − 4"). Let the grid paint background + border (selection/alternating colors stay correct), then draw the image into a centered rect:

```csharp
if (e.Col == grid.Cols["HasPCDiff"]?.Index && flag)
{
    e.Text = string.Empty;
    e.Image = null;
    e.DrawCell(DrawCellFlags.Background | DrawCellFlags.Border); // also sets e.Handled = true
    int size = Math.Min(16, e.Bounds.Height - 4);
    var r = new Rectangle(e.Bounds.X + (e.Bounds.Width - size) / 2,
                          e.Bounds.Y + (e.Bounds.Height - size) / 2, size, size);
    e.Graphics.DrawImage(DiffIcon, r);
}
```

Skip `DrawCellFlags.Content` (that would draw the text/natural-size image too). `DrawCell(...)` sets `e.Handled = true` itself (confirmed in the local `C1.Win.C1FlexGrid.4.8.xml`), so the grid won't repaint over your image. If you only modify `e.Style`/`e.Text`/`e.Image` and don't call `DrawCell`, don't set `Handled` — the grid paints normally.

Precedent for `ImageAlign:CenterCenter` set via the Designer column style: `OrderTabControl.resx` (`HasPC`/`HasPL` columns). Precedent for a cached icon property: `GridOwnerDraw.DiffIcon` in `CalcImportPrices\View\SubVIew\GridEditHelpers.cs` (natural size, no resize). API members verified against the local `C1.Win.C1FlexGrid.4.8.xml` (`4.8.20231.596`) on 2026-10-19 — not yet exercised in a running form.

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

### `ValidateEdit`: checking the just-typed value and aborting/reverting before it commits

None of the three events above can validate *the value the user just typed* before it lands in the model: `StartEdit`/`BeforeEdit` fire before typing starts (only the old value exists yet), and `AfterEdit` fires only once the new value is already committed to the cell and the bound model — too late to abort without a manual revert. For "check what was just typed, reject it if invalid," use **`ValidateEdit`** (`C1.Win.C1FlexGrid.ValidateEditEventArgs`) instead. Per the local `C1.Win.C1FlexGrid.4.8.xml` doc, it "fires before the control exits cell edit mode, while the editor is still active" — the new value hasn't been applied to the grid or model yet, so rejecting here needs no manual revert.

Read the not-yet-committed value from **`grid.Editor.Text`** (the grid cell / bound model still hold the *old* value at this point — that's the whole reason this event exists, distinct from `AfterEdit`). Two ways to reject, both real precedent in this codebase:

- **`e.Cancel = true`** — the editor stays open with the invalid text still showing; the user must correct it or press Esc to actually cancel. Precedent: `GoodsTree4PriceListForm.cs` (`_detailGrid_ValidateEdit`/`_masterGrid_ValidateEdit`, 0–100% range checks on percent columns), `PricingParametersEditForm.cs` (`_detailGrid_ValidateEdit`, same range-check pattern), `TgMembersForm.cs` (`membersGrid_ValidateEdit`, `PhoneCw` column).
- **`e.Cancel = true` plus `grid.FinishEditing(true)`** — forces the grid out of edit mode immediately *and* restores the original value in the same call. Confirmed in the XML doc: `FinishEditing(true)` "the original cell value is restored and the grid is guaranteed to leave the edit mode" (the parameterless overload / `FinishEditing(false)` instead tries to apply the edit and re-fires `ValidateEdit`/`LeaveEdit`/`AfterEdit` as usual, staying in edit mode if validation fails). Use this when the invalid entry shouldn't keep sitting in the editor waiting for the user to notice it. Precedent: `TgMembersForm.cs` (`membersGrid_ValidateEdit`, `CompanyUrlCw` column).

```csharp
private void grid_ValidateEdit(object sender, C1.Win.C1FlexGrid.ValidateEditEventArgs e)
{
    if (e.Row < grid.Rows.Fixed || e.Col != grid.Cols["SomeColumn"]?.Index) return;

    if (!double.TryParse(grid.Editor.Text, out var value) || value < 0 || value >= 100)
    {
        MyMessageBox.Show("Редактирование", "Значение должно быть в пределах от 0 до 100.", "Warning");
        e.Cancel = true;
        // grid.FinishEditing(true); // add this line to also force an immediate revert instead of leaving the editor open for correction
    }
}
```

Wire it in Designer.cs like any other grid event: `this.grid.ValidateEdit += new C1.Win.C1FlexGrid.ValidateEditEventHandler(this.grid_ValidateEdit);`.

**If the check itself needs an async/server round trip** (e.g. a uniqueness check against the DB), `ValidateEdit` can't await one — the editor can't be held open across an `async` gap. In that case let the edit commit, validate in `AfterEdit`, and revert manually if it turns out invalid (restore the previous value onto the row/model yourself — there is no framework-level "undo commit" once `AfterEdit` has fired). No CWATIS precedent for this half exists in this codebase yet; capture the pre-edit value in `StartEdit` if this path needs it later.

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

**Recurrence 2026-11-16 (CWATIS invoices):** same stack after saving an invoice header that sits in the **last** row of `gridDocs`
(footer with a `Count` on column 1) — the save replaces the row in place (`RemoveAt` + `Insert`), no cell edit involved. Workaround
in CWATIS warehouse forms: `Features/Warehouse/Shared/View/WhGridLastRow.IsLast` — the view's in-place replace methods
(`WhMovementControl.UpdateDocumentRow`, `WhStockGoodsForm.UpdateCards`, `WhGoodOwnersForm.ReplaceOwners`) return `false` without
touching the list when a target is the last data row, and the presenter does a full reload instead. A hack, not a fix: the
mechanism is still unknown, removals / appends and cell edits on the last row are not covered.

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

### Update 2026-11-10: alternative fix that keeps the footer — full rebind instead of editing the bound list (user report, not yet verified)

The user reports that the crash happens **only when the bound `C1DataCollectionBindingList` is modified** (`RemoveAt` + `Insert`
"replace in place", `Add`, `RemoveAt`) after the edit — which is exactly what `SubdivisionsForm` did on save. Pattern built on that:
keep the footer, let `AfterEdit` only save, then re-read the data and assign a **new** `C1DataCollectionBindingList` to `DataSource`
(restore the selection by id). Never touch the bound list in place on an editable grid with a footer. Precedent:
`CWATIS/Features/Warehouse/Warehouses/View/WhWarehousesForm.cs` (name column inline, "Всего:" footer on the № column). At the time of
writing this was built and compiled but **not yet exercised in the app** — confirm by editing that form's last row before citing it
as proof, and update this note with the result either way.

### Checkbox / Boolean columns (`Style:"DataType:System.Boolean;..."`)

A column styled with `DataType:System.Boolean` renders a checkbox glyph and toggles on click, going through the exact same `StartEdit`/`AfterEdit` lifecycle as a text column — no special handling needed beyond the guard + persist pattern above (and the `GetModelRow<T>` gotcha, which applies here just as much as to any other column). `AllowEditing` at the column level (default `true` unless set otherwise) controls whether the checkbox responds to clicks at all; there's no need to set it to `False` and drive the checkbox by hand via `MouseDown`/`HitTest` — that was tried as a workaround while the real bug (the `DataSource`-casting gotcha above) was still unidentified, and turned out to be unnecessary once the actual defect was fixed.

Working precedent for a guarded checkbox column: `userpermissionsui/RolesManagerForm.cs` — `RolesGrid` (also a `MicrosSmartGrid.SmartGrid`, also `DrawMode=OwnerDraw`) has a `DataType:System.Boolean` column literally named `"IsDefault"`, guarded in `RolesGrid_StartEdit`. Its own code comment states the `BeforeEdit`-vs-`StartEdit` reasoning directly: "Почему проверка происходит здесь а не в BeforeEdit? Потому что BeforeEdit вызывается не только при пользовательском редактировании, но и при рендеринге." **Correction:** this used to also cite `ContractTypesForm.cs`'s `masterGrid_StartEdit`/`masterGrid_AfterEdit` as following the same pattern for its own `"IsDefault"` column (single-default-type invariant, confirmation on replacing an existing default) — per the same `StartEdit`-citation error caught elsewhere in this doc, `ContractTypesForm.cs`'s Designer.cs only wires `masterGrid_BeforeEdit`/`masterGrid_AfterEdit`, no `StartEdit` at all. Whether the "IsDefault" enforcement logic itself still exists in that `BeforeEdit`/`AfterEdit` pair, or exists at all, wasn't re-verified — check the actual current file before citing this as precedent for anything.

## Gotcha: yellow "selected" row visible in the VS designer (`HighLight` default is `Always`)

C1FlexGrid's `HighLight` property defaults to `HighLightEnum.Always`, which paints the current row/cell with the `Highlight` style **even when the grid has no focus — including on the VS designer surface**. In CWATIS the shared `StyleInfo` defines `Highlight{BackColor:255, 255, 180;...}` (pale yellow), so a grid dropped on a form/UserControl shows a phantom yellow first data row in the designer.

The same flag controls both surfaces (the designer host is just an unfocused grid), so a single value can't give "clean designer + always-visible runtime selection". **Fix is two-step**: serialize `WithFocus` in the Designer.cs, then restore `Always` at runtime in the constructor, guarded by the design-time flag (see `../SolvedProblems/winforms/winforms-usercontrol-design-time-safety.md` for `_isDesignTime`):

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

## Gotcha: `SmartGrid.GetSelectedRows()` has no bounds check — throws if called after `DataSource` shrinks while an old multi-selection is still set

**Confirmed against `SmartGrid`'s own source** (`Prototypes/SmartGrid/SmartGrid/SmartGrid.cs` — the prototype project the compiled `MicrosSmartGrid`/`MicrosSmartGrid.V2` package is built from; property/method names match exactly, only some C1 namespaces differ, e.g. `C1.Win.FlexGrid` there vs `C1.Win.C1FlexGrid` in the currently-referenced package version):

```csharp
public List<int> SelectedRows { get; set; }

public List<Row> GetSelectedRows()
{
    List<Row> res = new List<Row>();
    for (int i = 0; i < SelectedRows.Count; i++)
    {
        res.Add(Rows[SelectedRows[i]]);   // no bounds check against Rows.Count
    }
    return res;
}
```

`GetSelectedRows()` is a thin wrapper over the `SelectedRows` property (a plain `List<int>` of row indices, populated by the grid's own Ctrl/Shift-click handling — separate from the native C1FlexGrid cell-range `Selection`). It does **not** validate that any index is still `< Rows.Count` before indexing `Rows[...]`. If `DataSource` is reassigned to a *shorter* list (e.g. `smartGrid1.DataSource = new C1DataCollectionBindingList(filtered)` inside a delete/mutate loop) while `SelectedRows` still holds indices sized for the pre-rebind row count, the next call to `GetSelectedRows()` — direct or indirect (e.g. from an `AfterSelChange` handler that fires as a side effect of the rebind) — throws a plain `ArgumentOutOfRangeException` ("Индекс за пределами диапазона...") straight out of `Rows[i]`, unrelated to any bounds-checking in caller code. `GetModelRow<T>`/`GetModelSelectedRow<T>` (`SmartGrid_Extentions.cs`) are **not** a safety net here — they guard their *own* `index` parameter correctly, but that's irrelevant when the exception is already thrown one call earlier, inside `GetSelectedRows()` itself, before any index ever reaches `GetModelRow`.

**Concretely, this bites any "mass operate on selected rows" loop that mutates+rebinds the grid per-item**, e.g.:

```csharp
// selected captured via GetSelectedRows() BEFORE the loop starts - fine at this point
var selected = GetSelectedModels();
foreach (var model in selected)
{
    await OnDelete(model.Id);   // -> RemoveRow -> rebinds DataSource to a shorter list
    // any subsequent DeleteSB.Enabled = HasTypes && HasDeletableSelection() (or similar helper
    // that re-calls GetSelectedRows()) can now throw, once a stale selected index >= the new Rows.Count
}
```

### Fix

- Anywhere that recomputes enablement/state as a *side effect* of a programmatic `DataSource` rebind (a shared "reapply filter and rebind" method, called from `RemoveRow`/`InsertRow`/`DisplayGridData`), use a single-row, scalar check (`grid.Row`, always kept in-bounds by the grid itself on rebind — e.g. via `GetModelSelectedRow<T>()`) instead of anything touching `GetSelectedRows()`/`SelectedRows`.
- Once a multi-selection has been read and is no longer needed (e.g. captured into a `toDelete`/`toChange` list before a batch loop), clear it explicitly — `grid.SelectedRows?.Clear();` — *before* starting the loop. This is the same thing `SmartGrid_Extentions.GetValues<T>()` already does right after consuming `SelectedRows` (`grid.SelectedRows.Clear();`), just applied proactively instead of after a single read.
- Reserve `GetSelectedRows()`-based (multi-select-aware) checks for handlers that only ever fire from live, real user interaction (`AfterSelChange`) — never from a method that itself just reassigned `DataSource`.

Found in GlobalForms, 2026-08-25, in `CategorizedSubdivisionsForm.DeleteSB_Click`'s mass-delete path, after a `HasDeletableSelection()` helper (checking `!m.IsApproved` across `GetSelectedRows()`) was wired into `ApplyTypeFilter` — called from `RemoveRow`, which runs once per row inside the mass-delete loop. Fixed by adding a scalar-only `CanDeleteFocusedRow()` for the rebind-triggered call sites and clearing `SelectedRows` before the loop in `DeleteSB_Click`; the same two-part fix applies to `approveCommand_Click`'s mass-approve loop.

Two more confirmed facts about `SmartGrid`, from the same prototype source (`Prototypes/SmartGrid/SmartGrid/SmartGrid.cs`), both relevant to the gotcha above:

- **`GetSelectedRows()` already falls back to the current row when `SelectedRows` is empty** — its full body is `for (...) res.Add(Rows[SelectedRows[i]]); if (res.Count == 0) res.Add(Rows[Row]); return res;`. So "is anything selected" already means "explicit multi-selection, or just the current row" without needing to ever add the current row to `SelectedRows` yourself.
- **Membership in `SelectedRows` repaints a *shared*, not per-row, style object** — `SmartGrid`'s own `SelChange` handler does:
  ```csharp
  if (Row >= Rows.Fixed)
  {
      if (SelectedRows != null && SelectedRows.Contains(Row))
      {
          Styles.Highlight.ForeColor = Color.Red;
          Styles.Focus.ForeColor = Color.Red;
      }
      else
      {
          Styles.Highlight.ForeColor = DefaultForeColor;
          Styles.Focus.ForeColor = DefaultForeColor;
      }
  }
  ```
  `Styles.Highlight`/`Styles.Focus` are the grid's shared style objects (used for whichever row is currently highlighted/focused), not something scoped to one row. Whether the *current* row happens to also be `SelectedRows`-listed flips these between red and normal text for the whole grid — this is the native "Bookkeep-style" explicit multi-select marking (`Insert`/`Num *` keys), a different concept from "this is simply the current row." **Never add a row to `SelectedRows` just to make it *look* selected/focused** (e.g. "restore first-row selection after a refresh") — that pulls in this red-text multi-select styling as a side effect, which reads as a different, wrong selection mode to anyone used to the grid's normal highlight. Set `grid.Row` alone; native `HighLight=Always` plus the default `Highlight`/`Focus` styles already render a normal current-row highlight without touching `SelectedRows` at all.

  Found in GlobalForms, 2026-08-25, in the same `CategorizedSubdivisionsForm.ApplyTypeFilter` — an initial fix for "first row should look selected after refresh" (regressed when the junk `SelectedRows` blob above was removed) added `smartGrid1.SelectedRows.Add(smartGrid1.Rows.Fixed)` alongside `smartGrid1.Row = smartGrid1.Rows.Fixed`, which made every refresh paint the first row's text red instead of the normal highlight color. Fixed by dropping the `SelectedRows` line entirely and keeping only the `.Row` assignment.

## Gotcha: hand-editing a `ColumnInfo` resx string — the leading number is the live column count

`AutoGenerateColumns=false` grids define their entire column set through one serialized string, written by the VS Designer and stored in the form's `.resx`:

```csharp
this.smartGrid1.ColumnInfo = resources.GetString("smartGrid1.ColumnInfo");
```

e.g. `<data name="smartGrid1.ColumnInfo"><value>3,0,0,0,0,-1,Columns:0{Width:30;Name:"RowNumber";...;}	1{...}	2{...}	</value></data>`. When adding/removing a column by hand (no Designer available to regenerate it), it's tempting to treat the leading `3,0,0,0,0,-1,` prefix as unrelated grid metadata (fixed-rows count, sort state, …) — especially misleading when two unrelated sibling forms both happen to have 3 columns and share that exact literal prefix, which looks like proof it's unrelated to column count.

**It isn't.** The first number in that prefix is the total column count, and must stay in sync with the number of `N{...}` column blocks in `Columns:...`. Confirmed by comparing forms with different column counts in this same repo: `PlacesTreeForm` has 8 columns (indices 0-7), prefix `8,1,0,0,0,-1,`; `LocationTreeViewForm` has 12 columns (indices 0-11), prefix `12,1,0,0,0,-1,`.

### Fix

When hand-editing `ColumnInfo` to insert/remove a column:
- Renumber every `N{...}` block's leading index sequentially (`0,1,2,...`) in the new column order.
- Update the leading count number in the prefix to the new total column count.
- Update the paired `Headers` array in the sibling `.Designer.cs` to match — it is a separate property and does not resync itself. Insert/remove the cell at the same column index in **every** header row (never retype the arrays from scratch — that destroys the merge layout); see "Multi-row headers" below for the format and the `Caption` rule.
- Copy an existing sibling column's exact `{...}` block (font/`StyleFixed`/`AllowMerging`) for the new column instead of inventing formatting from scratch, and grep sibling forms for the same resource name to find a column-count-to-prefix-number precedent before trusting the edit.

Found in GlobalForms, 2026-08-26, adding a "Код" column to `SubdivisionCategoriesForm` (inserted between the existing `IsApproved`/`Name` columns, copying the `Code` column block already present in the sibling `CategorizedSubdivisionsForm`).

### Symptom when the count is too low: trailing columns silently vanish

A prefix smaller than the number of blocks is **not an error**: the grid builds only the first N blocks and drops the rest. Build is
green, the `.resx` visibly contains the missing column, and the bound model property exists, so everything you can grep says the
column should be there; it just isn't on screen. Worse, the next time the VS Designer saves that form it re-serializes the
truncated set, so the dropped blocks disappear from the `.resx` for good (looks like "the designer deleted my columns").
"Columns at the end of the list are missing, the ones before them are fine" is the fingerprint. Before assuming the designer
reverted the file, compare the prefix with the block count. Every grid in a feature folder can be checked at once (run from a file
written with the Write tool, not a Bash heredoc, if the pattern gets backslashes, see claude-global/SolvedProblems/tooling/git-bash-mangles-msbuild-slash-flags.md):

```python
import re, sys
s = open(sys.argv[1], encoding='utf-8').read()
for m in re.finditer(r'<data name="([^"]+)\.ColumnInfo"[^>]*>\s*<value>(\d+),[^C]*Columns:(.*?)</value>', s, re.S):
    n = len(re.findall(r'(?:^|\t)\d+\{', m.group(3)))
    if int(m.group(2)) != n: print(sys.argv[1], m.group(1), 'declared', m.group(2), 'defined', n)
```

Found in CWATIS, 2026-11-11, `WhDocTypesForm`: a grid cloned from a 4-visible-column form kept prefix `4` after columns were added,
so «Код» (appended last) never showed. The source form `WhWarehousesForm` itself declares 4 but defines 5; only its hidden
`wh_list_id` column is dropped, which is harmless there because rows are found by model, not by that column.


## Multi-row headers: when `Rows.Fixed > 1`, headers live in `Headers`, not in column `Caption`

If `grid.Rows.Fixed > 1` the grid has several header rows. From that point the column `Caption` field must stay **empty** for every column — captions are defined only through the `Headers` property (set in the `.Designer.cs`, edited via the `[…]` editor in the Properties grid).

`Headers` holds one entry per fixed header row; in the Designer each entry is a tab-separated string with one cell per column, in column order (the editor shows it as a table). Cells with equal text in neighbouring positions merge automatically (horizontally within a row, and vertically across rows), which is how group headers are built:

```csharp
this.grid.Headers = new string[] {
    "№\tТовар\tУчет\tФакт\tРасхождение\t",       // row 0: group header, repeated text merges
    "№\tТовар\tУчет\tФакт\tИзлишек\tНедостача"    // row 1: leaf header
};
```

**Rules**
- Single fixed row (`Rows.Fixed == 1`): use column `Caption` as usual. More than one: clear all `Caption` values and use `Headers` only — mixing the two gives duplicated or unmerged header text.
  - Why: at runtime the column `Caption` **overwrites header row 0**, whatever `Headers` says. Symptom (CWATIS warehouse
    directories, 2026-11-23): `Headers` row 0 was `…\tУтвердил\tУтвердил` with Кто / Когда below, but the running grid showed
    two unmerged cells «Утвердил» | «Утвержден» — the second column's leftover `Caption:"Утвержден"`. Either empty `Caption`, or
    `Caption` equal to the column's row-0 text (as `WhInventoryReportsForm.gridReports` does: both «Акт» columns have
    `Caption:"Акт"`). When a group header won't merge, compare every `Caption` in the `.resx` `ColumnInfo` with row 0 first.
- **Adding or removing a column means editing every row of `Headers` by hand** — the full checklist (together with the `ColumnInfo` count/index edits) is in "hand-editing a `ColumnInfo` resx string" above. Copy the neighbouring column's cells as a starting point so merging with its group stays intact.
- Hit-testing and `GetModelRow` treat all fixed rows as non-data rows (`hit.Row >= grid.Rows.Fixed` to skip them).

Source: team convention stated by the project owner, 2026-10-09 (not derived from the library docs; `SmartGridReference.md` describes `Headers` only briefly).

## Gotcha: hiding zero values in a SmartGrid column — use `OwnerDrawCell`, match columns by `Name`

Wanted: numeric read-only columns (stock balances) that show **nothing** instead of `0,00`.

**What did not work (observed in CWATIS `WhInventoryReportsForm.gridLines`, 2026-10-15):** setting `grid.Cols[name].Format = "#,##0.00;-#,##0.00;"`
(empty third section) in the form constructor — the zeros stayed. The grid there is `DrawMode = OwnerDraw`; the exact reason was not established,
so don't rely on `Column.Format` for this.

**What to do** - blank the text in `OwnerDrawCell` (the grid must have `DrawMode = DrawModeEnum.OwnerDraw`, and the event is wired in the Designer):

```csharp
private static readonly string[] HideZeroColumns = { "Actual_Rws_Qty", "Actual_Bk_Bln_Qty" };

private void grid_OwnerDrawCell(object sender, OwnerDrawCellEventArgs e)
{
    if (e.Row < grid.Rows.Fixed || e.Row >= grid.Rows.Count) return;      // header / footer rows untouched
    if (e.Col < 0 || e.Col >= grid.Cols.Count) return;
    if (Array.IndexOf(HideZeroColumns, grid.Cols[e.Col].Name) < 0) return;

    if (decimal.TryParse(e.Text, NumberStyles.Number, CultureInfo.CurrentCulture, out decimal shown) && shown == 0m)
        e.Text = string.Empty;
}
```

Notes:
- Check the **display text** (`e.Text`), as the working precedent `ProductsNotSoldRepForm.DetailGrid_OwnerDrawCell` does. An earlier variant here read the raw
  value through `grid[e.Row, e.Col]`; it was never seen working, but it was also never tested with matching column names (see next point), so
  it is not proven bad — `e.Text` is simply the form already known to work in this project.
- The handler matches columns by `Cols[e.Col].Name`. **If a column's `Name` in the resx `ColumnInfo` changes, update the list.** And if the zeros "come back", first
  re-read the `ColumnInfo` names on disk: the VS designer can silently revert them
  (`../SolvedProblems/winforms/vs-designer-reverts-external-resx-edits.md`) — that, not the handler, was the cause of the repeated "still zeros" in the case above.
- Only the shown text changes; the cell value stays `0` (sorting, `GetModelRow`, aggregates are unaffected).
- Other precedent: `SmartGridExtentionsCW.HideTimeInDateTimeColumn(grid, e)` (blanks `DateTime.MinValue`).

## Gotcha: a `Count` footer ("Всего:") counts only non-empty cells of its column

`AggregateEnum.Count` is documented as "Returns the count of non-empty cells in a range" (local `C1.Win.C1FlexGrid.4.8.xml`, package
`4.8.20231.596`). A "Всего:" footer placed on a nullable column (an optional owner, a serial number, a comment) therefore
under-counts the rows silently. The usual precedent (`gridLines`: `Column = 1` = goods name) only works because that column is
always filled.

**Rule:** anchor the `Count` aggregate on a column that is never empty for a data row — typically the № column. If the № is drawn
by position in `OwnerDrawCell` (so it stays continuous after `RemoveAt`/sorting), still give the bound `RowNum` property a
non-zero value: the aggregate reads the cell values, not the drawn text. Precedent: `Features/Warehouse/GoodsDetails/View/WhGoodsDetailsForm`
(2026-10-15; documentation-based, the under-count itself was not reproduced).

## Gotcha: never let the selection rest on a footer row — a data change then crashes C1 paint ("index" out of range)

**Symptom** (CWATIS invoice form, 2026-10-21): edit a cell (price) in a grid with a footer, then click the footer row directly ->
«Ошибка приложения»: "Индекс за пределами диапазона ... Имя параметра: index". A plain footer click does nothing; edit + click on a
data row is fine. Stack (from the `Errors\*.err` log, see `../../projects/CWATIS/.claude/docs/SolvedProblems/cwatis-error-dialog-stack-trace-log.md`):
`ArrayList.get_Item` <- `C1.Win.C1FlexGrid.RowCollection.get_Item` <- `C1FlexGrid.DrawRow` <- `C1FlexGridBase.OnPaint` — no app frame at all.

**Mechanism (inferred, not instrumented):** clicking the footer while an editor is open commits the edit (`AfterEdit`) and then moves the
selection onto the footer row. The `AfterEdit` handler then changes bound data (it writes the recalculated amount into a cell), C1
recalculates the footer aggregates by rebuilding the footer rows, and the next paint draws the still-selected old footer row index,
which no longer exists. Separately, `GetModelRow<T>(footerRow)` throws the same error on its own: `Rows[footer].DataSource` maps to
a list index one past the end.

**Fix:** cancel any selection / current-row change that reaches a footer row. Locate the footer **from the data source count**, not
from the grid's row count:

```csharp
// first footer row = fixed rows + rows in the bound list (0 before the first binding)
private static int FirstFooterRow(SmartGrid g)
    => g.Rows.Fixed + ((g.DataSource as System.Collections.ICollection)?.Count ?? 0);
private static void KeepOffFooter(SmartGrid g, RangeEventArgs e)
{
    int first = FirstFooterRow(g);
    if (e.NewRange.r1 >= first || e.NewRange.r2 >= first) e.Cancel = true;
}
// wire for every grid with footers:
grid.BeforeSelChange   += (s, e) => KeepOffFooter(grid, e);
grid.BeforeRowColChange += (s, e) => KeepOffFooter(grid, e);
```

and treat a footer row as "no selection" wherever the current row is read (`grid.Row >= FirstFooterRow(grid)` -> empty model).

**Trap in the guard itself:** the first version computed `FirstFooterRow` as `Rows.Count - Footers.Descriptions.Count` ("footers are the
last rows"). That is only true once the grid is fully built. While `DataSource` is being replaced, C1 already moves the current row to
the first data row **before** it appends the footer rows, so that formula points at a data row, the guard cancels C1's default
"select the first row after binding", and the grid loads with nothing selected (seen 2026-10-21: the invoice list no longer selected
its first document on load). Counting the bound list is correct at every moment, including mid-rebind and before any binding.
Precedent: `Features/Warehouse/Movement/View/SubView/WhMovementControl.cs` (fix built 2026-10-21, not yet re-tested in the app at the
time of writing). `WhInventoryReportsForm` has the same exposure (footer sums on edited columns, `GetModelSelectedRow` on selection) and
did not have the guard yet.

## Per-column edit permissions: `Tag` JSON read by `UserPermissionsControl` (already supported)

Whole picture (form `Tag`, command `UserData`, tables, startup): [UserPermissions.md](UserPermissions.md).

`UserPermissionsControl` (NuGet 1.1.0.6, source: `userpermissionscontrol` repo, `master` branch,
`PermissionHandlers/C1FlexGridPermissionHandler.cs`) already handles any `C1FlexGrid`, so `SmartGrid` too. The grid's `Tag`
holds a JSON list of right objects, each naming the columns it governs (column `Name`, not caption):

```csharp
_detailGrid.Tag = "{\"Tags\":[{\"Name\":\"PriceEdit\",\"Columns\":[\"Price4Disc\",\"BaseRtlPrice\"]},"
                + "{\"Name\":\"QntEdit\",\"Columns\":[\"FreeDepQNT\",\"BkBlnQnt\"]}]}";
```

Per listed column: `Edit` → `Visible = true, AllowEditing = true`; `View`/`Select` → `Visible = true, AllowEditing = false`;
`None` → `Visible = false`. Precedent: `Forms/PriceList/Views/GoodsTree4PriceListForm` (2026). Verified by reading the source
and checking the packaged DLL contains `SetupGridColumnPermission` (2026-11-17).

Traps:
- **`ShowWithPermission()` does not apply it.** That method (`TmplMDIForm`/`TemplateFormMDI`) only calls `CanShowForm()` (the
  form `Tag`). Grid columns and toolbar `UserData` commands are applied only by `new PermissionManager(this).Initialize()`,
  which the form must call itself — in `Load`, **after** the grid's columns exist (and once only: it re-queries every right
  synchronously; calling it from `Activated` hung `GoodsTree4PriceListForm`).
- **Fails open.** A right object the service returns `null` for is skipped, so its columns stay as the designer left them
  (unlike toolbar commands, which default to `None`/disabled). Keep the designer default `AllowEditing = false` on guarded
  columns so a missing right means read-only.
- **Not enforced afterwards.** It's a one-shot `AllowEditing` write; form code that later sets `column.AllowEditing = true`
  (view-mode switches), rebuilds columns (`AutoGenerateColumns` + rebind), writes cells in code, or SmartGrid's Ctrl+V
  `Paste()` path is not checked. Re-check the right in the form's own edit gate (`BeforeEdit`/save) when it matters.
- **It can un-cancel your `StartEdit`.** Its `StartEdit` handler sets `e.Cancel = false` for Edit-permitted columns and is
  subscribed after the Designer's handlers, so a business-rule cancel placed in `StartEdit` is overridden. Put such checks in
  `BeforeEdit` (raised before `StartEdit`; a cancel there stops editing before the permission handler runs).
- Overwrites `Visible = true` on every listed `Edit`/`View` column, un-hiding columns the designer or a view mode hid.
## Gotcha: assigning `DataSource` raises `AfterSelChange` mid-rebind — `GetSelectedRows()` in the handler throws

Setting `grid.DataSource = new C1DataCollectionBindingList(...)` makes C1 select a row itself (`C1FlexGridBase.Select` from the binding code),
which raises `AfterSelChange` **while the rebind is still in progress**. A handler that calls `grid.GetSelectedRows()` (e.g. to enable / disable
toolbar commands from the selection) then fails:

```
System.ArgumentOutOfRangeException  Source=mscorlib  (Имя параметра: index)
   at C1.Win.C1FlexGrid.RowCollection.get_Item(Int32 index)
   at MicrosSmartGrid.SmartGrid.GetSelectedRows()
   at ...Form.<your AfterSelChange handler>
   at C1.Win.C1FlexGrid.C1FlexGridBase.Select(Int32 row, Int32 col)
```

The same applies to the in-place `RemoveAt` + `Insert` update (between the two calls the list is one row short).

**Fix:** unsubscribe the handler around the rebind / in-place update and run the refresh once afterwards (the "no suppression flags" rule —
unsubscribe/resubscribe, not a bool):

```csharp
grid.AfterSelChange -= grid_AfterSelChange;
grid.SelectedRows?.Clear();
grid.BeginUpdate();
grid.DataSource = new C1DataCollectionBindingList(items);
grid.EndUpdate();
grid.AfterSelChange += grid_AfterSelChange;
RefreshCommands();
```

Also: with no marked rows `GetSelectedRows()` falls back to the current row, which on an empty grid is `-1` / a fixed row → the same exception.
Guard `Rows.Count <= Rows.Fixed || Row < Rows.Fixed` before calling it.

Found 2026-10-24 in CWATIS (`Features/Warehouse/StockGoods/View/WhStockGoodsForm`: `BindCards` / `UpdateCards` / `GetSelectedCards`).

## Standard: clear the designer's placeholder rows when the form is created

Every grid comes out of the Designer with placeholder rows (`Rows.Count = 5`, `10`, `51`, ...). Until the first real load they show as an empty
striped table, and code that reads "the current row" sees rows that aren't data. **Standard pattern: right after `InitializeComponent()`, set each grid
(SmartGrid and TreeFlexGrid alike — both are `C1FlexGrid`) to zero data rows, keeping the fixed and footer rows:**

```csharp
public SomeForm()
{
    InitializeComponent();

    // пустые строки из дизайнера не должны мелькать до загрузки данных
    ClearGrid(gridMaster);
    ClearGrid(gridDetail);
    ...
}

/// <summary>Ноль строк данных: остаются только фиксированные строки и строки футера.</summary>
private static void ClearGrid(C1FlexGrid grid)
{
    grid.Rows.Count = grid.Rows.Fixed + grid.Footers.Descriptions.Count;
}
```

Count the footer rows in (`+ Footers.Descriptions.Count`). A plain `Rows.Count = Rows.Fixed` (older forms, e.g. `TgMembersForm`, `MTPProfilesForm`)
is only correct for a grid with no footer. Don't fix it by changing `Rows.Count` in the Designer: the designer rewrites it on the next save.

Precedents: `WhInventoryReportsForm`, `WhMovementControl`, `WhGoodsDetailsForm`, `WhStockGoodsForm` (`Features/Warehouse`).

## Standard: a read-only checkbox column shows a checkbox only when the value is true

In a `DataType:System.Boolean` column the user can't toggle (the column's or the grid's `AllowEditing` is `false`), an empty checkbox on every
`false` row is visual noise: a column full of empty squares in which the few `true` rows are hard to spot. **Rule: if the checkbox column isn't
editable, draw nothing for `false`, only the ticked checkbox for `true`.** Editable checkbox columns keep the normal empty box: there it is the
click target.

In `OwnerDrawCell` (grid `DrawMode = OwnerDraw`), draw only the background and border for an unchecked cell, so the check glyph is never painted:

```csharp
/// <summary>Нередактируемый чекбокс показываем только у true: у false — только фон и рамка, без глифа.</summary>
private void grid_OwnerDrawCell(object sender, OwnerDrawCellEventArgs e)
{
    if (e.Row < grid.Rows.Fixed || e.Row >= grid.Rows.Count) return;
    if (e.Col < 0 || e.Col >= grid.Cols.Count) return;

    var col = grid.Cols[e.Col];
    if (col.DataType != typeof(bool) || (grid.AllowEditing && col.AllowEditing)) return;

    if (grid.GetCellCheck(e.Row, e.Col) == CheckEnum.Unchecked)
    {
        e.DrawCell(DrawCellFlags.Background | DrawCellFlags.Border);
        e.Handled = true;
    }
}
```

- Read the state with `GetCellCheck`, i.e. what the grid is about to paint. Reading the value through `grid[row, col]` is unreliable for rows of a
  `C1DataCollectionBindingList` (see the inventory form's `gridLines_OwnerDrawCell` note).
- Keep `Background | Border` (not `e.Handled = true` alone): the row's selection / highlight colour and the cell border must still be drawn.
- Check bounds first: `OwnerDrawCell` also fires for fixed and footer rows.

Precedent: `WhStockGoodsForm.gridCards_OwnerDrawCell` (column `stockgoods_is_closed`, 2026-10-24). Status: builds; the painting itself has not yet
been confirmed in the running app. If the glyph still shows, check that the column really has `DataType:System.Boolean` (otherwise `GetCellCheck`
returns `None`).
