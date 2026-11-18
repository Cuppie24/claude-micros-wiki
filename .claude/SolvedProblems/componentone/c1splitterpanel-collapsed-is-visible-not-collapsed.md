# C1SplitterPanel.Collapsed is just `Visible` — `Collapsed == true` means the panel is OPEN

Found 2026-10-09 in CWATIS (`WhInventoryReportsForm`), C1.Win.C1SplitContainer 4.8.20231.596.

## Symptom
"Expand the panel if it is collapsed" written as
`if (panel.Collapsed) panel.Collapsed = false;` did the opposite: pressing the button **hid an already open panel** and
never opened a collapsed one. No error, nothing in the name suggests an inversion.

## Root cause
The XML doc says only "Gets or sets whether the panel is collapsed (duplicates the Visible property)". Confirmed by reading
the IL (reflection, no license needed): `get_Collapsed` returns a private bool that `OnVisibleChanged`/`SetVisibleCore` write
(a mirror of visibility, so `true` for an open panel), and `set_Collapsed(value)` calls `Control.set_Visible(value)`.
So `Collapsed = false` **hides** the panel and `Collapsed = true` shows it.

## Fix
Do not use `Collapsed` at all; use `Visible`, which means what it says:
```csharp
if (!splitPanelRight.Visible) splitPanelRight.Visible = true;   // expand
```
(assumes a collapsed C1 panel reports `Visible == false`, which the IL above implies; verify visually once.)

## Checklist
- Any C1 property documented as "duplicates X" deserves a check of which direction it duplicates.
- Reading IL without instantiating the control works around the C1 design-time license: load `C1.Win.4.8.dll` first, then
  `Assembly.LoadFrom` the control assembly, `GetMethodBody().GetILAsByteArray()`, resolve tokens via `Module.ResolveMember`.
