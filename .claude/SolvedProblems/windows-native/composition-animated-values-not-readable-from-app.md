# Windows.UI.Composition: animated values can't be read back on the app side (AnimationController.Progress stays 0)

**Symptom:** Something outside the compositor - here a Win32 interaction window that must hug an animating `Visual` (see [[win32-overlay-httransparent-does-not-pass-clicks-cross-process]]) - needs the *current* animated `Size`/`Offset` of a visual mid-animation. Reading `visual.Size()` returns the target value the app set (that's documented and expected). The obvious next thing, `visual.TryGetAnimationController(L"Size").Progress()`, looks like a live progress read but returns `0.000` on every poll for the entire run, and the controller keeps being returned even after the animation has finished. Nothing throws; it just never moves.

**Root cause:** The app-side Composition objects are write-through proxies; the compositor thread never pushes animated state back. `AnimationController.Progress` is a *control* property (set it after `Pause()` to scrub), not a telemetry channel. Verified 2026-09-21 by polling `Progress()` every 8 ms through a 300 ms implicit `Size` animation: 130+ samples, all `0.000`. There is no supported API that returns a visual's live animated property value (`Pause()`-then-read snaps the value, and stopping an animation reverts the property to the app-set value rather than freezing the current one).

**Fix:** Mirror the animation on the app side instead of sampling it, and clock the mirror from the compositor's frame ticks so it advances when the picture does:
```cpp
// Shared by the implicit animation and the mirror - one source of truth.
constexpr auto kMorphDuration = 300ms;
constexpr float2 kEase1{ 0.16f, 1.0f }, kEase2{ 0.30f, 1.0f };
auto easing = compositor.CreateCubicBezierEasingFunction(kEase1, kEase2);

// App-side evaluation of the same CSS-style cubic bezier: bisection on x, then y.
float MorphEase(float t) { /* see prototype/main.cpp */ }

// Per-frame tick source: a worker blocked on the compositor clock, posting one
// message per composed frame while a morph is in flight and sleeping on an
// event otherwise (zero idle cost). Requires dcomp.lib; the SDK header gates the
// declaration on NTDDI_VERSION >= NTDDI_WIN10_CO, which _WIN32_WINNT=0x0A00 alone
// already satisfies (it defaults NTDDI_VERSION to the SDK's latest).
DCompositionWaitForCompositorClock(0, nullptr, 100 /* ms, so a stalled clock can't hang the morph */);
PostMessageW(hwnd, WM_APP_COMPOSITOR_TICK, 0, 0);
```
On the UI thread, start the mirror's clock on the **first tick after** the properties were set (the commit carrying them lands on that frame, which is also when the compositor's animation clock starts), interpolate `lerp(from, to, MorphEase(t))` each tick, and snap to the exact target when `t >= 1`. For an interrupted morph use the mirror's own current value as `from` - that's (to within a frame) the `StartingValue` the compositor's new implicit animation picks up too. Precedent: `MorphTracker`/`SyncInteractionLayer`/`CompositorTickThread` in `D:\projects\p\cup-island\prototype\main.cpp`; measured to track a 160→380 px morph frame-by-frame in ~300 ms.

**Checklist / rule:**
- Never design around reading an animated Composition value on the app thread - not via the property getter, not via `AnimationController.Progress`, not via a `CompositionPropertySet` an expression writes to. Only `CompositionScopedBatch.Completed` (end-of-animation) comes back.
- Whatever needs to follow the animation must own a copy of the animation's inputs (endpoints, duration, easing control points) - keep them in shared constants so the two can't drift apart.
- Tick from the compositor clock (`DCompositionWaitForCompositorClock`, or `DwmFlush` on older targets) on a worker that only runs while something is animating, not from `SetTimer`; and gate it with an event so idle cost is nil.

Found in cup-island 2026-09-21 (Win32 + Windows.UI.Composition dynamic-island prototype).
