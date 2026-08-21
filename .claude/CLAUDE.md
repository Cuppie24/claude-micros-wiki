# Global CLAUDE.md

## Knowledge Base — Solved Problems & Gotchas

Whenever you solve a **non-obvious problem** (the root cause required real investigation, or the error message pointed away from the actual cause) or discover a **library/tool gotcha** (surprising default, misleading API, designer quirk), document it in this directory (`C:\Users\ishnazarov.saidnazar\.claude\`) before finishing the task — future sessions must not re-derive it.

### Where to write

1. **Gotcha in an area covered by an existing topic file** (SmartGridConventions.md, TreeFlexGridConventions.md, …) — append a `## Gotcha: …` section to that file. Do not create a duplicate file.
2. **Standalone solved problem** — create a new file `SolvedProblems\<short-kebab-case-topic>.md`, one file per problem.
3. **Always add a link** in the index below (for new topic files — a new section like the existing ones). The link line in this CLAUDE.md is what makes the knowledge reachable in every session; an unlinked file is invisible.

### What to write in a solved-problem file

- **Symptom** — the exact error message / observed behavior, verbatim (that's what gets searched for), and where it appears (VS designer / runtime / build).
- **Root cause** — the actual mechanism, not just the workaround. Note explicitly when the error message points at the wrong component.
- **Fix** — the concrete change with a minimal code sample.
- **Checklist / rule** — how to avoid or recognize it next time.
- Keep it **reusable**: no task-specific details; project file paths only as precedent references. Date it and name the project where it was found.

### What to write in topic (convention) files

The rule, the wrong way vs the right way (short code samples), why, and codebase precedents — same style as the existing sections in SmartGridConventions.md. Verified facts beat assumptions: note how a claim was confirmed (reflection, local API docs, instrumented test).

### Index of solved problems

- [WinForms UserControl design-time safety](SolvedProblems/winforms-usercontrol-design-time-safety.md) — VS designer breaks ("Only objects of the type C1DockingTabPage can be added…") on forms hosting UserControls that do live work in constructors/Load; `DesignMode` is false for nested controls — use `LicenseManager.UsageMode`.
- [C1DockingTab: disable tab switching without hiding tabs](SolvedProblems/c1dockingtab-disable-tab-switching.md) — no per-page disable property exists; cancel the built-in `SelectedIndexChanging` (`CancelEventArgs`) instead of `ShowTabs = false`; also: disabling a container panel preserves children's own `Enabled`/`ReadOnly` flags for free.
- [C1ToolBar buttons don't gray when parent disabled](SolvedProblems/c1toolbar-not-graying-when-parent-disabled.md) — C1ToolBar paints buttons from `C1Command.Enabled`, not the control hierarchy; disabling a container blocks input but changes nothing visually — disable the commands (with snapshot/restore), and set a grid's own `Enabled` for the disabled look.
- [WinForms Button.DialogResult closes a modal before an async Click handler's awaited work finishes](SolvedProblems/winforms-button-dialogresult-async-click.md) — a Designer-set `DialogResult` on a button auto-closes the dialog as soon as the synchronous part of `async void Click` returns, not when the awaited work completes; set `DialogResult` in code, only at the actual success point.
- [CWATIS and bookkeep don't share id spaces for their own per-org reference tables](SolvedProblems/cwatis-bookkeep-reference-id-mismatch.md) — bookkeep-local tables like `bookkeep.subdivisions` are Host-scoped with their own id sequence; a same-shaped id from elsewhere silently resolves to the wrong row instead of erroring. Fix is matching via the shared `global_db.rfr_subdivisions.SDId`, not by id and not by code (`DocHelper.ResolveLocalSubDivId` in bookkeep). Also recurs entirely within bookkeep's own forms during its ongoing local-to-global SubDivId migration.
- [Refers.CustomCombo.EndAddCustomItem resets the selection and replaces the whole item list](SolvedProblems/customcombo-endaddcustomitem-resets-selection.md) — every call does `DataSource = null` (selection lost, never restored automatically) and the passed list *replaces* rather than appends, so the "fill blank item, then top up" pattern silently drops the blank; with diff-based saving the lost selection becomes a real `UPDATE` and a `0` in a nullable FK blows up on the constraint.
- [ContractsMap maps an absent DB value to 0, not null, on several nullable ContractModel fields](SolvedProblems/contractsmap-zero-instead-of-null.md) — `ExecuterId`/`ContractSellerId`/`ContractBuyerId`/`CurrencyId`/`CurrencyPaymentId`/`InitiatorId`/`SdId` etc. become `0` (not `null`) when the DB value is NULL, despite being typed `int?`. A `== null` check misses it; check `<= 0` instead when treating these as "is this set".
- [CWATIS stored procedures live only in the live DB, never in the repo](SolvedProblems/cwatis-stored-procedures-not-in-repo.md) — zero `.sql` files exist anywhere in the repo; a procedure's schema can't be confirmed by grepping, and neither the `definer=` line nor sibling naming conventions reliably indicate which schema it lives in — ask the user for the exact `schema.procedure_name` rather than guessing.
- [Tauri `Window::center()` centers on work area; `monitor.size()`/`position()` give the full screen](SolvedProblems/tauri-window-center-work-area-vs-monitor-size.md) — mixing `center()` with hand-rolled multi-monitor/cursor-aware centering math causes a small, consistent vertical jump (half the taskbar height) on every resize after show; make all manual centering use `monitor.work_area()`, not `size()`/`position()`.
- [MySQL procedure: `IN` parameter mistaken for an output channel](SolvedProblems/mysql-in-param-mistaken-for-output.md) — `SELECT LAST_INSERT_ID() INTO _someIdParam` against a param declared `IN` (not `OUT`/`INOUT`) compiles and runs with no error, but only assigns a local copy the caller never sees; a C# repository trusting that param's "returned" value as the new row's id gets a silently-wrong id that then fails on any follow-up call keyed on it. Fix is `IN` → `INOUT` on that one parameter.
- [`HelperMethodClasses.UniSaver` looks generic, is hardcoded to `cwatis.pricecards`](SolvedProblems/helpermethodclasses-unisaver-hardcoded-pricecards.md) — signature (`ID, UserId, colname, value`) reads like a general single-column-update helper for `global_db.UniSaver`, but the schema/table are hardcoded in the method body to `cwatis`/`pricecards`; calling it for any other table compiles and runs but silently touches the wrong table. Write a small `SaveFieldAsync(schema, table, id, userId, colname, value)` in the owning repository instead.
- [`global_db.UniSaver`'s `schema` argument must be the table's real physical schema, verified per-table](SolvedProblems/unisaver-schema-must-be-physical-not-procedure-namespace.md) — a wrong schema string produces a MySQL syntax error deep in UniSaver's generated SQL. `rfr_subdivisions` lives in `global_db` (confirmed 2026-08-18) — an earlier version of this entry wrongly generalized that to all `rfr_*` tables based on other tables living in `refers`; that generalization doesn't hold. Verify each table's real schema individually via its own call sites, don't infer it from a procedure's namespace or from a same-family table's precedent.
- [Git Bash mangles Windows-style `/flag` arguments (MSBuild etc.)](SolvedProblems/git-bash-mangles-msbuild-slash-flags.md) — running MSBuild's `/p:...`/`/nologo`-style switches through the Bash tool gets them silently rewritten as POSIX-style paths (MSYS path-conversion), producing a confusing MSBuild-side argument error (`MSB1008: only one project can be specified`) that doesn't point at the shell at all. Use the PowerShell tool for any Windows CLI that takes `/flag` syntax; `--`/`-` style flags (npm, git) are unaffected.
- [SubdivFinances: CallService.cs is dead code, and the `Refers` submodules are never checked out](SolvedProblems/subdivfinances-callservice-and-refers-are-external.md) — `SubdivFinances\CallService.cs` looks like the active implementation but is 100% commented out; the real `CallService`/`GRP_Client` type resolves via a `GlobalUsings.cs` alias into the external `Core.Client`/`GRP-Client.V2` package. `Refers`-namespace classes (`RfrSubDivisionsForm`, `CustomCombo`, `TDBG`, …) come from the precompiled `Refers.V2` NuGet package, not the (uninitialized, empty) `submodules/refers` git submodule. Both are opaque external binaries from this repo — don't try to find or edit their source here.
- [SubdivFinances/Bookkeep: designer FileNotFoundException from a transitive package-version mismatch](SolvedProblems/subdivfinances-refersv2-mcglobalforms-version-mismatch.md) — `packages.config` does no transitive dependency resolution, so bumping `MC_GlobalForms` independently of `Refers.V2` (whose own nuspec pins an exact `MC_GlobalForms` version baked into its compiled manifest, invisible to grep) leaves two versions in play; since `MC_GlobalForms` is unsigned, `bindingRedirect` can't bridge the drift — the referenced version must match exactly what `Refers.V2` was compiled against. Recurred in Bookkeep 2026-08-21 with different version numbers — confirmed generic, not a one-off; re-extract `Refers.V2`'s nuspec each time rather than assuming a version from precedent. Ideal fix belongs on the producing side (rebuild `Refers.V2`); from the consuming repo, downgrade `MC_GlobalForms` to match and restore.
- [GitLab CI: NuGet restore fails for internal-feed packages that exist and resolve fine locally](SolvedProblems/gitlab-ci-nuget-restore-missing-internal-feed.md) — neither `Refers.sln` nor `SubdivFinances.csproj` ships a repo-level `NuGet.Config`, so restore falls back to whatever sources are registered machine-wide on whichever runner picks up the job; the internal `mars.micros.uz:9109` feed is absent there. The "Использованные веб-каналы" (used feeds) line at the end of the restore log is the tell — total absence of a source (vs. a listed connection error) means it isn't registered on that runner, not that the package is missing. Fix: commit a `NuGet.Config` at the solution root declaring the internal source (no credentials needed).
- [System.ValueTuple FileNotFoundException at runtime after a NuGet bump on net47+ packages.config projects](SolvedProblems/system-valuetuple-reference-removed-on-nuget-upgrade.md) — bumping the package makes NuGet delete the `.csproj`'s explicit `<Reference>`/`HintPath` (replaced by a conditional `.targets` import), so the DLL stops being copied to `bin\`; the package's own `.targets` also actively strips any auto-suggested `bindingRedirect` for net471+ targets, so a rebuild can't self-heal it. Fix: manually restore both the `<Reference HintPath>` (new version) and the `bindingRedirect` in `app.config`.

## Working with Wiki

When the user asks to save information to the wiki, follow these rules.

### Location

The wiki is located in a folder named `Wiki` or `<ProjectName>Wiki` in the project root. The project-level `CLAUDE.md` may specify the exact path — that takes priority.

### Rules

1. **Check first** — before creating a new file, look for an existing file covering the same topic. Update it rather than creating a duplicate.

2. **Structured folders** — organize pages in thematic subfolders. Do not dump everything in the root.

3. **No duplication** — if information belongs to an existing page, add a section there. Only create a new file when the topic is clearly distinct.

4. **Readable filenames in the project language** — use meaningful words, not technical identifiers or abbreviations.

5. **Documentation only** — the wiki contains factual information about the system: table schemas, field descriptions, component relationships, workflows. Do not include task-specific observations, conclusions tied to a current development task, or anything that would become stale once the task is done.

6. **Cross-page links** — use Obsidian wiki-link syntax `[[Page Name]]` to connect related pages. Link liberally, even to pages that do not exist yet.

## Coding Conventions (CWATIS)

See `CodingConventions.md` (same directory as this file) for CWATIS coding conventions — MVP pattern (naming, View/Presenter/Repository roles), SmartGrid/TreeFlexGrid usage basics, Designer file rules, class-size decomposition, "no hack solutions" rules, async/await rules, repository/mapper patterns, and the per-feature folder structure.

## SmartGrid Conventions (CWATIS)

See `SmartGridConventions.md` (same directory as this file) for SmartGrid usage conventions — context menu patterns, resolving a row's id when the grid has no bound id column, etc.

## TreeFlexGrid Conventions (CWATIS)

See `TreeFlexGridConventions.md` (same directory as this file) for `Manufacture.Compotents.TreeFlexGrid` usage conventions — the `IDataTree` contract, column setup, and a gotcha where `AutoGenerateColumns = true` silently breaks tree rendering.

## JSONVIewForm (CWATIS)

See `JSONVIewForm.md` (same directory as this file) for how the reusable JSON tree/raw-text viewer dialog works and how to open it from another form.

## FormattedMessageBox (CWATIS)

For a confirmation/message dialog that needs **bold** text (the native `MessageBox` only supports plain text), use `CWATIS.HelperClasses.FormattedMessageBox.Show(...)` instead of `System.Windows.Forms.MessageBox.Show(...)` — mark bold spans with `**double asterisks**`. See `FormattedMessageBox.md` (same directory as this file) for the full API and how it's implemented.

## Using the GitLab CLI (glab)

See `GlabCliUsage.md` (same directory as this file) — installation (`winget install GLab.GLab`, not `GitLab.GLab`), why `glab` doesn't show up on PATH immediately, getting through the interactive `glab auth login` wizard for the self-hosted instance (`gitlab.micros.uz`), why multi-line text (e.g. an issue description) needs to go through `ProcessStartInfo.Arguments` directly rather than a normal `&` call in PowerShell 5.1, the three priority labels on `microsdvl/smartgrid` to apply when creating work items there, why scoped (`key::value`) labels render as a flat pill instead of two-tone on this instance (Community Edition, no Premium license), and: **issues/work items created on GitLab should be written in Russian**, even though this instructions file itself is in English.

## Writing SQL Procedures

Always write the schema prefix on the procedure name — in `CREATE PROCEDURE`/`DROP PROCEDURE` statements, and anywhere else the procedure is referenced (calls, grants, etc.). Never emit a bare procedure name.

```sql
DROP PROCEDURE IF EXISTS bookkeep.Contracts_Get;

create
    definer = bookkeep@`%` procedure bookkeep.Contracts_Get(...)
BEGIN
    ...
END;
```
