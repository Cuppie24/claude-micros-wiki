# WinForms UserControl design-time safety (VS designer breaks on forms hosting custom UserControls)

**Solved:** 2026-07-27, CWATIS (`ContractsForm` + `ContractDSListControl`/`ContractDSLinesTabControl`).

**Recurred:** 2026-08-10, CWATIS (`ContractsForm` → `RequisitesLinkBuilder` → `DocumentTypeTemplateControl`, two levels of UserControl nesting). Same shape: `DocumentTypeTemplateControl_Load` called `LoadData()` (a `CallService.CallProcedureDT` call) unconditionally. Also had trap #3 in the wild — its public `ContractId` had already been serialized into the host's Designer.cs (`RequisitesLinkBuilder.Designer.cs`) as `this.documentTypeTemplateControl1.ContractId = null;`; removed after hiding the property from the designer.

**Recurred again:** 2026-08-12, same CWATIS file/control. The 2026-08-10 fix was **never actually committed** — `git log --all -S "_isDesignTime" -- CWATIS/Controls/PriceList/Contract/DocumentTypeTemplateControl.cs` came back empty across every branch, while the same search on `ContractDSListControl.cs`/`ContractDSLinesTabControl.cs` (fixed in the same session) found their commits fine. The prior session must have diagnosed and written up the fix without the code change landing in git (lost working copy, session ended before commit, etc.) — this file documenting the fix is not proof the fix shipped. Re-applied the same pattern to `DocumentTypeTemplateControl`, and this time also hid the equivalent leaking properties on its siblings in the same nesting chain (`RequisitesLinkBuilder`, `ContractRequisitesControl`, `ContractRequisitesBuyerControl`) and scrubbed the resulting trap-#3 junk out of both `RequisitesLinkBuilder.Designer.cs` and `ContractsForm.Designer.cs` — not just the one property that happened to crash. **Verify a fix like this actually landed** with `git log -S "_isDesignTime" -- <path>` (or just re-reading the file) before trusting this doc's account of what's already fixed.

## Symptom

Opening a form in the Visual Studio designer shows an error dialog such as:

> Only objects of the type C1DockingTabPage can be added to the Controls collection of C1DockingTab.

or the designer white-screens / shows placeholder controls — even though the hand-written `InitializeComponent` is structurally identical to VS-generated code (verified against a VS-generated sibling). The error message points at a container control (C1DockingTab), which is misleading: the container code is fine.

## Context

Happens when a form hosts **UserControls whose constructors or `Load` handlers do real work** (service calls, repository/presenter construction). Applies to any WinForms project; the container in the error message varies.

## Root cause

The VS designer instantiates **nested controls for real** inside the `devenv` process — their actual constructors run, and their `Load` events **do fire at design time** (the designer sites and shows them). If any of that throws (e.g. a gRPC/DB call like `CallService.GetTable(...)` with no backend available inside VS), the designer's error recovery substitutes placeholder controls and re-parents things; strict containers (C1DockingTab accepts only C1DockingTabPage children) then reject the placeholder → the misleading dialog, far from the real cause.

Three compounding traps:

1. **`Control.DesignMode` is unreliable** — it is `false` inside constructors, and stays `false` for controls *nested inside another UserControl* (only the directly-designed control gets a designer site). The reliable check is `LicenseManager.UsageMode == LicenseUsageMode.Designtime`, and it is only valid **during construction** — capture it into a field.
2. **`Load` fires at design time** for child controls placed on a designed surface.
3. **The designer serializes public read-write properties** of the UserControl into the *host* form's Designer.cs (e.g. a stray `this._myControl.RootId = 0;`). Runtime-only properties must be hidden from serialization.

## Fix pattern

```csharp
public partial class MyControl : UserControl
{
    // LicenseManager.UsageMode is only reliable during construction; DesignMode is
    // false for controls nested inside another UserControl, so it can't be used here.
    private readonly bool _isDesignTime = LicenseManager.UsageMode == LicenseUsageMode.Designtime;

    public MyControl()
    {
        InitializeComponent();
        if (!_isDesignTime)
        {
            _repository = new SomeRepository();      // real work only at runtime
            new SomePresenter(this, _repository);
        }
    }

    private void MyControl_Load(object sender, EventArgs e)
    {
        if (_isDesignTime) return;                   // Load fires inside the VS designer too
        var dt = CallService.GetTable("...");        // never reaches the designer
    }

    [Browsable(false)]
    [DesignerSerializationVisibility(DesignerSerializationVisibility.Hidden)]
    public int RootId { get; set; }                  // runtime state - keep out of the designer
}
```

If event handlers delegate to objects created only at runtime, null-guard them (`_controller?.Handle(...)`) so designer-time event firing can't NRE.

## Checklist for every new UserControl

- [ ] `_isDesignTime` field via `LicenseManager.UsageMode`, captured at construction.
- [ ] Constructor: repositories/presenters/services created only when `!_isDesignTime`.
- [ ] `Load` handler: early-return at design time before any service call.
- [ ] Every public runtime-state property: `[Browsable(false)]` + `[DesignerSerializationVisibility(Hidden)]`.
- [ ] Handlers that use runtime-only objects: null-conditional (`?.`).
- [ ] After fixing: check the host form's Designer.cs for already-serialized junk property assignments and delete them.

## How to recognize it again

A designer-only error naming a container type you barely touched, on a form that hosts custom UserControls → suspect the UserControls' design-time behavior first, not the container's Designer.cs.
