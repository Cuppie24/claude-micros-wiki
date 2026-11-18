# WinForms: grid/control setup that reads back post-init state belongs in `Load`, not the constructor

**Symptom** — `System.NullReferenceException` ("Ссылка на объект не указывает на экземпляр объекта." / "Object reference not set to an instance of an object.") thrown directly from a form's own constructor — the stack trace's top frame is `SomeForm..ctor()`, often with no line number if the host app doesn't have that assembly's PDB loaded. Reproduces on **every** instantiation of the form (e.g. via a host app's menu command doing `new SomeForm()`), not intermittently.

Concretely hit in `MC_GlobalForms.Forms.CategorizedSubdivisions.View.CategorizedSubdivisionsForm..ctor()`, called from `CWATIS.MainForm.subdivisionsCommand_Click`. The crashing line:

```csharp
public CategorizedSubdivisionsForm()
{
    InitializeComponent();
    smartGrid1.Cols["IsApproved"].Style.TextAlign = C1.Win.C1FlexGrid.TextAlignEnum.CenterCenter; // NullReferenceException
    ...
}
```

**Root cause** — `Cols["ColumnName"]` (and, by the same principle, any other property/collection on a child control that depends on the control being fully realized — handle created, first layout pass done) is not guaranteed to be populated purely by an earlier property assignment inside `InitializeComponent()` (here, `smartGrid1.ColumnInfo = resources.GetString(...)`, a few lines above). Immediately after `InitializeComponent()` returns — i.e. still inside the *form's own constructor* — the child control may not have a native window handle yet, since the whole form hasn't been shown. `Cols["X"]` can come back `null` there, even though the exact same access from `Load` (or later) resolves every time.

This codebase already had the tell for it before this bug: `grid.Cols["Name"]?.Index`-style null-conditional access appears at several existing call sites (e.g. `smartGrid1_AfterEdit`, `SmartGrid_Extentions.cs`) — that `?.` only makes sense if `Cols["X"]` is known to legitimately return `null` sometimes. An unguarded, constructor-time `Cols["X"].SomeProperty = ...` skips that same safety net.

Don't trust an existing, apparently-working constructor precedent as proof the pattern is safe in general — a pre-existing line in the same constructor (`Cols["RowNumber"].AllowEditing = false;`) had never crashed, but that doesn't mean it was ever actually safe; it may just not have hit the same timing window, or nobody exercised that exact path recently. When this bug was found, that older line was moved to `Load` too rather than left as a trusted precedent.

**Fix** — move any control setup that reads back post-initialization state (not just "assign a designer-serialized property verbatim") out of the constructor and into the form's `Load` handler, and guard it defensively:

```csharp
private void SomeForm_Load(object sender, EventArgs e)
{
    var col = smartGrid1.Cols["IsApproved"];
    if (col != null)
    {
        col.Style.TextAlign = C1.Win.C1FlexGrid.TextAlignEnum.CenterCenter;
        col.StyleFixed.TextAlign = C1.Win.C1FlexGrid.TextAlignEnum.CenterCenter;
    }
    ...
}
```

Not SmartGrid-specific — the same reasoning applies to any control whose child collections/sub-objects are resolved lazily from a declarative string/resource rather than being directly constructed objects (tree nodes, list items, column collections on other grid controls, etc.).

**Checklist**

- Rule of thumb: constructor = call `InitializeComponent()`, wire up events, assign simple field-level defaults. Anything that reads back a child control's *post-initialization* state (`Cols[...]`, `Items[...]`, `Nodes[...]`, computed layout, anything that could still be `null`/unpopulated before the control is realized) = `Load`.
- If a `NullReferenceException` stack trace's top frame is a form's own `.ctor()` with no further detail, look first at anything in that constructor that indexes into a named sub-collection on a child control, rather than assuming the exception is deeper in framework/library code.
- Guard the access with a null check regardless of where it ends up — cheap, and consistent with the `?.` pattern already used elsewhere in this codebase for exactly this uncertainty.
- A pre-existing constructor-time access that "has always worked" is not proof the pattern is safe — it may simply not have been exercised at the exact moment it matters. Prefer moving it to `Load` over leaving it as trusted precedent once the risk is identified.

Found in GlobalForms, 2026-08-25, `CategorizedSubdivisionsForm`/`SubdivisionCategoriesForm` (both had the same pattern added in the same session — `Cols["IsApproved"]` for a new checkmark column's centering — and `SubdivisionCategoriesForm`'s older `Cols["RowNumber"].AllowEditing` line was moved alongside it once the risk was identified). Confirmed via a real captured stack trace (`CWATIS.MainForm.subdivisionsCommand_Click` → `CategorizedSubdivisionsForm..ctor()`), not just static reasoning.
