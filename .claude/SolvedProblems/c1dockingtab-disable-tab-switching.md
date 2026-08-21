# C1DockingTab: disable tab switching without hiding tabs

**Date:** 2026-07-27 · **Project:** CWATIS · **Component:** C1.Win.C1Command.C1DockingTab (v4.8.20231.596)

## Symptom

Need to lock a `C1DockingTab` during an edit mode: tabs must stay **visible** but not switchable. The obvious hack is `c1DockingTab1.ShowTabs = false`, which hides the tab headers entirely — bad UX. There is no per-tab "disabled" property to reach for.

## Root cause / facts (verified by reflection on the assembly)

- `C1DockingTabPage` has **no** `Enabled`/`AllowSelect`-style property on the tab header. Its declared properties are only: `Text`, `CaptionText`, `TabVisible`, `CaptionVisible`, `Image`, `ImageIndex`, `ToolTipText`, `IsSelected`, `CloseTabBehavior`, `TabBounds`, colors (`TabBackColor`, `TabForeColor`, `TabBackColorSelected`, `TabForeColorSelected`), `IsHot`, `IsPressed`. Setting the page's inherited `Control.Enabled` disables the page *content*, not header selection.
- `C1DockingTab` **does** expose a cancelable event: `SelectedIndexChanging` of delegate type `C1.Win.C1Command.SelectedIndexChangingEventHandler`, whose args `SelectedIndexChangingEventArgs` derive from `System.ComponentModel.CancelEventArgs` (has `Cancel`). This is the built-in mechanism to block switching.

## Fix

Subscribe in the Designer:

```csharp
this.c1DockingTab1.SelectedIndexChanging += new C1.Win.C1Command.SelectedIndexChangingEventHandler(this.c1DockingTab1_SelectedIndexChanging);
```

Handler cancels while the lock condition holds:

```csharp
private void c1DockingTab1_SelectedIndexChanging(object sender, C1.Win.C1Command.SelectedIndexChangingEventArgs e)
{
    e.Cancel = _isInEditMode;
}
```

Tabs remain visible and clickable-looking; clicks simply don't switch. For visual feedback, optionally gray the inactive headers via `C1DockingTabPage.TabForeColor` while locked.

Precedent: `CWATIS/Forms/Contracts/View/ContractsForm.cs` (`c1DockingTab1_SelectedIndexChanging`, guarded by the edit-mode flag `IsDisableControlGroupBox`).

## Checklist / rule

- Don't use `ShowTabs = false` to "lock" a C1DockingTab — that hides the headers; cancel `SelectedIndexChanging` instead.
- Don't look for a per-page disable property on `C1DockingTabPage` — it doesn't exist (verified v4.8.20231.596).
- Related WinForms fact used alongside: disabling a **container** (`panel.Enabled = false`) blocks/grays all children **without overwriting each child's own `Enabled`/`ReadOnly` flags** — re-enabling the container restores every child to its exact pre-existing state. Prefer this over recursive per-child disabling with a saved-state dictionary.
