# System.ValueTuple FileNotFoundException at runtime after bumping the NuGet package on a net47+ packages.config project

**Date:** 2026-08-21 · **Project:** Bookkeep (`D:\projects\bookkeep`, `Bookkeep2\BookkeepClient.csproj`) · **Component:** any `packages.config`-style .NET Framework project targeting `net471`+ that references precompiled internal packages (`Refers.V2`, `GRP-Client.V2`, etc.) or netstandard2.0 libraries, and bumps the `System.ValueTuple` NuGet package as part of a broader dependency-update pass. Same root-cause family as [[subdivfinances-refersv2-mcglobalforms-version-mismatch]] (uncoordinated multi-package `packages.config` bump silently breaking a dependency invisible to grep), different concrete mechanism.

## Symptom

Runtime (not designer) error dialog on app startup:

```
Произошла ошибка. Обратитесь к разработчикам.
Не удалось загрузить файл или сборку "System.ValueTuple, Version=4.0.3.0, Culture=neutral, PublicKeyToken=cc7b13ffcd2ddd51"
либо одну из их зависимостей. Не удается найти указанный файл.
```

(`Could not load file or assembly 'System.ValueTuple, Version=4.0.3.0, ...' or one of its dependencies. The system cannot find the file specified.`)

Started immediately after a NuGet reference-update pass that bumped `System.ValueTuple` (among a dozen other packages) — no code changes to trigger it otherwise.

## Root cause

`packages.config` bumped `System.ValueTuple` `4.5.0` → `4.6.1`. NuGet's package-upgrade tooling didn't just change the reference's version — it **deleted the explicit `<Reference Include="System.ValueTuple"...><HintPath>...` element from the `.csproj` entirely**, replacing it with a conditional `<Import Project="..\packages\System.ValueTuple.4.6.1\build\net471\System.ValueTuple.targets" .../>`.

That `.targets` file (shipped inside the package itself) contains only this:

```xml
<!-- System.ValueTuple is inbox on .NET Framework >= 4.7.1 and therefore any potential redirect for it should be removed.
     This is necessary as the assembly version in the already shipped packages is higher than what's provided inbox on .NET Framework. -->
<Target Name="RemoveValueTupleRedirectForNet471AndAbove" DependsOnTargets="ResolveAssemblyReferences" BeforeTargets="GenerateBindingRedirects">
  <ItemGroup>
    <SuggestedBindingRedirects Remove="System.ValueTuple, Culture=neutral, PublicKeyToken=cc7b13ffcd2ddd51" />
  </ItemGroup>
</Target>
```

i.e. for a project targeting `net471`+ (this one targets `v4.8`), the package's own build logic actively **removes System.ValueTuple from the auto-generated binding-redirect suggestions** during `GenerateBindingRedirects` — on the assumption that the framework's inbox `ValueTuple<>` types make the standalone facade assembly unnecessary.

That assumption only holds for the project's *own* code. It does not hold for precompiled binaries already sitting in the reference list that were compiled against the old facade assembly and have `System.ValueTuple, Version=4.0.3.0, PublicKeyToken=cc7b13ffcd2ddd51` baked into their own manifest as an external dependency (candidates here: `Refers.V2.dll` — already the source of an unrelated version-mismatch bug, see [[subdivfinances-refersv2-mcglobalforms-version-mismatch]] — and/or one of the netstandard2.0 packages added in the same pass: `ClosedXML`, `DocumentFormat.OpenXml`, `RBush`, `SixLabors.Fonts`). The CLR has no built-in type-forwarding from inbox `mscorlib` types to a third party's exact external assembly identity request — it still needs a real `System.ValueTuple.dll` on disk (or a binding redirect to one) to satisfy that dependent's load request, regardless of what the compiling project's own target framework provides.

Net effect once the `<Reference>` was removed: `System.ValueTuple.dll` stopped being copied to the build output at all. Confirmed on disk: `bin\Release\System.ValueTuple.dll` was present only as a stale leftover from a build made *before* this upgrade (untouched, old timestamp) — `bin\Debug\` had no copy whatsoever. A clean rebuild of either configuration would drop it entirely, since nothing in the project references it anymore.

## Fix

Two changes, both required — and the second one is actively fought by the package's own `.targets` logic, so it will never appear via auto-generation:

1. **Restore the explicit `<Reference>`** in the `.csproj`, pointing at the new package version's actual DLL (assembly version bumped to `4.0.5.0` in package `4.6.1`, confirmed via `[System.Reflection.AssemblyName]::GetAssemblyName(...)` — it is *not* the file/package version):
   ```xml
   <Reference Include="System.ValueTuple, Version=4.0.5.0, Culture=neutral, PublicKeyToken=cc7b13ffcd2ddd51, processorArchitecture=MSIL">
     <HintPath>..\packages\System.ValueTuple.4.6.1\lib\net47\System.ValueTuple.dll</HintPath>
   </Reference>
   ```
   This alone is what makes MSBuild copy the DLL back into `bin\` on build.

2. **Manually add the `bindingRedirect`** in `app.config` — do not rely on `GenerateBindingRedirects` to add it, because the package's `.targets` file explicitly strips any `System.ValueTuple` entry from `SuggestedBindingRedirects` for net471+ targets:
   ```xml
   <dependentAssembly>
     <assemblyIdentity name="System.ValueTuple" publicKeyToken="cc7b13ffcd2ddd51" culture="neutral" />
     <bindingRedirect oldVersion="0.0.0.0-4.0.5.0" newVersion="4.0.5.0" />
   </dependentAssembly>
   ```
   A manually-authored redirect already in `app.config` survives the merge — the target only prevents a *new* one from being suggested, it doesn't delete one you wrote yourself.

## Checklist / rule

- Treat `System.ValueTuple` as a package that needs its `<Reference>`/`HintPath` re-verified by hand after every version bump on any `packages.config`-style net47+ project that also carries precompiled/opaque internal packages (`Refers.V2`, `GRP-Client.V2`, `Profile.V2`, `Tools.V2`, `MC_GlobalForms`, ...) or netstandard2.0 third-party libs — NuGet's upgrade tooling will silently remove the reference on the assumption the framework provides it, which is only true if literally nothing in the dependency closure still needs the standalone facade.
- If a `FileNotFoundException`/load error names an assembly with a `PublicKeyToken` you don't recognize needing a fix for, don't assume a `bindingRedirect` alone will fix it — first confirm the assembly's `<Reference>` (with `HintPath`) still exists in the `.csproj` at all and that the DLL is actually landing in `bin\`. A redirect pointing at a version that was never copied to output fails identically to having no redirect.
- Don't trust `bin\Release\` as a sanity check that "it must still work" — a stale DLL left over from before the breaking change can sit there indefinitely since incremental builds don't delete outputs just because a reference disappeared. Check `bin\Debug\` too, and ideally do a clean rebuild before concluding a fix worked.
- When a package ships a `build\<tfm>\*.targets` file, read it before trusting the upgrade — it can contain logic (like the `RemoveValueTupleRedirectForNet471AndAbove` target here) that actively undoes what you'd otherwise expect MSBuild's default binding-redirect generation to do for you.
