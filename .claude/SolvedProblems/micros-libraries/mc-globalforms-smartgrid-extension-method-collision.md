# MC_GlobalForms update adds SmartGrid extension methods that collide with a product's own same-named ones

**Symptom** — After bumping the `MC_GlobalForms` NuGet package, code that previously compiled fine now fails with:

```
error CS0121: Неоднозначный вызов следующих методов или свойств: "CWATIS.Extentions.SmartGridExtentions.GetModelSelectedRow<T>(MicrosSmartGrid.SmartGrid)" и "MC_GlobalForms.HelperClasses.SmartGrid_Extentions.GetModelSelectedRow<T>(MicrosSmartGrid.SmartGrid)"
```

at ordinary extension-method call sites like `grid.GetModelSelectedRow<T>()` / `grid.GetModelRow<T>(index)`, in files that have nothing to do with whatever feature motivated the package bump. It can look like a *new* regression from unrelated work, especially if it doesn't surface until *after* fixing some other, earlier compile error elsewhere in the project — severe binding-phase errors (e.g. `CS0234`, an unresolvable `using`) can suppress a lot of downstream semantic-analysis diagnostics project-wide, so this ambiguity may have been latent since the package bump but only becomes visible once those earlier errors are cleared.

**Root cause** — `MC_GlobalForms.HelperClasses.SmartGrid_Extentions` (note the underscore — different type name, same containing concept) independently added its own `GetModelRow<T>`/`GetModelSelectedRow<T>` extension methods (to solve the same `C1DataCollectionBindingList`/`IDataCollectionBindingListItem` unwrapping problem documented in [[globalforms-smartgrid-missing-model-unwrap-helper]]) with **identical signatures** to methods a consuming product (CWATIS) already had in its own `CWATIS.Extentions.SmartGridExtentions`. Any file with both `using CWATIS.Extentions;` and `using MC_GlobalForms.HelperClasses;` now has two equally-applicable extension methods in scope for the same call — C# can't pick one, so it's a compile error, not a silent pick of either.

Don't reach for "just remove one of the `using`s" as the fix without checking first — in practice both files affected in CWATIS used many *other* unrelated symbols from `MC_GlobalForms.HelperClasses` (`MyMessageBox`, `Numeric`, `JsonHelper`, etc.), so removing that `using` entirely would have broken those call sites instead.

A same-named-but-different-namespace third candidate can compound this: if the file *also* imports `Refers.RefersM.MSubjectNewHelper.Extentions` (which apparently has its own `SmartGridExtentions` class, no underscore, matching CWATIS's own class name exactly), then even a call fully qualified down to `SmartGridExtentions.GetModelSelectedRow<T>(grid)` is itself ambiguous (`CS0104`, "SmartGridExtentions" is an ambiguous reference) — that needs the *namespace* included too: `CWATIS.Extentions.SmartGridExtentions.GetModelSelectedRow<T>(grid)`.

**Fix** — At each ambiguous call site, replace the extension-method dot-syntax with an explicit static-method call naming the specific class you want (test whether the bare class name alone is already ambiguous with a third candidate before deciding how much to qualify):

```csharp
// Before (ambiguous once MC_GlobalForms.HelperClasses.SmartGrid_Extentions exists):
var model = grid.GetModelSelectedRow<ContractModel>();

// After:
var model = CWATIS.Extentions.SmartGridExtentions.GetModelSelectedRow<ContractModel>(grid);
```

This resolves the ambiguity without touching any `using` directive, so it can't collaterally break other symbols pulled in from the same namespace.

**Checklist / rule**
- After any `MC_GlobalForms` (or other shared-library) version bump, a clean build is the only reliable signal - don't assume "it compiled before the bump" still holds, and don't assume a sudden new error in an unrelated file means you broke something unrelated. Check whether the library's own changelog/diff added extension methods that shadow first-party ones with the same signature.
- A `CS0121` "ambiguous call" between an extension method you wrote and one from a newly-updated dependency is a naming collision, not a real design conflict - it doesn't mean either implementation is wrong, just that both are visible at once. Qualify the call, don't rewrite either implementation.
- Don't blindly strip a `using` to "fix" an ambiguity - grep the file first for other symbols from that namespace (in CWATIS: `MyMessageBox`, `Numeric`, `JsonHelper`, `CurrencyUtils`, etc. all live in the same flat `MC_GlobalForms.HelperClasses` namespace as the colliding extension methods, despite being defined in different subfolders of the `globalforms` source repo).
- If disambiguating the class name alone still errors as ambiguous (`CS0104`), something else imported in the same file has a same-named type too (seen here with `Refers.RefersM.MSubjectNewHelper.Extentions.SmartGridExtentions`) - qualify with the full namespace instead of just the class name.

Found in: CWATIS, 2026-08-25, while porting `ContractsForm.cs`/`MainForm.cs`/`BkCntrSelectForm.cs`/`CalcTabControl.cs` off a deleted local module - the same `MC_GlobalForms` version bump (1.1.6.19→1.1.6.24) that added `CategorizedSubdivisionsForm.SelectMode` also added this collision, and it was masked until unrelated `CS0234` errors in `ContractsForm.cs`/`MainForm.cs` were fixed first.

## Gotcha: a merge can combine two different, individually-valid fixes for this same collision into a broken build

Two branches can independently "fix" this exact collision in incompatible ways without either being wrong in isolation:
- **Branch A** (this file's documented fix): keeps `CWATIS.Extentions.SmartGridExtentions` (or a renamed variant) with working `GetModelRow<T>`/`GetModelSelectedRow<T>`, and qualifies every colliding call site with the explicit class name.
- **Branch B**: instead renames the local class (e.g. `SmartGridExtentions` → `SmartGridExtentionsCW`) and deletes/comments out its `GetModelRow<T>`/`GetModelSelectedRow<T>`, so every plain extension-syntax call (`grid.GetModelSelectedRow<T>()`) in the rest of the codebase unambiguously falls through to `MC_GlobalForms.HelperClasses.SmartGrid_Extentions`.

Both branches build clean on their own. Merged together, the result is broken either way you look at it:
- Call sites added on Branch A still reference the old class name (`CS0103`/`CS0234: "SmartGridExtentions" does not exist`), because Branch B's rename won for the shared file.
- If you "fix" that by restoring Branch A's methods under Branch B's new class name, every *pre-existing* plain-extension-syntax call site elsewhere in the codebase (that used to resolve unambiguously to `MC_GlobalForms` only because the local method was absent) becomes ambiguous again (`CS0121`), exactly as in the base gotcha above.

**Fix** — apply the base gotcha's rule (qualify, don't rewrite) *everywhere at once*, not just at the call sites that first errored: restore the local `GetModelRow<T>`/`GetModelSelectedRow<T>` under whichever class name survived the merge, then grep the **whole project** for plain extension-syntax calls to these two method names (`\.GetModelRow<`, `\.GetModelSelectedRow<`) in any file that also has `using MC_GlobalForms.HelperClasses;` in scope, and qualify each one with the local class name too. Don't stop at the first clean build after fixing only the originally-erroring files - a build that's clean after round one can still regress with new `CS0121`s once the deleted method comes back, because restoring it flips every other file's extension-method overload resolution project-wide. Rebuild after each round until a full solution build is 0 errors, not just the files you touched.

Recurred in CWATIS, 2026-08-26 (`MC_GlobalForms` 1.1.6.38): a `development` merge combined an "ishnazarov" branch using the qualify-the-call fix in `ContractsForm.cs`/`BkCntrSelectForm.cs`/`CalcTabControl.cs` with an "oleg" branch that had renamed the class to `SmartGridExtentionsCW` and gutted both methods; restoring the methods to fix the first 3 files' `CS0103`/`CS0234` immediately surfaced fresh `CS0121`s in 4 unrelated files (`UserRightsForm.cs`, `ContractDSLinesTabControl.cs`, `OrgsGroupForm.cs`, `OrderTabControl.cs`) that had been relying on the method's absence.
