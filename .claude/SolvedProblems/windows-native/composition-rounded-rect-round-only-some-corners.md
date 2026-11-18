# Windows.UI.Composition: rounding only some corners of a CompositionRoundedRectangleGeometry (and ShapeVisual's implicit clip)

**Symptom:** A `ShapeVisual`-drawn pill/card needs square top corners and rounded bottom corners (e.g. a notch hanging from the screen edge), but `CompositionRoundedRectangleGeometry.CornerRadius` is a single `Vector2` (x/y radius) applied to **all four** corners - there is no per-corner radius anywhere in `Windows.UI.Composition`. The obvious alternatives are all worse: a `CompositionPathGeometry` needs a Direct2D/Win2D path that can't be bound to the visual's animating `Size` via an `ExpressionAnimation` (you'd have to rebuild the path per frame or switch to `PathKeyFrameAnimation`), and stacking a plain rectangle over the top half of a rounded one double-blends the overlap for any non-opaque brush.

**Root cause / key fact:** `ShapeVisual` **clips its shapes to its own `Size`** (nothing drawn outside `[0, Size]` in the visual's local space is visible, and that clip scales with the visual's `Scale`/`CenterPoint`). Not stated on the class's doc page; verified empirically 2026-09-21 by giving the visual a 30 px top margin and screenshotting - the geometry extended above the visual's top edge was cut off flush at the edge, square corners and all. So a geometry can be deliberately oversized so that the corners you *don't* want fall outside the visual's bounds.

**Fix:** Keep the single rounded rectangle, but make it one corner radius taller than the visual and shift it up by that radius. The top corners land above the visual's top edge and get clipped; the visible remainder is square on top, rounded at the bottom. Everything stays bound to `pill.Size` by expressions, so it keeps reflowing as the visual animates:
```cpp
auto bgGeometry = compositor.CreateRoundedRectangleGeometry();

// radius = half the (animating) height, as before
auto radiusExpr = compositor.CreateExpressionAnimation(
    L"Vector2(Min(pill.Size.X, pill.Size.Y) / 2, Min(pill.Size.X, pill.Size.Y) / 2)");
radiusExpr.SetReferenceParameter(L"pill", g_pill);
bgGeometry.StartAnimation(L"CornerRadius", radiusExpr);

// shifted up by one radius ...
auto bgOffsetExpr = compositor.CreateExpressionAnimation(
    L"Vector2(0, -Min(pill.Size.X, pill.Size.Y) / 2)");
bgOffsetExpr.SetReferenceParameter(L"pill", g_pill);
bgGeometry.StartAnimation(L"Offset", bgOffsetExpr);

// ... and one radius taller, so the top corners sit outside the ShapeVisual's clip
auto bgSizeExpr = compositor.CreateExpressionAnimation(
    L"Vector2(pill.Size.X, pill.Size.Y + Min(pill.Size.X, pill.Size.Y) / 2)");
bgSizeExpr.SetReferenceParameter(L"pill", g_pill);
bgGeometry.StartAnimation(L"Size", bgSizeExpr);
```
Mirror the offsets (shift down / left / right instead) to square off the bottom or a side instead. `-Min(...)` - unary minus - is valid in the expression language.

**Checklist / rule:**
- `CompositionRoundedRectangleGeometry` = one radius for all corners; don't hunt for a per-corner property.
- A `ShapeVisual` clips to its `Size` - rely on that (oversize the geometry, push the unwanted corners out of bounds) before reaching for `CompositionPathGeometry`/Win2D or an explicit `InsetClip`/`GeometricClip`. A `ContainerVisual`/`SpriteVisual` parent does **not** clip its children; only set an `InsetClip` if the geometry has to live on a non-`ShapeVisual`.
- Hit-testing/layout code that reads the visual's `Size`/`Offset` is unaffected - the visual's bounds remain the visible bounds; only the geometry inside is oversized.
- If the visual gets a press-feedback `Scale`, the clip scales with it, so the hidden corners stay hidden; put `CenterPoint` on the edge the shape is "attached" to (e.g. `Vector3(Size.X / 2, 0, 0)` for a top notch) so the scale doesn't visibly detach it from that edge.

Found in cup-island 2026-09-21 (`prototype/main.cpp`, `BuildScene` - Win32 + Windows.UI.Composition dynamic-island prototype).
