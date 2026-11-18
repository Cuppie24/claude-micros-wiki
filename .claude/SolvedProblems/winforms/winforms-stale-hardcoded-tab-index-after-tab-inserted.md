# WinForms: a hardcoded tab-index literal (`SelectedIndex == 2`) goes silently wrong after a tab is inserted before it

**Symptom** — On `ContractsForm`, sitting on the "Доп соглашения" (additional agreements) tab and changing the selected contract row leaves the ДС list (`_dsListControl` inside `_dsLinesTabControl`) showing the previous contract's data — only the tab-header count updates. Switching *away* and back to the tab does reload it. No error. Runtime only.

**Root cause** — The only call that reloads the ДС list is `RefreshDSTabAsync()` (sets `_dsLinesTabControl.RootId`, calls `RefreshDSListAsync()`). In `_masterGrid_AfterSelChange` and in `RunTasks()` it was gated on a **hardcoded literal**:

```csharp
if (c1DockingTab1.SelectedIndex == 2) await RefreshDSTabAsync(token);
else await UpdateDSTabHeaderAsync(token);   // header count only, no list reload
```

The form already defines `private const int DSTabIndex = 3;` and uses it correctly elsewhere (`c1DockingTab1_SelectedIndexChanged`, `EditMaster.Enabled = ... != DSTabIndex`). A layout refactor had inserted a "Реквизиты и печать" tab, so the designer's child order became

```
c1DockingTab1.Controls.Add(c1DockingTabPage2);  // idx 0  "Главная"
c1DockingTab1.Controls.Add(c1DockingTabPage4);  // idx 1  "Реквизиты и печать"  <-- inserted
c1DockingTab1.Controls.Add(c1DockingTabPage1);  // idx 2  "Условия контракта"
c1DockingTab1.Controls.Add(c1DockingTabPage3);  // idx 3  "Доп соглашения"
```

The ДС tab moved 2 → 3. The `== DSTabIndex` call sites followed; the two `== 2` literals did not, so on the ДС tab the row-change handler took the `else` branch. (The *other* `== 2` checks in the same file — `PopulateFormExample`, `DisableAddedInfoControl` — legitimately refer to "Условия контракта" at index 2 and were correct to leave alone.)

**Fix** — Replace the stale literals with the existing constant:

```csharp
if (c1DockingTab1.SelectedIndex == DSTabIndex) await RefreshDSTabAsync(token);
```

**Checklist / rule**

- Tab / page / step index is `designer child-add order`. Inserting one page silently invalidates **every** hardcoded index at or after it, with no compile error and no runtime error — just wrong branches.
- If the file defines a `const` for the index (`DSTabIndex`, `DSTabIndex = 3`), grep the whole file for bare integer comparisons against the same `SelectedIndex`/`SelectedTab`/step variable — a partial migration (some sites use the const, some use `2`) is the tell.
- After any designer tab/page reorder, search the code-behind for `SelectedIndex ==`, `SelectedIndex !=`, `TabPages[`, `SelectedTab ==` and re-verify each against the new order.
- Same class of bug for `TabControl`, `C1DockingTab`, wizard step enums, `Panel` z-order — anything where a position number is written as a literal in logic.

Found in CWATIS, 2026-08-27, `ContractsForm` (`_masterGrid_AfterSelChange`, `RunTasks` — two `SelectedIndex == 2` → `== DSTabIndex`; introduced by commit "Refactor ContractsForm layout and controls" which added the "Реквизиты и печать" tab).
