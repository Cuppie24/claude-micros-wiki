# Coding Conventions (CWATIS)

Coding conventions for the CWATIS WinForms/gRPC client and server (MVP pattern, folder structure, async rules, etc.). See also `SmartGridConventions.md`/`TreeFlexGridConventions.md` for control-specific grid/tree conventions.

## MVP Pattern

Every feature follows a strict Model-View-Presenter triad.

**Naming:**
| Artifact | Name | Example |
|---|---|---|
| Form class | `{Feature}Form.cs` | `ContractsForm.cs` |
| View interface | `I{Feature}View.cs` | `IContractsView.cs` |
| Presenter | `{Feature}Presenter.cs` | `ContractsPresenter.cs` |
| Repository interface | `I{Feature}Repository.cs` | `IContractsRepository.cs` |
| Repository impl | `{Feature}Repository.cs` | `ContractsRepository.cs` |
| Mapper | `{Feature}Map.cs` | `ContractsMap.cs` |

**Rules:**
- The View exposes **events** to the Presenter (`event Func<T, Task<U>>`). The Presenter subscribes in its constructor.
- The View exposes only what the Presenter needs — prefer narrow methods (`int GetSelectedId()`) over exposing whole controls (`SmartGrid MasterGrid`).
- The Presenter contains all business logic. Forms contain only UI wiring.
- The Presenter is constructed inside the Form; inject the repository via the constructor.

## UI Controls

**Grids — SmartGrid**  
Use `SmartGrid` for all tabular data. Never use standard `DataGridView`.

```csharp
_masterGrid.IdName = "Id";
_masterGrid.BeginUpdate();
_masterGrid.DataSource = new C1DataCollectionBindingList(data);
_masterGrid.EndUpdate();
var model = _masterGrid.GetModelSelectedRow<ContractsModel>();
```

`DataSource` must always be set via `C1DataCollectionBindingList` (never a raw `List<T>`/array) — otherwise column-header sorting is disabled. Wrap the assignment in `BeginUpdate()`/`EndUpdate()`.

Use `SmartGridExtensions` helpers (`GetModelSelectedRow<T>`, `FindModel<T>`, etc.) instead of raw row/column index access.

**Trees — TreeFlexGrid**  
Use `TreeFlexGrid` for all hierarchical/tree data. Never use `TreeView`.

```csharp
_masterGrid.BuildTreeNew(data);
_masterGrid.CollapseByLevel(1);
_masterGrid.ExpandToNodeById(id);
```

For detailed SmartGrid and TreeFlexGrid features, see `SmartGridConventions.md`/`TreeFlexGridConventions.md` (same directory as this file) and `SmartGridWiki.md` in the CWATIS project root (legacy/English, do not extend it — the active wiki is the `CwatisWiki` vault).

## Designer Files

All controls, layouts, and event subscriptions must live in the `.Designer.cs` file. Do **not** dynamically create or configure controls in the code-behind `.cs` file — no `new Button()`, no `.Controls.Add(...)`. The `.cs` file contains only logic.

## Modularity — Class Size

If a class exceeds ~400–500 lines, decompose it.

- Extract related presenter methods into a dedicated service class (e.g., `ContractFilterService`, `PriceEditService`).
- Extract groups of repository methods into a focused secondary repository if they cover a distinct sub-domain.
- Use `#region` blocks as a diagnostic — if a region is large enough to stand alone, it becomes its own class. Do not use regions as a substitute for decomposition.

## No Hack Solutions

- **No boolean suppression flags** for event cascades (`_IsRefreshMaster = true/false`). Use `BeginUpdate()`/`EndUpdate()` on the grid, or unsubscribe/resubscribe the event.
- **No `.Result` or `.Wait()` on Tasks** — causes deadlocks. Use `async void` only at the top event handler level, `await` everything below.
- **No magic numbers or strings** — use named constants or enums.
- **No UI calls (MessageBox, dialogs) inside Repositories** — repositories return results; the Presenter or View handles user feedback.
- **No direct global state access (`Main.UInfo`, `UserSingleton`) inside Repositories** — pass required values as parameters.
- **No stub/empty event handlers** in committed code — remove or implement them.

## Async/Await

- Repository methods that do I/O must be `async Task<T>`.
- Presenter methods that call repositories must be `async Task`.
- Form event handlers that call the Presenter are `async void` (the only acceptable use of `async void`).

## Repositories & Mapping

- Repositories call `CallService.CallProcedureDT()` and convert results via a mapper. No business logic in repositories.
- Mappers implement `IMapper<DataRow, TModel>` and are injected into the Repository constructor with a null-coalescing default:
  ```csharp
  _mapper = mapper ?? new ContractsMap();
  ```
- One mapper per model. Mappers do no filtering or computation — only field-to-field conversion with null/DBNull guards.

## Folder Structure per Feature

```
Forms/{Feature}/
├── View/
│   ├── {Feature}Form.cs
│   ├── {Feature}Form.Designer.cs
│   └── I{Feature}View.cs
├── Presenter/
│   └── {Feature}Presenter.cs
├── Repositories/
│   ├── I{Feature}Repository.cs
│   └── {Feature}Repository.cs
├── Mappers/
│   └── {Feature}Map.cs
├── Models/
│   └── {Feature}Model.cs
└── DTO/
    └── {Feature}FilterDTO.cs
```

Folder names are plural and consistent: `Repositories/`, `Mappers/`, `Models/`, `DTO/`.

## Suggestions

Patterns observed in the codebase that should be addressed going forward:

1. **Expose behavior, not controls** — `IView` interfaces currently expose `SmartGrid` directly. Prefer methods like `void BindMasterData(IEnumerable<T> data)` and `int GetSelectedMasterId()`. This makes the Presenter decoupled from the control library.
2. **Separate DB model from view model** — `ContractsModel` carries both `[Column]` DB attributes and UI-bound properties. Define a separate persistence model and a display model; the mapper bridges them.
3. **Centralize deletion confirmation** — the `MessageBox.Show(...)` confirmation pattern is repeated in many presenters. Extract to a `DialogHelper.ConfirmDelete(string entityName)` utility.
4. **Log exceptions, don't just rethrow** — bare `catch { throw; }` blocks add no value. Either handle and show a user message, or let it propagate to a top-level handler in `Program.cs` that logs via `log4net`.
5. **Consistent folder naming** — standardize on the names in the folder structure above (`Repositories/` not `Repositoies/`, `Mappers/` not `Mapper/`).
