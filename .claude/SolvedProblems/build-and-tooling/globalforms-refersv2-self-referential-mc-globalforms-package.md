# GlobalForms: adding Refers.V2 pulls in a NuGet package named MC_GlobalForms — which collides with the project itself

**Date:** 2026-08-24 · **Project:** GlobalForms (`D:\projects\projects\globalforms`, `.csproj` assembly/root namespace `MC_GlobalForms`). **Component:** `GlobalForms\MC_GlobalForms.csproj` + `packages.config`, triggered by installing `Refers.V2` via the VS NuGet UI.

## History: this is not new in Refers.V2, GlobalForms just never had a live reference during the window it applies to

Queried the internal feed (`https://mars.micros.uz:9109/nuget`, OData `FindPackagesById()?id='Refers.V2'`, `Dependencies` property per version) directly rather than guessing from a single version:

- `Refers.V2 1.0.4.94` — the exact version this repo had installed 2025-01-17–2025-02-06 (see below) — depends on `GRP-Client.V2:1.0.3|MicrosSmartGrid.V2:2.0.0.7|Profile.V2:1.0.1|System.ServiceModel.Primitives:4.10.0|Tools.V2:1.0.3.3|UserPermissionsControl:1.0.0`. **No `MC_GlobalForms` anywhere.**
- `Refers.V2 1.0.4.95` — the very next published version — is the **first** version whose `Dependencies` includes `MC_GlobalForms` (pinned at `1.0.0.20` at that point). Confirmed by binary-searching all 243 published versions (`1.0.4.50` … `1.1.3.7`) on this exact field: every version from `1.0.4.95` onward depends on `MC_GlobalForms`, every version up to and including `1.0.4.94` does not.
- Every version since has continued to pin some `MC_GlobalForms` version (climbing over time right alongside `Refers.V2`'s own version — same pattern noted in the sibling `subdivfinances-...` entry). This looks like a permanent, structural property of `Refers.V2`'s build going forward, not a transient bug — expect every future `Refers.V2` release consumed by *this* project to need the same reference-removal treatment.

**So "when did this problem first appear" has two different correct answers:** in `Refers.V2`'s own release history, at `1.0.4.95` (very shortly after the `1.0.4.94` this repo happened to be pinned to). In *this repo's own experience of it*, only today (2026-08-24) — GlobalForms had zero `Refers`/`Refers.V2` reference at all from 2025-02-06 until today (see history below), so the window where it would have hit this never overlapped with a live reference.

## History: Refers.V2 was already tried once in this repo, and fully removed — for an unrelated reason

- **Added** `612c723` (2025-01-17, "upload nuget package RefersV2 for templateFormMDI..."): `Refers.V2 1.0.4.94` + `Profile.V2 1.0.1` (new), `GRP-Client.V2` bumped `1.0.2.9→1.0.3`, `Tools.V2` bumped `1.0.3→1.0.3.3`, plus new `System.ServiceModel.Primitives 4.10.0`/`UserPermissionsControl 1.0.0`. Also vendored three DLLs directly under `GlobalForms\Libs\` (`ICSharpCode.SharpZipLib.dll`, `MessageCompressor.dll`, `Newtonsoft.Json.dll`) rather than via NuGet. `CountryTreeForm`/`LocationTreeViewForm`/`PlacesTreeForm` were switched to a `Refers`-based `TemplateFormMDI` base.
- **Removed** `5cdc704` (2025-02-06, ~3 weeks later, "удалил nuget refers вместе с его profile оказалось который подтянулся с ним и с templateFormMDI перевёл снова на Form..." — "removed the refers nuget package along with its Profile, which turned out to have been pulled in with it, and switched templateFormMDI back to plain Form for now"): removed `Refers.V2` **and** `Profile.V2` entirely, reverted the three forms back to plain `Form`, removed the three vendored `Libs\*.dll` files. The stated reason was the unwanted `Profile.V2` transitive package, not `MC_GlobalForms` — consistent with `1.0.4.94` genuinely not depending on it.
- **Reinstalled** today (2026-08-24), landing on `Refers.V2 1.1.3.7` (243 versions newer than `1.0.4.94`) — which is why the `MC_GlobalForms` self-collision showed up this time and didn't the first time.
- Confirmed via `git log --all -p -- GlobalForms/packages.config` that `Refers.V2` appears in exactly these three events (add, remove, today's re-add) across the entire repo history — no other attempts in between.

Related but distinct from [subdivfinances-refersv2-mcglobalforms-version-mismatch.md](subdivfinances-refersv2-mcglobalforms-version-mismatch.md) — that entry is about a *version mismatch* between a consuming project's own `MC_GlobalForms` reference and the version `Refers.V2` expects. This is a different failure shape: here the *consuming project itself is named/compiles to `MC_GlobalForms`*, so pulling in `Refers.V2`'s transitive `MC_GlobalForms` package dependency creates a genuine self-reference, not a drift between two versions of an external dependency.

## Symptom

After adding `Refers.V2` (to resolve an unrelated missing-`Refers`-namespace compile error), the build breaks across many unrelated, pre-existing files with `CS0121` ambiguous-call errors, e.g.:

```
error CS0121: Неоднозначный вызов следующих методов или свойств:
"MC_GlobalForms.HelperClasses.Helper.TextR(C1.Win.C1Input.C1TextBox)
    [D:\...\GlobalForms\HelperClasses\Common\Helpers\Helper.cs(61)]"
и
"MC_GlobalForms.HelperClasses.Helper.TextR(C1.Win.C1Input.C1TextBox)
    [D:\...\packages\MC_GlobalForms.1.1.6.18\lib\net48\MC_GlobalForms.dll]"
```

Every type this project defines under its own root namespace (`Helper.TextR`, `SmartGrid_Extentions.GetValueRow<T>`, `Convertion.Val`, etc.) is now ambiguous — the compiler sees two identical candidates: the project's own source, and a NuGet package DLL with the exact same assembly name.

## Root cause

`GlobalForms\MC_GlobalForms.csproj` compiles to assembly/root-namespace `MC_GlobalForms` (confirmed: `<AssemblyName>MC_GlobalForms</AssemblyName>`, `<RootNamespace>MC_GlobalForms</RootNamespace>`). This project is also, separately, published as a NuGet package of the same name (evidenced by the repo's own latest commit at the time, "Update assembly version to 1.1.6.18" — a version-bump commit for exactly that package).

`Refers.V2`'s own nuspec pins an exact `MC_GlobalForms` version as one of its dependencies (the same mechanism documented in the sibling entry above — the version is baked into `Refers.V2`'s compiled manifest, invisible to grep). Installing `Refers.V2` via NuGet in this project therefore adds `<package id="MC_GlobalForms" version="1.1.6.18" .../>` to `packages.config` and a `<Reference Include="MC_GlobalForms, Version=1.1.6.18, ...">` (`HintPath` into `packages\MC_GlobalForms.1.1.6.18\lib\net48\MC_GlobalForms.dll`) to the `.csproj` — a *packaged copy of this exact project* sitting alongside its own live source. Every type both define collides.

This is a one-way trap for this specific repo: any consumer of `Refers.V2` that is *not* itself `MC_GlobalForms` would just get a second, unrelated dependency (the ordinary case the sibling entry covers). Only in `GlobalForms`/`MC_GlobalForms.csproj` itself does the transitive dependency resolve back to the consuming project.

## Fix / rule

Remove the transitive `MC_GlobalForms` package reference — both the `<Reference Include="MC_GlobalForms, ...">`/`<HintPath>` block in the `.csproj` and the `<package id="MC_GlobalForms" .../>` line in `packages.config` — while keeping `Refers.V2` and its *other* transitive packages (`GRP-Client.V2`, `DSSFormLib`, `ICSharpCode.SharpZipLib.dll`, the extra `C1.*` sub-packages it pulls in, etc.) intact. The project's own compiled source already provides every member the package would have, so nothing is lost — the ambiguity resolves in favor of the local source once the duplicate reference is gone (confirmed: full rebuild went from ~20+ `CS0121` errors across the project to zero after removing just that one `<Reference>`/`packages.config` entry).

**How to recognize this specific shape of the problem** (vs. an ordinary version mismatch): the two conflicting locations in the `CS0121` message are the project's own `HelperClasses\...\*.cs` source files *and* a `packages\<ThisProject'sOwnAssemblyName>.<version>\lib\...\<ThisProject'sOwnAssemblyName>.dll` — i.e. the package name literally matches `$(AssemblyName)`/`$(RootNamespace)` of the very project failing to build. If so, the fix is removal, not a version bump/downgrade (there is no "correct" version of yourself to reference).
