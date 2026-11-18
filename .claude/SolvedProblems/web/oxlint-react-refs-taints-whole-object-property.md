# oxlint react(refs) false-positive: taints every property of an object once one property is used as a `ref`

## Symptom

`npm run lint` (oxlint) prints, repeatedly, for lines that are plain data/handler
reads - not ref access:

```
warning react(refs): Cannot access refs during render
help: React refs are values that are not needed for rendering. Refs should only
be accessed outside of render, such as in event handlers or effects. Accessing
a ref value (the `current` property) during render can cause your component
not to update as expected
```

Flagged lines look like `thread.commentText`, `onSubmit={thread.submitComment}`,
`thread.attachments.length` - none of them touch `.current` or any ref at all.
Only shows up in `npm run lint` output; `tsc -b`/`vite build` and runtime
behavior are unaffected.

## Root cause

The rule does not track which specific property path was used in a `ref={...}`
JSX attribute. It taints the whole base identifier instead: once any
`x.someProp` MemberExpression appears directly inside a `ref={}` slot (e.g.
`ref={thread.commentInputRef}`), the rule treats every *other* `x.otherProp`
access anywhere else in that component as if it were also a ref access, even
though `otherProp` is unrelated plain state living on the same object.

This surfaces specifically when a custom hook bundles a real `useRef(...)`
object alongside plain state/handlers into one returned object (an "opaque
handle" a component destructures from, e.g. `const thread =
useSomeHook(...)`), and a component both passes `thread.someRef` to a `ref`
prop AND reads other `thread.xxx` fields elsewhere for rendering or event
handlers.

Callback refs (`ref={(el) => ...}`) do **not** trigger this - only a direct
MemberExpression in the `ref` slot does.

## Fix

Destructure the ref out into its own local binding before the JSX, and pass
the bare local variable (not a base-object property access) to `ref`:

```tsx
// Before - taints every other thread.xxx access below with a false "ref" warning
<textarea ref={thread.commentInputRef} value={thread.commentText} ... />

// After - clean
const { commentInputRef } = thread
<textarea ref={commentInputRef} value={thread.commentText} ... />
```

No behavior change - this is a pure lint-satisfying refactor.

## Checklist / rule

- If oxlint's `react(refs)` fires on a line that clearly isn't touching
  `.current`, check whether the *same* component also does
  `ref={someObject.someProperty}` anywhere - that's very likely the real
  trigger, not the flagged line.
- When a custom hook's return value bundles a `useRef` alongside plain
  state/handlers and gets passed around as one object prop, destructure just
  the ref field out at the top of any component that wires it to a `ref`
  prop, even though every other field stays accessed via the bundled object.
- Confirmed 2026-09-15 in `feedback-app` (React 19 + oxlint) while extracting
  a `CommentComposer` component from `SuggestionDetailView`:
  `useCommentThread()`'s returned object carried a `commentInputRef` next to
  ~25 plain fields; passing that whole object as a `thread` prop and writing
  `ref={thread.commentInputRef}` produced 11 false-positive warnings across
  unrelated lines in the same component.
