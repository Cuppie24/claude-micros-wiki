# `MSubjectDataVirtualizingForm`: forcing the "Тип" filter overwrites the user's remembered choice

Found 2026-11-10 in CWATIS (warehouse goods owners form). Verified against MC_GlobalForms source 1.1.7.16
(`GlobalForms/Forms/Subjects/MSubjectDataVirtualization/MSubjectDataVirtualizingForm.cs`) and by disassembling the 1.1.7.24 DLL
(same `Load` / `SetSelectedSubjType` / `TYPE_SUBJCombo_SelectedValueChanged` sequence).

## Symptom

No error. After opening the counterparty picker from a screen that forces a type (`ModeEdit = false; SubjType = 0`), the same
picker opened from *other* screens starts on that forced type instead of the user's own last choice. Opening it with
`ShowSelect(text)` likewise resets the user's remembered type to "Все".

## Root cause

The profile manager passed to the constructor (`new MSubjectDataVirtualizingForm(Main.ProfileKeysValueManager)`) is used for
exactly one thing: remembering the "Тип" filter (profile key `_SubjTypeFilter`, value = combo `SelectedIndex`).

`Load` (named `MSubjectDataVirtualizinfgForm_Load`, typo in the library):
1. if the name filter is empty: `TYPE_SUBJCombo.SelectedIndex = <profile value>`, then `SetSelectedSubjType()` — which, when
   `!ModeEdit && SubjType > -1`, sets `SelectedValue = SubjType` and **disables** the combo;
   if the name filter has text (`ShowSelect(text)`): `SelectedIndex = 0` ("Все"), no forcing.
2. Every value change runs `TYPE_SUBJCombo_SelectedValueChanged` → refresh → `AddOrReplaceProfileKey("_SubjTypeFilter", SelectedIndex)`.

So forcing the type (or the "Все" from `ShowSelect`) is saved into the user's profile as if they had picked it.

## Fix

When the caller fixes the type itself (or searches by text), don't give the form the profile manager. With `null`,
`GetSelectedProfileValue` returns `""` and `AddOrReplaceProfileKey` is a no-op; the forcing still works (`SubjType` + `ModeEdit = false`).

```csharp
using (var form = new MSubjectDataVirtualizingForm((Profile.IKeysValueManager)null)) // cast required, see below
{
    form.ModeEdit = false;
    form.SubjType = 0;          // ЮЛ; the combo is set and disabled in Load
    form.MultiSelect = true;
    form.Owner = this;
    if (form.ShowDialog() != DialogResult.OK) return;
    ...
}
```

`new MSubjectDataVirtualizingForm()` does **not** compile: `CS0121` ambiguous call between the `(IKeysValueManager = null)` and the
`(ISubjMediator, IMapper<Subject, SubjDTO>, bool)` constructors (both all-optional). Pass the typed `null`.

## Checklist

- Picker type must be fixed? `ModeEdit = false` + `SubjType = <id>` (-1 = not forced; 0 ЮЛ, 1 ФЛ, 2 Ю/Ф, 3 SPC) **and** no profile manager.
- `ShowSelect(text)` always shows "Все" (no way to force a type there without changing the library).
- Precedent: `CWATIS/Features/Warehouse/GoodOwners/View/WhGoodOwnersForm.PickSubjects` / `ShowSubject`.

## Gotcha: grid doesn't match the "Тип" filter (dropped refresh)

Found 2026-11-24 in CWATIS (warehouse invoice counterparty picker), confirmed in MC_GlobalForms source
(`MSubjectDataVirtualizingForm.cs`, `Load` / `OnRefresh` / `TYPE_SUBJCombo_SelectedValueChanged`).

**Symptom:** the "Тип" combo shows «ЮЛ» (or whatever was picked) but the grid lists other types too; pressing «Обновить»
once more fixes it. Happens on open with a forced `SubjType`, and sometimes after the user changes the filter.

**Root cause:** `OnRefresh` starts with `if (IsRefresh) return;` — a refresh requested while another one is in flight is
**dropped**, not queued (the `cts.Cancel()` inside it never gets a chance to cancel anything). `FilterDTO` is read when the
query starts, so the running query uses the old filter. On open, `Load`:
1. `TYPE_SUBJCombo.SelectedIndex = <profile value>` (with no profile: `0` = «Все») -> `SelectedValueChanged` -> `OnRefresh`
   starts with «Все» and awaits the DB;
2. `SetSelectedSubjType()` sets `SelectedValue = SubjType` (ЮЛ) -> `SelectedValueChanged` -> `OnRefresh` -> **dropped**;
3. `await OnRefresh()` at the end of `Load` -> **dropped** too.
The grid gets the «Все» result while the combo shows ЮЛ. Any filter change made during a running refresh is lost the same way.

**Fix (library, not done yet):** in `OnRefresh`, when `IsRefresh` is set, remember a pending request and re-run once in
`finally` (or cancel the running query and restart — the `cts` is already there). Consumers can't work around it cleanly:
the refresh command and `OnRefresh` are private.
