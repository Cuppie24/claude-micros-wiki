# WebView2 SetVirtualHostNameToFolderMapping: DirectoryNotFoundException with no indication of which path is wrong

**Symptom** (runtime, thrown from inside the `FeedbackForm_Load` async method or wherever `EnsureCoreWebView2Async`/`SetVirtualHostNameToFolderMapping` is called):

```
System.IO.DirectoryNotFoundException
  HResult=0x80070003
  Message=Системе не удается найти указанный путь. (Исключение из HRESULT: 0x80070003)
  Source=Microsoft.Web.WebView2.Core
  StackTrace:
   at Microsoft.Web.WebView2.Core.Raw.ICoreWebView2_3.SetVirtualHostNameToFolderMapping(...)
   at Microsoft.Web.WebView2.Core.CoreWebView2.SetVirtualHostNameToFolderMapping(...)
```

The exception gives no hint about *which* path is wrong or why — it's a raw HRESULT translated by the COM interop layer.

## Root causes (two independent bugs, both produce this exact exception)

**1. The second argument must be a folder, not a file.** `SetVirtualHostNameToFolderMapping(hostName, folderPath, accessKind)` maps a virtual origin to a *directory*; it does not accept a path to a specific `.html` file. Building the path with the target filename baked in (e.g. `Path.Combine(baseDir, "www", "feedback-form.html")`) throws this exception even if the file exists, because as a "folder" it doesn't.

**2. In a legacy (`packages.config`-style, non-SDK) `.csproj`, a plain `<Content Include="...">` item does NOT copy to the output directory by default.** Unlike SDK-style projects, you must add `<CopyToOutputDirectory>PreserveNewest</CopyToOutputDirectory>` explicitly, or the file silently never reaches `bin\Debug`/`bin\Release` even though the build succeeds with no errors or warnings. (Other native-dependency DLLs in the same project that appear to "just work" via a bare `<Content Include>` are actually being copied by their owning NuGet package's own `.targets` file, not by that Content item — don't use their presence in `bin\` as proof that plain Content copying works in this project.)

A related trap: if the source file lives **outside** the project directory (e.g. a `www\` folder at the repo root, sibling to the `.csproj`), `<Content Include="..\www\file.html">` without a `<Link>` copies to `bin\Debug\..\www\file.html` (i.e. one level *above* the output directory, not inside it) — because MSBuild preserves the relative path from the project file. Add `<Link>www\file.html</Link>` to pin the destination path inside the output directory regardless of where the source lives.

## Fix

```csharp
// Folder, not file:
private readonly string wwwFolderPath =
    System.IO.Path.Combine(AppDomain.CurrentDomain.BaseDirectory, "www");

await webView.EnsureCoreWebView2Async();
webView.CoreWebView2.SetVirtualHostNameToFolderMapping(
    "feedback.app", wwwFolderPath, CoreWebView2HostResourceAccessKind.Deny);
webView.CoreWebView2.Navigate("https://feedback.app/feedback-form.html");
```

```xml
<!-- .csproj -->
<Content Include="..\www\feedback-form.html">
  <Link>www\feedback-form.html</Link>
  <CopyToOutputDirectory>PreserveNewest</CopyToOutputDirectory>
</Content>
```

## Checklist

- `SetVirtualHostNameToFolderMapping`'s `folderPath` argument is always a directory — never point it at a specific file.
- After adding any new `Content` item to a `packages.config`-style `.csproj`, verify it actually lands in `bin\<Config>\` after a build (don't assume — check the file exists on disk); the build succeeding is not proof the copy happened.
- If the source file is outside the project's own directory tree, add `<Link>` so the destination path doesn't inherit the `..\` from the source's relative path.
- CWATIS-specific: found in `CWATIS\Forms\Feedback\View\FeedbackForm.cs` (2026-08-26), while wiring a local HTML feedback form (`www\feedback-form.html`) into a WinForms `WebView2` control.
