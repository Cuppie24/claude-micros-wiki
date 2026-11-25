# Micros / WinForms knowledge base (all projects under this folder)

Repo `claude-micros-wiki`. Loaded for every project in `projects/` next to this `.claude/` folder (git-ignored here;
each project is its own repo). Covers the company stack: WinForms, ComponentOne, our component libraries (GlobalForms,
Refers, SmartGrid, CallService, Profile, …), company MySQL rules and CI. Universal rules (documenting solutions, the
user's preferences, wiki rules, shell gotchas, general MySQL/tool problems) are in the global `claude-global` repo,
imported from `~/.claude/CLAUDE.md`. Project-only knowledge lives in the project's own `CLAUDE.md` and `.claude/docs/`.
Find this folder by locating this file — never hardcode its path.

## Layout

```
.claude/
  CLAUDE.md          rules + index (this file, always loaded)
  Topics/            convention / how-to pages, read on demand
  SolvedProblems/    one file per solved problem: componentone/ winforms/ micros-libraries/ build-and-tooling/
```

## Where solved problems go (procedure: `document-solution` skill)

- About one project's own code → that project's `.claude/docs/`, indexed in its `CLAUDE.md`.
- About the company stack → here: a `## Gotcha: …` section of the matching topic page, or
  `SolvedProblems/<area>/<topic>.md` plus one index line below.
- About a general tool / language / OS / plain MySQL → `claude-global`.

## Ongoing cross-repo work

- **SubDivId migration** (bookkeep local `subdivisions` ids → `global_db.rfr_subdivisions.SDId`, table by table):
  tracked in `projects/bookkeep/Задача - Перенос SubDivId на global_db.rfr_subdivisions.md`, GitLab issue
  `microsdvl/bookkeep#1`, and the local notes `projects/subdivisions-refactor.md` next to this `.claude/` folder (not in any repo; its "Status
  per table" list).
  Re-read them before touching any `SubDivId`; "procedures already updated" claims have been wrong before — verify
  each procedure. Background: [cwatis-bookkeep-reference-id-mismatch](SolvedProblems/micros-libraries/cwatis-bookkeep-reference-id-mismatch.md).

## Rules that always apply

**SQL procedures**
- Always schema-qualify the procedure name (`CREATE`/`DROP`/calls/grants). Never a bare name.
- Always ``definer = bookkeep@`%` ``, whatever the schema.
- Procedures that insert/update/delete entity rows log every changed row via the table schema's `add_rec_info`
  (`cwatis.add_rec_info(_Id, 'table', 'CREATE'|'EDIT'|'DELETE', _UserId)`), never `INSERT INTO log_db.*` directly.
  Take `IN _UserId int`; save `LAST_INSERT_ID()` before logging; log `DELETE` before deleting, inside the transaction;
  log `EDIT` only for rows that change. Write paths that can't log (`CallService.DeleteLines`, `UpdateCell`, raw SQL)
  get replaced by a logging procedure. Every grid of DB entities has the «Информация о записи» menu item.
  Details: [RecordChangeLog.md](Topics/RecordChangeLog.md).

**Every new or changed form** must pass the testers' UI checklist: [FormUiChecklist.md](Topics/FormUiChecklist.md).

```sql
DROP PROCEDURE IF EXISTS cwatis.WhInventoryReportLine_Add;
create definer = bookkeep@`%` procedure cwatis.WhInventoryReportLine_Add(...)
BEGIN ... END;
```

**Shared libraries are ours** — `MC_GlobalForms`, `Refers`, `SmartGrid`, `Profile`, … are developed by the same team
(sources in sibling repos under `projects/`, e.g. `projects/globalforms`). When one lacks something a task needs, say so
and ask whether to extend it; don't silently swap in a different component. A "can't do X" finding may be outdated
the same day — re-check the live source.

**GitLab** — issues/work items on `gitlab.micros.uz` are written in **Russian**. CLI usage: [GlabCliUsage.md](Topics/GlabCliUsage.md).

**Publishing GlobalForms** — bump the patch number in `GlobalForms\Properties\AssemblyInfo.cs` (both attributes),
push to `development`; CI publishes `MC_GlobalForms` to the internal feed. Consumers bump their reference by hand.
Version-pin traps with `Refers.V2`: [GlobalFormsPublishing.md](Topics/GlobalFormsPublishing.md).

## Topics

| Page | Read when |
|---|---|
| [FormUiChecklist.md](Topics/FormUiChecklist.md) | Building, changing or finishing any form: the testers' mandatory UI checklist (МВХ) — standard buttons with tooltips, window title, filter zone, max height 600 px, number/text alignment, Tab order, grid context menu «Информация о записи», «Всего:» footer, zeros blank, sorting, resizable columns |
| [SmartGridConventions.md](Topics/SmartGridConventions.md) | Any SmartGrid work: DataSource, in-place CRUD on `C1DataCollectionBindingList`, «Информация о записи», footers, model unwrap, per-column permissions via `Tag` JSON, many gotchas. Standards for every new grid form: clear designer placeholder rows after `InitializeComponent()` (`Rows.Count = Rows.Fixed + Footers.Descriptions.Count`); a read-only checkbox column draws a box only for `true`. Footer + last-row crash («Индекс за пределами диапазона… index» in `C1FlexGrid.DrawRow`): an inline edit on a `C1DataCollectionBindingList` grid makes C1 replace the row in the list itself |
| [SmartGridReference.md](Topics/SmartGridReference.md) | SmartGrid component features (RU): multi-row headers, context menu API, tooltips, font zoom, default sort, footer formats, tree mode |
| [CallServiceUsage.md](Topics/CallServiceUsage.md) | Talking to MySQL from WinForms clients: `CallProcedureDT`/`CallProcedureNonQuery`/`GetTable`/`GetValueFromFunc`/`UniSaver`, binding and null rules, which assembly actually runs |
| [UserPermissions.md](Topics/UserPermissions.md) | Tying a form, toolbar commands or grid columns to rights (`UserPermissionsControl`). Preferred: everything declared in the designer (form `Tag`, command `UserData`, editable columns + grid `Tag`) and enforced by `Initialize()`, no rights checks in code. `right_objects`/`right_values` tables, form `Tag`, command `UserData` (`canApprove`), grid `Tag` JSON, `PermissionManager.Initialize()` in `Load`; a JSON name not matching `Right_Object_Code` silently disables buttons for everyone |
| [RecordChangeLog.md](Topics/RecordChangeLog.md) | `log_db` / `add_rec_info` per schema, batch logging, `Main.RecInfoView` menu item, logger limitations |
| [GlobalFormsPublishing.md](Topics/GlobalFormsPublishing.md) | Releasing `MC_GlobalForms`; consumer version bumps |
| [GlabCliUsage.md](Topics/GlabCliUsage.md) | `glab` install/auth on the self-hosted instance, multi-line args from PowerShell 5.1, priority labels |

## Solved problems index

### componentone
- [c1dockingtab-disable-tab-switching](SolvedProblems/componentone/c1dockingtab-disable-tab-switching.md) — no per-page disable on C1DockingTab: cancel `SelectedIndexChanging` instead of `ShowTabs = false`.
- [c1toolbar-not-graying-when-parent-disabled](SolvedProblems/componentone/c1toolbar-not-graying-when-parent-disabled.md) — C1ToolBar paints from `C1Command.Enabled`; disable the commands (snapshot/restore), not just the container.
- [c1mainmenu-no-right-alignment](SolvedProblems/componentone/c1mainmenu-no-right-alignment.md) — no right-align property on `C1CommandLink`; use a separate `C1ToolBar` anchored Top|Right.
- [c1splitterpanel-collapsed-is-visible-not-collapsed](SolvedProblems/componentone/c1splitterpanel-collapsed-is-visible-not-collapsed.md) — `C1SplitterPanel.Collapsed` is really `Visible` (`Collapsed == true` means open); use `Visible`.
- [c1input-text-assign-hidden-parent](SolvedProblems/componentone/c1input-text-assign-hidden-parent.md) — "Cannot assign text to readonly, disabled or hidden C1Input control" when the parent panel/tab is hidden; set `TextDetached = true`. But not on a `C1DateEdit` set via `Value`: there it makes the date not show ("on click nothing happens").
- [c1dropdowncontrol-readonly-hidden-by-base-property](SolvedProblems/componentone/c1dropdowncontrol-readonly-hidden-by-base-property.md) — `ReadOnly = false` via a `C1TextBox` variable doesn't reach `C1DropDownControl`'s own `ReadOnly`; modal/drop-down buttons stay dead.
- [componentone-license-coverage-blocks-build](SolvedProblems/componentone/componentone-license-coverage-blocks-build.md) — "License Period Expired" / `MSB6006: "lc.exe" … -1` after a C1 bump past the license; revert the package, rebuild the DLL from the local SDK.
- [vs-designer-c1supertooltip-filenotfound-files-present](SolvedProblems/componentone/vs-designer-c1supertooltip-filenotfound-files-present.md) — designer "Could not load file or assembly 'C1.Win.C1SuperTooltip.4.8…'" with the DLL present. KNOWN ISSUE, cause unconfirmed.

### winforms
- [winforms-usercontrol-design-time-safety](SolvedProblems/winforms/winforms-usercontrol-design-time-safety.md) — designer breaks ("Only objects of the type C1DockingTabPage can be added…") on UserControls doing live work; nested `DesignMode` is false, use `LicenseManager.UsageMode`.
- [winforms-button-dialogresult-async-click](SolvedProblems/winforms/winforms-button-dialogresult-async-click.md) — designer-set `Button.DialogResult` closes the modal before `async void Click` finishes; set `DialogResult` in code on success.
- [winforms-child-control-setup-constructor-vs-load](SolvedProblems/winforms/winforms-child-control-setup-constructor-vs-load.md) — `Cols["X"]` can be null in the constructor after `InitializeComponent()`; do setup in `Load`.
- [winforms-stale-hardcoded-tab-index-after-tab-inserted](SolvedProblems/winforms/winforms-stale-hardcoded-tab-index-after-tab-inserted.md) — `SelectedIndex == 2` silently wrong after a tab is inserted; grep `SelectedIndex ==`, `TabPages[` after reordering tabs.
- [webview2-virtualhost-mapping-needs-folder-and-content-copy](SolvedProblems/winforms/webview2-virtualhost-mapping-needs-folder-and-content-copy.md) — `DirectoryNotFoundException` (`0x80070003`) from WebView2: the mapping needs a folder, and Content items need `CopyToOutputDirectory`.
- [shared-package-property-type-change-stale-resx-blob](SolvedProblems/winforms/shared-package-property-type-change-stale-resx-blob.md) — `InvalidCastException` from `InitializeComponent()` after retyping a designer property in a shared package; stale `.resx` blob.
- [git-merge-scrambles-resx-grid-entries](SolvedProblems/winforms/git-merge-scrambles-resx-grid-entries.md) — a clean git merge pairs one grid's `ColumnInfo` with another's; check `grep '<data name=' | sort | uniq -d`.
- [vs-designer-reverts-external-resx-edits](SolvedProblems/winforms/vs-designer-reverts-external-resx-edits.md) — external `.resx` edits are reverted by the designer while the form is open in VS; close it first. Also `sed -i` strips CRLF.
- [datarow-itemarray-positional-vs-changed-proc-columns](SolvedProblems/winforms/datarow-itemarray-positional-vs-changed-proc-columns.md) — `dr.ItemArray = new object[]{…}` breaks (`FormatException` on Load) after a DB-only proc column change; write columns by name.

### micros-libraries
- [cwatis-bookkeep-reference-id-mismatch](SolvedProblems/micros-libraries/cwatis-bookkeep-reference-id-mismatch.md) — bookkeep-local reference ids (e.g. `bookkeep.subdivisions`) ≠ global ids; match via `global_db.rfr_subdivisions.SDId`, never by id/code. `X_Local`/`X_Global` columns are stale.
- [customcombo-endaddcustomitem-resets-selection](SolvedProblems/micros-libraries/customcombo-endaddcustomitem-resets-selection.md) — `CustomCombo.EndAddCustomItem` resets the selection and replaces the whole list.
- [cwatis-stored-procedures-not-in-repo](SolvedProblems/micros-libraries/cwatis-stored-procedures-not-in-repo.md) — procedures live only in the DB; ask for the exact `schema.procedure_name`, don't guess.
- [callproceduredt-unknown-params-silently-dropped](SolvedProblems/micros-libraries/callproceduredt-unknown-params-silently-dropped.md) — a dict key the proc doesn't declare is silently ignored; add the `IN` param on the DB too.
- [unisaver-schema-must-be-physical-not-procedure-namespace](SolvedProblems/micros-libraries/unisaver-schema-must-be-physical-not-procedure-namespace.md) — `global_db.UniSaver` needs the real physical schema **and** table: wrong schema = MySQL syntax error, wrong table or non-column JSON key = silently skipped, still "success". Verify via a working proc, not naming.
- [deletelines-dbtabexclude-silently-ignored-and-fk-block-is-silent](SolvedProblems/micros-libraries/deletelines-dbtabexclude-silently-ignored-and-fk-block-is-silent.md) — `CallService.DeleteLines` ignores the exclude arg server-side and FK-blocked ids are skipped with no error.
- [subdivfinances-callservice-and-refers-are-external](SolvedProblems/micros-libraries/subdivfinances-callservice-and-refers-are-external.md) — `CallService.cs` in SubdivFinances is dead code; `Refers.*` come from the `Refers.V2` package, not the empty submodule.
- [ourorgsform-new-outid-is-oursubid](SolvedProblems/micros-libraries/ourorgsform-new-outid-is-oursubid.md) — `OurOrgsForm_New.OutId` is `m_oursubject.OurSubId`, not the `m_subject` id; convert via `refers.m_oursubject`.
- [msubjectdatavirtualizingform-type-filter-and-profile](SolvedProblems/micros-libraries/msubjectdatavirtualizingform-type-filter-and-profile.md) — forcing the "Тип" filter overwrites the user's saved choice; pass `(Profile.IKeysValueManager)null`. Grid not matching the shown "Тип" (fixed by a second «Обновить»): `OnRefresh` drops requests made while one runs.
- [msubjectshortform-subjtype-is-list-index](SolvedProblems/micros-libraries/msubjectshortform-subjtype-is-list-index.md) — `MSubjectShortForm` with `SubjType = 0` opens on «Все»: a non-zero `SubjType` is the «Тип» list index (2 = ФЛ), not `TYPE_SUBJ`; 0 → ЮЛ fixed in 1.1.8.7. Multi-select: `MultiSelect = true` → `ReferIds` / `SelectedItems` (1.1.8.6+).
- [globalforms-categorizedsubdivisionsform-not-a-picker](SolvedProblems/micros-libraries/globalforms-categorizedsubdivisionsform-not-a-picker.md) — `CategorizedSubdivisionsForm` picks only with `SelectMode = true`; otherwise use `MSubDivisionsForm { OnlyView = true }`.
- [globalforms-smartgrid-missing-model-unwrap-helper](SolvedProblems/micros-libraries/globalforms-smartgrid-missing-model-unwrap-helper.md) — reading a bound row's model needs unwrapping `IDataCollectionBindingListItem.Item`; GlobalForms `GetModelRow<T>`.
- [mc-globalforms-smartgrid-extension-method-collision](SolvedProblems/micros-libraries/mc-globalforms-smartgrid-extension-method-collision.md) — `CS0121` ambiguous `GetModelSelectedRow<T>` after a GlobalForms update; qualify the call with the class.
- [smartlookupdropdown-programmatic-text-triggers-search](SolvedProblems/micros-libraries/smartlookupdropdown-programmatic-text-triggers-search.md) — setting `SmartLookUpDropDown.Text` from code starts a search and opens the dropdown; `SelectedRowData` isn't the saved record — keep the id yourself, `TextDetached = true`.
- [treedbgrid-swallowed-background-error-becomes-nre](SolvedProblems/micros-libraries/treedbgrid-swallowed-background-error-becomes-nre.md) — `TreeDBGrid` turns any failed background load (e.g. missing procedure) into a misleading `NullReferenceException` in `DataToNodes`.

### build-and-tooling
- [subdivfinances-refersv2-mcglobalforms-version-mismatch](SolvedProblems/build-and-tooling/subdivfinances-refersv2-mcglobalforms-version-mismatch.md) — designer `FileNotFoundException` for MC_GlobalForms: must match the exact version `Refers.V2` was built against (read its nuspec).
- [globalforms-refersv2-self-referential-mc-globalforms-package](SolvedProblems/build-and-tooling/globalforms-refersv2-self-referential-mc-globalforms-package.md) — installing `Refers.V2` into GlobalForms adds a package of itself → `CS0121` everywhere; remove that reference.
- [system-valuetuple-reference-removed-on-nuget-upgrade](SolvedProblems/build-and-tooling/system-valuetuple-reference-removed-on-nuget-upgrade.md) — System.ValueTuple `FileNotFoundException` after a NuGet bump; restore the `HintPath` reference and `bindingRedirect`.
- [gitlab-ci-nuget-restore-missing-internal-feed](SolvedProblems/build-and-tooling/gitlab-ci-nuget-restore-missing-internal-feed.md) — CI can't restore internal packages: no repo `NuGet.Config`; commit one with the `mars.micros.uz:9109` source.
