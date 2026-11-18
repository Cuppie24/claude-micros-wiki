# Tauri `Window::center()` centers on work area; `monitor.size()`/`position()` give the full screen

**Symptom**: a Tauri popup window that is manually centered on show (e.g. cursor-based
multi-monitor centering) visibly jumps a small but consistent amount (commonly ~half the
taskbar height, e.g. 20-25px) vertically every time it resizes afterward — even though each
individual centering computation looks correct in isolation. Debug logging of `outer_pos` after
each op shows two different "centered" y-values for the same window size, alternating.

Found in CupTranslate (Tauri v2, `net10`-era, Windows), 2026-08-11.

**Root cause**: Tauri's built-in `Window::center()` (and `WebviewWindow::center()`) computes the
target position from `monitor.work_area()` (screen minus taskbar/OS reserved space) —
confirmed in `tauri-runtime-wry-2.11.3/src/window/mod.rs`,
`calculate_window_center_position()`:

```rust
pub fn calculate_window_center_position(
  window_size: PhysicalSize<u32>,
  target_monitor: MonitorHandle,
) -> PhysicalPosition<i32> {
  let work_area = target_monitor.work_area();
  PhysicalPosition::new(
    (work_area.size.width as i32 - window_size.width as i32) / 2 + work_area.position.x,
    (work_area.size.height as i32 - window_size.height as i32) / 2 + work_area.position.y,
  )
}
```

If any other code in the same app does its own centering math using `monitor.size()` /
`monitor.position()` (the *full* monitor bounds, taskbar included) instead of
`monitor.work_area()`, the two "centered" positions disagree by half the excluded taskbar
height. A common pattern that triggers this: custom cursor-based "center on whichever monitor
the mouse is on" logic (needed because `center()` only centers on the window's *current*
monitor, not the one the user is looking at) written against `monitor.size()`, combined with
plain `window.center()` called elsewhere (e.g. after a `ResizeObserver`-driven resize). The
window will look correctly centered right after `show()`, then hop a few pixels on the very
next resize.

**Fix**: make all manual centering math use `monitor.work_area()` (`.position` and `.size`
fields on the returned `PhysicalRect`), matching what `center()` itself uses internally:

```rust
let work_area = monitor.work_area();
let x = work_area.position.x + (work_area.size.width as i32 - size.width as i32) / 2;
let y = work_area.position.y + (work_area.size.height as i32 - size.height as i32) / 2;
```

**Checklist / rule**: any time an app mixes Tauri's built-in `center()` with hand-rolled
centering (multi-monitor-aware, cursor-aware, etc.), make sure both use the same reference
frame — `work_area()`, not `size()`/`position()`. A steady small offset between two otherwise-
correct-looking centering calls is the signature of this mismatch, not a rounding/DPI bug.
