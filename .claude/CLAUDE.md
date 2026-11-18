# Shared knowledge base (all projects under this folder)

This folder (`.claude/` next to this file, git repo `claude-micros-wiki`) holds knowledge shared by every project: the
micros libraries (GlobalForms, Refers, SmartGrid, CallService, …), ComponentOne/WinForms, MySQL, build/CI and tooling.
Project-only knowledge lives in that project's own `CLAUDE.md` and `.claude/docs/` (e.g. `micros/CWATIS`).
Find this folder by locating this file — never hardcode its path.

## Layout

```
.claude/
  CLAUDE.md          rules + index (this file, always loaded)
  Topics/            convention / how-to pages, read on demand
  SolvedProblems/    one file per solved problem, grouped by area:
    componentone/ winforms/ micros-libraries/ build-and-tooling/ mysql/ windows-native/ web/
```

## Recording solved problems and gotchas

When a problem needed real investigation (or its error pointed away from the cause), or you hit a library/tool gotcha,
write it down before finishing the task:

1. **Where** — knowledge useful beyond one project goes here; knowledge about one project's own code goes to that
   project's `.claude/docs/` and its `CLAUDE.md` index.
   - Gotcha inside an area that has a topic page → append a `## Gotcha: …` section to that page.
   - Otherwise → new file `SolvedProblems/<area>/<short-kebab-topic>.md` (pick the closest area folder).
2. **Index it** — add one line to the index below (or the project's index): link + one sentence containing the
   verbatim symptom text, so it can be found by searching the error. An unindexed file is invisible.
3. **Content** — *Symptom* (exact error/behaviour, where it shows: designer / build / runtime), *Root cause* (the
   mechanism; say so when the error points at the wrong component), *Fix* (minimal code), *How to recognise / avoid*.
   Reusable, no task-specific details, dated, project named. Say how a claim was verified (reflection, docs, test).
4. Topic pages: the rule, wrong way vs right way (short code), why, precedents.
5. Commit the change in this repo (and in the project repo for project docs).

## Working with this user

- C#/.NET (WinForms) developer. When explaining another language (e.g. Python in the LangChain prototypes), contrast it
  with the C# equivalent; skip programming basics.
- Status write-ups (migration/refactor logs, progress notes): keep **Finished** and **Not finished / in progress** in
  separate lists — never one blended sentence.

## Ongoing cross-repo work

- **SubDivId migration** (bookkeep local `subdivisions` ids → `global_db.rfr_subdivisions.SDId`, table by table):
  tracked in `micros/bookkeep/Задача - Перенос SubDivId на global_db.rfr_subdivisions.md`, GitLab issue
  `microsdvl/bookkeep#1`, and the local untracked notes `micros/subdivisions-refactor.md` (its "Status per table" list).
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

```sql
DROP PROCEDURE IF EXISTS cwatis.WhInventoryReportLine_Add;
create definer = bookkeep@`%` procedure cwatis.WhInventoryReportLine_Add(...)
BEGIN ... END;
```

**Shared libraries are ours** — `MC_GlobalForms`, `Refers`, `SmartGrid`, `Profile`, … are developed by the same team
(sources in sibling repos under `micros/`, e.g. `micros/globalforms`). When one lacks something a task needs, say so
and ask whether to extend it; don't silently swap in a different component. A "can't do X" finding may be outdated
the same day — re-check the live source.

**GitLab** — issues/work items on `gitlab.micros.uz` are written in **Russian**. CLI usage: [GlabCliUsage.md](Topics/GlabCliUsage.md).

**Publishing GlobalForms** — bump the patch number in `GlobalForms\Properties\AssemblyInfo.cs` (both attributes),
push to `development`; CI publishes `MC_GlobalForms` to the internal feed. Consumers bump their reference by hand.
Version-pin traps with `Refers.V2`: [GlobalFormsPublishing.md](Topics/GlobalFormsPublishing.md).

**Wiki** — when asked to save to a project wiki (`Wiki`/`<Project>Wiki` in the project root, unless the project
`CLAUDE.md` says otherwise): update an existing page before creating one; thematic subfolders; readable filenames in
the project language; facts about the system only (no task notes); link pages with `[[Page Name]]`.

## Topics

| Page | Read when |
|---|---|
| [SmartGridConventions.md](Topics/SmartGridConventions.md) | Any SmartGrid work: DataSource, in-place CRUD on `C1DataCollectionBindingList`, «Информация о записи», footers, model unwrap, per-column permissions via `Tag` JSON, many gotchas. Standards for every new grid form: clear designer placeholder rows after `InitializeComponent()` (`Rows.Count = Rows.Fixed + Footers.Descriptions.Count`); a read-only checkbox column draws a box only for `true` |
| [SmartGridReference.md](Topics/SmartGridReference.md) | SmartGrid component features (RU): multi-row headers, context menu API, tooltips, font zoom, default sort, footer formats, tree mode |
| [CallServiceUsage.md](Topics/CallServiceUsage.md) | Talking to MySQL from WinForms clients: `CallProcedureDT`/`CallProcedureNonQuery`/`GetTable`/`GetValueFromFunc`/`UniSaver`, binding and null rules, which assembly actually runs |
| [RecordChangeLog.md](Topics/RecordChangeLog.md) | `log_db` / `add_rec_info` per schema, batch logging, `Main.RecInfoView` menu item, logger limitations |
| [GlobalFormsPublishing.md](Topics/GlobalFormsPublishing.md) | Releasing `MC_GlobalForms`; consumer version bumps |
| [GlabCliUsage.md](Topics/GlabCliUsage.md) | `glab` install/auth on the self-hosted instance, multi-line args from PowerShell 5.1, priority labels |

## Solved problems index

### componentone
- [c1dockingtab-disable-tab-switching](SolvedProblems/componentone/c1dockingtab-disable-tab-switching.md) — no per-page disable on C1DockingTab: cancel `SelectedIndexChanging` instead of `ShowTabs = false`.
- [c1toolbar-not-graying-when-parent-disabled](SolvedProblems/componentone/c1toolbar-not-graying-when-parent-disabled.md) — C1ToolBar paints from `C1Command.Enabled`; disable the commands (snapshot/restore), not just the container.
- [c1mainmenu-no-right-alignment](SolvedProblems/componentone/c1mainmenu-no-right-alignment.md) — no right-align property on `C1CommandLink`; use a separate `C1ToolBar` anchored Top|Right.
- [c1splitterpanel-collapsed-is-visible-not-collapsed](SolvedProblems/componentone/c1splitterpanel-collapsed-is-visible-not-collapsed.md) — `C1SplitterPanel.Collapsed` is really `Visible` (`Collapsed == true` means open); use `Visible`.
- [c1input-text-assign-hidden-parent](SolvedProblems/componentone/c1input-text-assign-hidden-parent.md) — "Cannot assign text to readonly, disabled or hidden C1Input control" when the parent panel/tab is hidden; set `TextDetached = true`.
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
- [msubjectdatavirtualizingform-type-filter-and-profile](SolvedProblems/micros-libraries/msubjectdatavirtualizingform-type-filter-and-profile.md) — forcing the "Тип" filter overwrites the user's saved choice; pass `(Profile.IKeysValueManager)null`.
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
- [gitlab-ci-shallow-clone-breaks-commit-based-versioning](SolvedProblems/build-and-tooling/gitlab-ci-shallow-clone-breaks-commit-based-versioning.md) — wrong/decreasing computed versions in CI; set `GIT_DEPTH: 0` and fetch tags.
- [git-bash-mangles-msbuild-slash-flags](SolvedProblems/build-and-tooling/git-bash-mangles-msbuild-slash-flags.md) — `MSB1008: only one project can be specified`: Git Bash rewrites `/flag` args, use PowerShell. Also the Bash tool collapses doubled backslashes even in quoted heredocs.
- [github-readme-image-stale-raw-cdn-and-bom-corruption](SolvedProblems/build-and-tooling/github-readme-image-stale-raw-cdn-and-bom-corruption.md) — README image broken after a fix: BOM-corrupted PNG, then a stale raw CDN object; wait and re-probe.
- [git-bash-sed-in-place-strips-crlf](SolvedProblems/build-and-tooling/git-bash-sed-in-place-strips-crlf.md) — Git Bash `sed -i` rewrites the whole file with LF even with no match (`core.autocrlf` hides it); use Edit or PowerShell.

### mysql
- [mysql-in-param-mistaken-for-output](SolvedProblems/mysql/mysql-in-param-mistaken-for-output.md) — `SELECT … INTO _param` on an `IN` param never reaches the caller; make it `INOUT`.
- [find-in-set-json-array-drops-first-last-element](SolvedProblems/mysql/find-in-set-json-array-drops-first-last-element.md) — `FIND_IN_SET` fed `"[1,2,3]"` misses the first and last id; pass `string.Join(",", ids)`.
- [pomelo-mysql-ignores-totable-schema](SolvedProblems/mysql/pomelo-mysql-ignores-totable-schema.md) — `Table '<db>.<x>' doesn't exist`: Pomelo ignores the schema in `ToTable`; bridge with a view. Then `Unknown column 'id'` with snake-case naming.
- [mysql-on-duplicate-key-burns-auto-increment](SolvedProblems/mysql/mysql-on-duplicate-key-burns-auto-increment.md) — get-or-create via `INSERT … ON DUPLICATE KEY` burns an id per call; SELECT first.
- [mysql-procedure-literal-collation-from-creating-connection](SolvedProblems/mysql/mysql-procedure-literal-collation-from-creating-connection.md) — `Illegal mix of collations (utf8mb4_unicode_ci,IMPLICIT) and (utf8mb4_0900_ai_ci,IMPLICIT)`: literals/params take the creating session's charset; declare `CHARSET … COLLATE …` on every compared string param.
- [mysql-tree-nested-counts-upward-cte](SolvedProblems/mysql/mysql-tree-nested-counts-upward-cte.md) — "count including descendants" via `FIND_IN_SET` on a path is quadratic; carry counts up `ParentId` in one recursive CTE (`UNION ALL` + `SUM`).

### windows-native
- [tauri-window-center-work-area-vs-monitor-size](SolvedProblems/windows-native/tauri-window-center-work-area-vs-monitor-size.md) — vertical jump on resize: `center()` uses the work area, `monitor.size()` the full screen.
- [composition-rounded-rect-round-only-some-corners](SolvedProblems/windows-native/composition-rounded-rect-round-only-some-corners.md) — round only some corners: oversize the rounded rect, `ShapeVisual` clips it.
- [composition-animated-values-not-readable-from-app](SolvedProblems/windows-native/composition-animated-values-not-readable-from-app.md) — animated Composition values can't be read back; mirror the animation app-side.
- [win32-overlay-httransparent-does-not-pass-clicks-cross-process](SolvedProblems/windows-native/win32-overlay-httransparent-does-not-pass-clicks-cross-process.md) — `HTTRANSPARENT` doesn't pass clicks to other processes; split render and hit-test windows.

### web
- [css-dark-mode-media-query-not-gated-by-forced-theme](SolvedProblems/web/css-dark-mode-media-query-not-gated-by-forced-theme.md) — near-invisible headings with OS-dark + app-forced-light; declare `color` on the component class.
- [oxlint-react-refs-taints-whole-object-property](SolvedProblems/web/oxlint-react-refs-taints-whole-object-property.md) — oxlint `react(refs)` flags every property once one is a `ref`; destructure the ref.
- [outline-postgres-sslmode-required](SolvedProblems/web/outline-postgres-sslmode-required.md) — self-hosted Outline SSL error against plain Postgres; `PGSSLMODE=disable`.
- [outline-self-hosted-first-team-requires-oauth-oidc](SolvedProblems/web/outline-self-hosted-first-team-requires-oauth-oidc.md) — `providers: []` on a fresh Outline: the first team needs OAuth/OIDC (use dex locally).
