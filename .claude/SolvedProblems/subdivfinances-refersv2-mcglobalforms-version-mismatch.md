# SubdivFinances/Bookkeep: WinForms designer FileNotFoundException for a package version that appears nowhere in the repo

**Date:** 2026-08-20, recurred 2026-08-21 · **Project:** first seen in SubdivFinances (`D:\projects\subdivfinances`), recurred in Bookkeep (`D:\projects\bookkeep\Bookkeep2`, form `SubDivisionsRB+SubDivisionsRefForm`) with different concrete version numbers — confirms this is a generic `Refers.V2`/`MC_GlobalForms` pairing issue, not a one-off. **Component:** any form deriving (directly or via a `Refers.V2` base class) from `Refers.V2`; more generally, any `packages.config`-style upgrade touching a package that other precompiled internal packages themselves depend on.

**Recurrence note (Bookkeep, 2026-08-21):** same mechanism, different numbers — `Refers.V2` was bumped to `1.1.3.5` (nuspec confirmed via the `.nupkg`-extraction method below: pins `MC_GlobalForms` at `1.1.6.4`), while `packages.config`/`.csproj` had independently bumped `MC_GlobalForms` to `1.1.6.8` in the same broad update pass. Fix was the same shape: edit `packages.config` + the `.csproj` `<Reference>`/`HintPath` down to the exact `1.1.6.4` that `Refers.V2 1.1.3.5`'s nuspec pins, then restore (`1.1.6.4` wasn't on disk yet — see below for a no-download way to get a working `nuget.exe` when none is on `PATH`). Also worth knowing: MC_GlobalForms and Refers.V2 versions apparently climb together over time — the version that was "too new" for an older `Refers.V2` in SubdivFinances (`1.1.6.4`) is exactly the version a newer `Refers.V2` (`1.1.3.5`, here in Bookkeep) now expects. Never assume a specific MC_GlobalForms version is universally "the new one" or "the old one" — always re-extract the nuspec of whatever `Refers.V2` version is actually in play.

**No-`nuget.exe`-on-PATH tip:** if `nuget.exe` isn't installed/on `PATH` and downloading a fresh copy isn't wanted, some old NuGet packages bundled their own `tools\NuGet.exe` for legacy install-script bootstrapping — e.g. this repo already had a genuine, current one at `packages\Microsoft.Office.Interop.Excel.<version>\NuGet.exe` (confirmed via `.VersionInfo` — real NuGet CLI, not a stub). Check `Get-ChildItem -Recurse -Filter nuget.exe` under an existing `packages\` folder before fetching a new one. `nuget.exe restore Solution.sln -ConfigFile NuGet.Config` then fetches whatever exact version the corrected `packages.config`/`.csproj` now asks for.

## Symptom

Opening the VS designer for `Main\ContentsForm.cs` (or any form ultimately deriving from `HelpClasses\TemplateFormMDI.cs`) throws, verbatim:

```
Microsoft Visual Studio
The control SubdivFinances.TemplateFormMDI has thrown an unhandled exception in the designer and has been disabled.

Exception:
Could not load file or assembly 'MC_GlobalForms, Version=1.1.5.46, Culture=neutral, PublicKeyToken=null' or one of its dependencies. Не удается найти указанный файл.
```

`MC_GlobalForms` *is* referenced in `SubdivFinances.csproj`/`packages.config`, and its DLL *is* physically present under `..\packages\` — just at version `1.1.6.4`. Grepping the entire repo for `1.1.5.46` finds nothing at all — it isn't a typo'd version anywhere in source control, which makes the error look like a phantom/cache issue.

## Root cause

`1.1.5.46` isn't a direct reference in this repo, so grep can't find it — it's baked into the *compiled manifest* of a different precompiled package. `SubdivFinances.TemplateFormMDI` inherits from `Refers.TemplateFormMDI`, which ships inside the `Refers.V2` NuGet package (see `SolvedProblems/subdivfinances-callservice-and-refers-are-external.md` for why that package's source is opaque/out-of-repo). `Refers.V2`'s own `.nuspec` — extractable from its `.nupkg`, since a classic `packages.config` restore doesn't leave a standalone `.nuspec` file on disk, only the `.nupkg` zip — declares:

```xml
<dependency id="MC_GlobalForms" version="1.1.5.46" />
```

i.e. that exact build of `Refers.V2.dll` was compiled against, and has `MC_GlobalForms 1.1.5.46` baked into its own IL assembly references.

Separately, this repo's `packages.config`/`.csproj` had `MC_GlobalForms` bumped to `1.1.6.4` as part of a broader dependency-update pass (a dozen packages touched in one uncommitted change), without `Refers.V2` being rebuilt against the newer version. `packages.config`-style restore (unlike `PackageReference`/`project.assets.json`) does **no transitive dependency resolution or reconciliation** — it restores exactly the versions explicitly listed in `packages.config`, so nothing warns you when a directly-referenced package (`Refers.V2`) and an internal-only package it depends on (`MC_GlobalForms`) drift apart.

A version drift like this would normally be invisible thanks to assembly `bindingRedirect`s — but `MC_GlobalForms` has **no strong name** (`PublicKeyToken=null`, confirmed both by the exception text and by its `<Reference>` entries in the `.csproj` never specifying a `PublicKeyToken`). `<bindingRedirect>` in `app.config` only works for strongly-named assemblies; there's no way to tell the CLR "a request for 1.1.5.46 → serve 1.1.6.4 instead" for an unsigned assembly. So when `Refers.V2.dll` asks the loader for `MC_GlobalForms, Version=1.1.5.46` and only `1.1.6.4` is actually present (plus a stale `1.1.5.13` copy sitting in `bin\Debug`, left over from before the upgrade and older than *either* version in play), it's a hard `FileNotFoundException` — surfaced by the WinForms designer as "control ... has thrown an unhandled exception," which points at the base class `TemplateFormMDI`, not at the real mismatched package pair.

## Fix / rule

- To find where a mystery assembly version is coming from when it matches nothing in the source tree: it's very likely baked into another precompiled package's manifest, not a typo in your own config. Extract the suspect dependent package's `.nuspec` from its `.nupkg` (a zip file) and check its `<dependencies>`:
  ```powershell
  Add-Type -AssemblyName System.IO.Compression.FileSystem
  $zip = [System.IO.Compression.ZipFile]::OpenRead("path\to\Package.x.y.z.nupkg")
  ($zip.Entries | Where-Object Name -like "*.nuspec") | ForEach-Object {
      (New-Object System.IO.StreamReader($_.Open())).ReadToEnd()
  }
  ```
- Before bumping any single precompiled internal package (`MC_GlobalForms`, `Refers.V2`, `Profile.V2`, `DSS.V2`, `Tools.V2`, `GRP-Client.V2`, etc.) in this `packages.config`-style project, check whether any *other* referenced internal package pins that same dependency via its own nuspec. `packages.config` won't reconcile or warn about the conflict — it'll silently let two different versions of the same weak-named DLL coexist in the reference list until something throws, at design-time or run-time.
- Because `MC_GlobalForms` is unsigned, version drift against it can't be papered over with a `bindingRedirect` the way the signed `Microsoft.Extensions.*`/`System.*`/`Microsoft.IdentityModel.*` packages in this same project can — the physically-referenced version must match exactly what the dependent package (`Refers.V2`) was compiled against. Check every unsigned `<Reference>` (no `PublicKeyToken` in the `.csproj` entry) first when triaging a "could not load assembly" designer/runtime error — it's the category that can't be fixed by adding a redirect.
- The correct fix belongs on the *producing* side: rebuild/republish `Refers.V2` against the newer `MC_GlobalForms`, then bump `Refers.V2` in this repo to that new version. `Refers.V2`'s source isn't in this repo (see `SolvedProblems/subdivfinances-callservice-and-refers-are-external.md`), so it can't be patched locally — a same-version rebuild of the outer package, or downgrading `MC_GlobalForms` back to the pinned `1.1.5.46`, are the only two options from this side.
- Separately worth checking after any such fix: `bin\Debug\` and `bin\Release\` can disagree on which version of a weak-named DLL they're holding (here `bin\Release\MC_GlobalForms.dll` already matched the new `packages.config` version while `bin\Debug\` was still stale at the *pre-upgrade* `1.1.5.13`). A plain rebuild of the active configuration overwrites it — don't assume Debug is current just because Release looks right, or vice versa.
