# C1ToolBar buttons don't gray out when a parent container is disabled

**Date:** 2026-07-27 · **Project:** CWATIS · **Component:** C1.Win.C1Command.C1ToolBar (v4.8.20231.596)

## Symptom

A `C1ToolBar` sits inside a panel that gets `Enabled = false`. Mouse/keyboard input to the toolbar **is** blocked (standard WinForms disabled-ancestor behavior), but the buttons keep their normal, enabled look — users report "the toolbar is not disabled". Runtime only; no error.

## Root cause

`C1ToolBar` renders each button from the state of its linked **`C1Command`** (`C1CommandLink.Command.Enabled`), not from the control hierarchy's effective `Enabled`. `C1Command` objects live in a `C1CommandHolder`, completely outside the control tree, so disabling any ancestor panel changes nothing in how the links are painted. Verified by reflection: `C1ToolBar` does not override `Enabled` or hook repaint to it; `C1Command.Enabled` is the property the renderer consumes.

The same applies to grids: a `C1FlexGrid`/`SmartGrid` under a disabled parent is input-blocked but keeps its normal appearance; only setting the grid's **own** `Enabled = false` produces the disabled look.

## Fix

Container-level `Enabled = false` is still right for input blocking and child-state preservation, but for visual feedback additionally disable the commands, snapshotting their prior state (some may be permission-disabled already):

```csharp
private readonly Dictionary<C1Command, bool> _toolbarCommandStates = new Dictionary<C1Command, bool>();

private void SetToolbarCommandsGrayed(bool grayed, params C1ToolBar[] toolbars)
{
    foreach (var toolbar in toolbars)
        foreach (C1CommandLink link in toolbar.CommandLinks)
        {
            if (link.Command == null) continue;
            if (grayed)
            {
                if (!_toolbarCommandStates.ContainsKey(link.Command))
                    _toolbarCommandStates[link.Command] = link.Command.Enabled;
                link.Command.Enabled = false;
            }
            else if (_toolbarCommandStates.TryGetValue(link.Command, out var wasEnabled))
                link.Command.Enabled = wasEnabled;
        }
}
```

Precedent: `CWATIS/Forms/Contracts/View/ContractsForm.cs` (edit-mode lock).

## Checklist / rule

- Disabling a parent panel blocks input to C1 toolbars/grids but does **not** gray them.
- To make a `C1ToolBar` look disabled, disable its `C1Command`s (or the specific commands), never rely on container `Enabled`.
- To make a grid look disabled, set the grid's own `Enabled = false` in addition to (or instead of) the container's.
- When bulk-disabling commands, snapshot and restore their previous `Enabled` — a blanket re-enable can resurrect permission-banned buttons.
- Related: [[c1dockingtab-disable-tab-switching]] — same family of "C1 renders from command/page state, not control state" quirks.
