# C1MainMenu / C1CommandLink has no per-item right-alignment property

**Symptom:** Wanting one top-level item (e.g. an "update available" notification command) to sit flush right in a `C1.Win.C1Command.C1MainMenu` while the rest stay left-flowed — the way `System.Windows.Forms.ToolStripItem.Alignment = ToolStripItemAlignment.Right` works, or the classic native Win32 `MFT_RIGHTJUSTIFY` menu trick. There is no obviously-named property for it, and it's easy to assume one exists and go hunting through the Properties grid or IntelliSense for it.

**Root cause:** `C1MainMenu` derives directly from `System.Windows.Forms.Control` (not from `MenuStrip`/native `HMENU`), and both it and `C1CommandLink` are fully custom-drawn, strictly left-to-right flowing containers. Confirmed by reflecting over `C1.Win.C1Command.4.8.dll` (MESCIUS ComponentOne WinForms v4.8, installed at `C:\Program Files (x86)\MESCIUS\ComponentOne\WinForms\bin\v4.8\C1.Win.C1Command.4.8.dll`):
- Dumped **every** public property of `C1CommandLink` and `C1MainMenu` (including inherited) — no `Alignment`/`AlignRight`/`RightJustify`/`Dock`-per-item property exists. `C1CommandLink.SortOrder` only controls relative ordering among siblings, not which side of the strip they render on.
- Cross-checked the shipped XML doc comments (`C1.Win.C1Command.4.8.xml`) for any mention of right-alignment anywhere in the whole assembly's menu/toolbar members — nothing.
- The closest-sounding members, `C1CommandLink.IsServiceLink` / `IsMoreLink`, are internal flags for the auto-generated overflow chevron ("more" button) and scroll buttons, not a general-purpose alignment flag — a plausible-looking dead end.
- `C1MainMenu.RightToLeft` is the standard WinForms RTL layout flip (mirrors the *whole* control), not per-item alignment.

**Fix:** Host the item that needs to be on the right in a **separate, small `C1.Win.C1Command.C1ToolBar`** (same `CommandHolder`, so it shares the app's existing `C1Command` objects/click handlers/state unchanged), positioned as a sibling of the main `C1MainMenu`:
```csharp
this.updateToolBar = new C1.Win.C1Command.C1ToolBar();
this.updateToolBar.Anchor = AnchorStyles.Top | AnchorStyles.Right;
this.updateToolBar.AutoSize = true;
this.updateToolBar.CommandHolder = this.c1CommandHolder1;   // same holder as the main menu
this.updateToolBar.CommandLinks.AddRange(new[] { existingLink }); // move the link out of c1MainMenu1.CommandLinks
this.updateToolBar.VisualStyle = this.updateToolBar.VisualStyleBase = C1.Win.C1Command.VisualStyle.Office2010Silver; // match the main menu
this.updateToolBar.Location = new Point(formWidth - guessedWidth, 0); // AutoSize + Anchor.Right settle the exact edge
// z-order: the new toolbar overlaps the full-width Dock=Top menu, so it must paint in front of it
this.Controls.Add(this.mdiChildTaskBar1);
this.Controls.Add(this.updateToolBar);   // add BEFORE the menu …
this.Controls.Add(this.c1MainMenu1);     // … so the menu (Dock=Top, full width) doesn't cover it
this.updateToolBar.BringToFront();       // belt-and-suspenders: don't rely on memory of add-order z-semantics
```
Remove the moved link (`this.c1CommandLink79`, in this example) from `c1MainMenu1.CommandLinks.AddRange(...)`, and drop any `Delimiter = true` it had — a leading separator made sense next to a sibling item in the old menu, not as the sole item in its own toolbar.

**Checklist / rule:**
- Don't search for a per-item "right align" property on `C1CommandLink`/`C1MainMenu` — it doesn't exist, confirmed by exhaustive reflection, not just missed IntelliSense.
- A second `C1ToolBar` (or `C1MainMenu`) anchored `Top | Right`, sharing the same `C1CommandHolder`, is the standard workaround — it keeps the existing `C1Command` (Click handler, `Visible`, `UserData`, `Image`, etc.) completely untouched; only the link's *host container* changes.
- Z-order matters: a `Dock = DockStyle.Top` main menu spans the full form width, so a right-anchored sibling toolbar sitting in that same band must be added to `Controls` before the menu (or `BringToFront()`'d explicitly) or the menu will silently paint over it.
- This can't be visually verified from reflection/compilation alone — a build succeeding only proves the wiring compiles, not that the pixel alignment/seam looks right. Open the form in the VS WinForms designer (or run the app) to fine-tune the toolbar's guessed width/position.

Found in CWATIS 2026-09-18, `MainForm.Designer.cs` (`updateCommand` "new version available" notification, previously the last, delimiter-prefixed item in `c1MainMenu1`).
