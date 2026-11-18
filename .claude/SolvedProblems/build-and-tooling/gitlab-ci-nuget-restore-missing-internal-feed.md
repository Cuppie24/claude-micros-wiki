# GitLab CI: NuGet restore fails for internal-feed packages that exist and resolve fine locally

**Date:** 2026-08-20 · **Project:** Refers (`D:\projects\refers`), also hit SubdivFinances and Bookkeep (`D:\projects\bookkeep`, solution `BookkeepClient.sln`) — applies to any `packages.config`-style project pulling from the internal Micros NuGet feed · **Component:** `.gitlab-ci.yml` `nuget restore`/msbuild restore step, any packages.config project with no repo-level `NuGet.Config`

## Symptom

The `build_project` CI job fails at `nuget restore Refers.sln`, verbatim:

```
NotFound https://api.nuget.org/v3-flatcontainer/mc_globalforms/1.1.6.4/mc_globalforms.1.1.6.4.nupkg
Unable to find version '1.1.6.4' of package 'MC_GlobalForms'.
  ...not found on source '...' (repeated for every source NuGet actually tried)
Unable to find version '1.0.1.13' of package 'DSSFormLib'.
Unable to find version '1.0.3.4' of package 'GRP-Client.V2'.
```

for package versions that were just bumped in `packages.config` and that exist on the internal feed and restore fine on a developer's own machine. Only some packages succeed (e.g. "Added package 'WebServiceClient.1.0.0.30' to folder ..."); internal/newer ones fail on every source NuGet tried — including the expected 404 from `api.nuget.org` (it's a private package, that 404 is normal) — while the private feed itself is never even attempted.

## Root cause

Neither `Refers.sln`'s project nor `SubdivFinances.csproj`'s project ships a repo-level `NuGet.Config`/`nuget.config` (the only `NuGet.Config` files anywhere under either repo tree are ones bundled *inside* a package's own content folder, e.g. `packages\ICSharpCode.SharpZipLib.dll.0.85.4.369\NuGet.Config` — not real project configs). With no repo-level config, restore falls back entirely to whatever NuGet sources are registered **machine-wide** on whichever box actually runs the job — for the CI runner that's `C:\Users\task.runner\AppData\Roaming\NuGet\NuGet.Config` plus `C:\Program Files (x86)\NuGet\Config\*.config` (`ComponentOne.config`, `Microsoft.VisualStudio.Offline.config`). The private company feed (`https://mars.micros.uz:9109/nuget`, registered under source key `Micros` in a developer's own `%APPDATA%\NuGet\NuGet.Config`) is not among them on this runner.

The tell is at the very end of the restore log, "Использованные веб-каналы" (used feeds) — on the failing run this list was only `nuget.org`, two local ComponentOne/SDK folder sources, and the local user package cache. The internal feed doesn't appear at all, not even as a failed/unreachable attempt. If it had been registered but merely unreachable (network/cert issue), NuGet would still list it alongside a connection error. Total absence from that list means the source isn't registered on this runner at all — it's a config gap, not a connectivity or "package doesn't exist" problem.

Packages that happened to already be sitting in one of the local mirror folders (`C:\Program Files (x86)\MESCIUS\ComponentOne\Packages`, `...\Microsoft SDKs\NuGetPackages\`) restore successfully regardless (e.g. `WebServiceClient.1.0.0.30`), which is what makes the failure look package-specific/version-specific rather than what it actually is: an environment gap on that particular runner that a repo-level config makes irrelevant.

## Fix / rule

- Commit a `NuGet.Config` at the solution root (next to the `.sln`), declaring the internal source explicitly, so restore no longer depends on any machine's global NuGet config:
  ```xml
  <?xml version="1.0" encoding="utf-8"?>
  <configuration>
    <packageSources>
      <add key="nuget.org" value="https://api.nuget.org/v3/index.json" protocolVersion="3" />
      <add key="Micros" value="https://mars.micros.uz:9109/nuget" allowInsecureConnections="True" />
    </packageSources>
  </configuration>
  ```
  This feed needs no credentials (confirmed: a working developer machine's config has no `<packageSourceCredentials>` for it), so it's safe to commit as plain text.
- When a CI NuGet restore claims a package/version is "not found" on every source, don't assume the package is actually missing or broken on the internal feed before checking the restore log's own "used feeds" / "Использованные веб-каналы" summary line — it states definitively which sources were actually consulted for that job. Compare it against a machine where restore works (e.g. a developer's `%APPDATA%\NuGet\NuGet.Config`) to spot a missing source, rather than assuming the published package/version itself is bad.
- `SubdivFinances.csproj` has the identical exposure (same absence of a repo-level `NuGet.Config`). Any future bump of a Micros-internal package (`MC_GlobalForms`, `Refers.V2`, `Profile.V2`, `DSS.V2`, `Tools.V2`, `GRP-Client.V2`, `GRP-Client`, `WebServiceClient`, `DSSFormLib`, `SmartMultiSelect`, `MicrosSmartGrid.V2`, `UserPermissionsControl`, `EImzo`, `DomainLookUpLibrary`) past whatever's cached in a runner's local mirror folders risks the identical failure there — add the same `NuGet.Config` proactively rather than waiting for the next CI break.
- Confirmed recurrence on Bookkeep (2026-08-20): `BookkeepClient.sln` had no repo-level config either; its CI log showed the identical "Использованные веб-каналы" gap (Micros feed absent) and failed on `EImzo` and `DomainLookUpLibrary` in addition to the packages above — both are internal Micros packages, same fix applies. Any *new* `packages.config`-based solution in this company's GitLab group should get this `NuGet.Config` committed at solution-root time, before the first CI break, rather than per-project reactively.
