# A leftover/legacy stylesheet's `@media (prefers-color-scheme: dark)` rule isn't gated by the app's own forced-theme attribute, so headings go near-invisible when OS is dark but the app is forced light

**Symptom** — A specific element's text (e.g. an `<h1>` page title) renders in a very pale/washed-out color while sibling text right next to it (e.g. a subtitle) renders normally dark and legible. No console error, no obviously-wrong CSS on the affected class itself — the class in question simply never declares `color`. Reproduces only for users whose OS color-scheme preference is dark **and** who have explicitly forced the app's theme to light via its own in-app theme switcher; users on system/light OS, or those who left the app on "system" theme, never see it (because in that case both text and background go dark together, and it still reads fine).

**Root cause** — The app has two independent sources of `color` for the same element:
1. Its own design-system tokens (e.g. `tokens.css`), which correctly gate the dark-mode override with something like `:root:not([data-theme="light"])` — so an explicit `data-theme="light"` on `<html>` overrides the OS preference, as intended.
2. A second, older/unrelated stylesheet (leftover from a starter template, a different page, a global reset) that also happens to style the same bare element selector (e.g. a generic `h1, h2 { color: var(--text-h) }`) — but its own `@media (prefers-color-scheme: dark)` block is **unconditional**, with no `data-theme` guard at all.

CSS specificity is what makes this invisible in code review: the design-system's own component class (`.fb-page-title`) has higher specificity than the bare `h1` selector, so it looks like it "wins." But if that component class never actually declares `color` itself, the cascade falls through *past* it to the next rule that does declare `color` for that element — the legacy bare-selector rule — regardless of the class's higher specificity on other properties. The class only overrides `color` if it explicitly sets `color`.

Net effect: the background correctly stays light (driven by the properly-gated token), but the text color flips to the *legacy stylesheet's* dark-mode value (typically a near-white) because that one rule's dark-mode override isn't aware `data-theme="light"` exists at all — near-white text on a white background.

**Fix** — Don't fix this by touching the legacy stylesheet's media query (that file may be shared/serve unrelated pages and gating it correctly is a bigger, riskier change). Instead, make every design-system component that renders a bare heading element declare its own `color` explicitly from the correct token, so it never depends on falling through the cascade to an unrelated stylesheet:

```css
.fb-page-title {
  font-size: var(--font-size-heading);
  font-weight: var(--font-weight-semibold);
  color: var(--color-text); /* was missing - fell through to a legacy `h1,h2{color:...}` rule */
}
```

**Checklist / rule**
- When a design-system component class styles a bare HTML element (`h1`, `h2`, `button`, etc.) but doesn't declare every property the design intends, check whether some *other*, unrelated stylesheet in the same app also targets that bare element — the cascade will silently fill the gap from there, specificity notwithstanding, for whichever properties your own class left undeclared.
- To confirm/reproduce a "text is invisible in one theme combination" report: emulate the OS color scheme (e.g. Playwright `colorScheme: 'dark'`) *and* force the app's own theme override in the direction that conflicts with it (e.g. write the app's light-theme value straight into whatever `localStorage`/attribute mechanism it uses, then reload) — checking `getComputedStyle(el).color` against the background color under that specific combination is what actually surfaces the bug; testing OS-dark with the app left on "system" won't reproduce it, because in that case both properties go dark together and still look fine.
- The fix belongs on the component that has an incomplete style declaration (make it self-contained), not on the leaking legacy stylesheet, unless that stylesheet is being actively maintained as part of the same design system.

Found in: feedback-app (micros), 2026-09-11 — `index.css` (leftover Vite starter-template CSS, its own separate `:root`/`--text-h` tokens unrelated to the app's `tokens.css`) had an ungated `h1, h2 { color: var(--text-h) }`; the feedback app's own `.fb-page-title`/`.fb-detail-title` classes never declared `color`, relying on inheritance from `.fb-app { color: var(--color-text) }` — which normally works, but loses to the legacy element-selector rule once that rule's own (ungated) dark-mode override kicks in.
