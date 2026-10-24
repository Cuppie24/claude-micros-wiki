# TreeFlexGrid Conventions (CWATIS)

Conventions for working with `Manufacture.Compotents.TreeFlexGrid` (assembly `Tools.V2`, `packages\Tools.V2.1.0.3.7\lib\net48\Tools.V2.dll`) in the CWATIS WinForms client. See also `SmartGridConventions.md` (same directory) for the sibling `SmartGrid` control, and `SmartGridExtentions.cs` (`CWATIS\Extentions\SmartGridExtentions.cs`) which holds several `TreeFlexGrid`-specific extension methods alongside the `SmartGrid` ones.

`Manufacture.Compotents.TreeFlexGrid` is the real, in-project tree grid — don't confuse it with anything from an unrelated app; every hierarchical grid in this codebase (`GoodsTree4PriceListForm`, `GoodsTree4PriceCardsForm`, `SetAddForm`, `ProductsNotSoldRepForm`, `JSONVIewForm`, among others) uses this exact type.

## The `IDataTree` contract

The data bound via `BuildTreeNew` must implement `Tools.IDataTree` (verified via reflection against `Tools.V2.dll`):

```csharp
public interface IDataTree
{
    int Id { get; set; }
    int ParentId { get; set; }
    string Name { get; set; }
}
```

`Id`/`ParentId` link the hierarchy (root nodes typically use a `ParentId` value that matches no real `Id`, e.g. `0`). `Name` is not just a display convenience — it is the specific property `TreeFlexGrid` uses to render the indentation/expand-collapse decoration. This matters more than it looks like it should; see the gotcha below.

## Setting `DataSource` / building the tree

```csharp
_masterGrid.BuildTreeNew(data);       // data: IEnumerable<T> where T : IDataTree
_masterGrid.CollapseByLevel(1);
_masterGrid.ExpandToNodeById(id);
```

(`BuildTreeNew`/`CollapseByLevel` are members of `TreeFlexGrid` itself; `ExpandToNodeById` is the extension method in `SmartGridExtentions.cs`, not to be confused with `TreeFlexGrid`'s own APIs.)

## Columns: always `AutoGenerateColumns = false`

Set `AutoGenerateColumns = false` and define `ColumnInfo` explicitly, same as `SmartGrid` (see `SmartGridConventions.md`). `ColumnInfo` can be a resx-stored string (`resources.GetString("grid.ColumnInfo")`, the usual VS-Designer-generated path) or a plain inline string literal assigned directly in `InitializeComponent()` — both are valid; use inline when the form doesn't otherwise need a `.resx` (precedent: `JSONVIewForm.cs`, which has no `.resx` at all).

### Column shape depends on how much the tree needs to show

- **Simple/single-purpose tree** (e.g. `JSONVIewForm`'s JSON viewer): one visible column named `"Name"`, everything else (`Id`, `ParentId`) hidden (`Visible:False`).
  ```
  3,0,0,0,0,-1,Columns:0{Width:400;StarWidth:"*";Name:"Name";Caption:"...";AllowEditing:False;}	1{Name:"Id";Caption:"Id";Visible:False;}	2{Name:"ParentId";Caption:"ParentId";Visible:False;}	
  ```
- **Business tree with multiple data columns** (e.g. `SetAddForm._priceGrid`, `GoodsTree4PriceListForm._masterGrid`): `"Name"` is still a required bound column (usually but not always column 0 — `SetAddForm` puts it at index 1, after a narrow `"No"` column), but many other real business columns stay visible (prices, quantities, article codes, etc.) alongside it; only genuinely internal fields (`Id`, `ParentId`, other bookkeeping columns) get `Visible:False`.

Either way: **every property on the bound model that isn't meant to be a visible column must still get an explicit `Visible:False` entry** — don't just omit it from `ColumnInfo`, or leave `AutoGenerateColumns = true`. See the gotcha.

### Gotcha: `AutoGenerateColumns = true` silently breaks tree rendering

Leaving `AutoGenerateColumns = true` on a `TreeFlexGrid` produces a tree that *looks* structurally correct — the expand/collapse nesting matches the real hierarchy depth, because `BuildTreeNew` binds correctly regardless — but the indentation/expand-icon decoration renders on whichever column ends up first in the **auto-generated column order**, which follows the model's property declaration order, not the property named `"Name"`. If `Id` is declared before `Name` on the model (a very natural ordering), the tree ends up indenting a column of bare sequential integers while the actual readable `Name` text sits in a separate, unindented column off to the side — visually reading as a mostly-blank tree with stray fragments. Always set `AutoGenerateColumns = false` with an explicit `ColumnInfo` to avoid this.

## Useful `TreeFlexGrid` extension methods (`SmartGridExtentions.cs`)

- `GetRowIndexById(this TreeFlexGrid grid, int Id)` — row index for a given `Id` (matches against `grid.IdName`, defaulting to `"Id"`).
- `GetNodeLevel(this TreeFlexGrid grid, int row)` — hierarchy depth (0 = root) of the node at a row.
- `GetNode(this TreeFlexGrid grid, int row)` — the underlying `Node` for a row, or `null` if it isn't a node row.
- `ExpandToNodeById(this TreeFlexGrid grid, int nodeId)` — expands all ancestors of the given node and selects it.
- `ExpandParentNodes(Node targetNode)` — expands every ancestor of a node (used internally by `ExpandToNodeById`, also callable directly).
- `StayLeaves(this TreeFlexGrid flexGrid, int id)` — selects the row for `id` by scanning the `"Id"` cell directly (falls back to the root row on any failure); an older/cruder alternative to `GetRowIndexById` + `grid.Row = ...`.
- `GetFirstRootNode(this TreeFlexGrid grid)` — `(RowIndex, Node, IsExpanded)?` for the first level-0 node, or `null`.
- `UnSelectAll(this TreeFlexGrid grid)` — clears row selection/highlight styling.

## Double-click on a column

Same pattern as `SmartGrid` — see "Handling Double-Click on a Specific Column" in `SmartGridConventions.md`. `TreeFlexGrid` supports `HitTest`/`MouseDoubleClick` identically.

## `BuildTreeNew` adds a synthetic root node

`BuildTreeNew` puts one extra node at the top, captioned `...`, with `Id = 0`. All the real root items (`ParentId` matching no `Id`) hang under it. Its other
bound columns show the model defaults (e.g. `0` in a count column). Seen 2026-10-24 in `WhStockGoodsForm`. Account for it:

- Blank its data columns in `OwnerDrawCell` (check `grid[row, "Id"] == 0` on a node row).
- Skip it when picking a default selection or reading the "current group" (`Id == 0` → nothing selected).
- To open the tree fully, set `row.Node.Expanded = true` on every node row. `CollapseByLevel(Tree.MaximumLevel)` did not leave the tree fully expanded there.

## Gotcha: `IdName` set in the Designer + serialized `SelectedID = 0` → `NullReferenceException` in `InitializeComponent`

The VS designer serializes `this.grid.SelectedID = 0;` for every `TreeFlexGrid`. The `SelectedID` setter calls `SetSelectedRow(id)`. That is
harmless while `IdName` is `null` (what every precedent designer has), but once `IdName` is set **earlier in `InitializeComponent`** it tries to
locate the row in the still-empty tree and throws:

```
System.NullReferenceException  Source=Tools.V2
   at Manufacture.Compotents.TreeFlexGrid.SetSelectedRow(Int32 id)
   at Manufacture.Compotents.TreeFlexGrid.set_SelectedID(Int32 value)
   at ...Form.InitializeComponent()
```

Deleting the `SelectedID` line alone is not enough: the designer writes it back on the next save. Keep `IdName = null` in the Designer and set it
in the constructor after `InitializeComponent()`. `IdName` is needed for `ExpandToNodeById` (it reads `row[grid.IdName]`):

```csharp
InitializeComponent();
gridGroups.IdName = "Id";
```

Found 2026-10-24 in CWATIS (`Features/Warehouse/StockGoods/View/WhStockGoodsForm`), on the first runtime open of the form.

## Precedent forms

`GoodsTree4PriceListForm.cs`, `GoodsTree4PriceCardsForm.cs`, `SetAddForm.cs`, `ProductsNotSoldRepForm.cs`, `ProductMoveRepForm.cs`, `ProductSalesRepForm.cs`, `JSONVIewForm.cs` (`Forms\JSONForm\View\`, the JSON tree/raw-text viewer opened from `ContractTypesForm`'s `HasJson` column).
