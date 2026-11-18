# Changing a design-time property's type in a shared package: compiles fine downstream, throws `InvalidCastException` from `InitializeComponent()` at runtime

**Symptom**: after a property on a reusable control/form (published as a package, e.g. `MC_GlobalForms`,
or just consumed by another project in the same solution) changes type — here
`List<int>` → `List<string>` — a **consuming** form that never even sets that property explicitly in
its own C# code still crashes on construction:

```
System.InvalidCastException
  Message=Не удалось привести тип объекта "System.Collections.Generic.List`1[System.Int32]"
  к типу "System.Collections.Generic.List`1[System.String]".
  Source=CWATIS
   at CWATIS.Forms.Contracts.View.ContractsForm.InitializeComponent() in ...ContractsForm.Designer.cs:line 1909
   at CWATIS.Forms.Contracts.View.ContractsForm..ctor(...) in ...ContractsForm.cs:line 201
```

The project **builds and compiles cleanly** both before and after the type change, on both sides
(package and consumer) — this is not a compile error, so a green build proves nothing here. It only
surfaces the moment that specific form is actually opened/constructed at runtime, so it can sit latent
for a while, especially on a form that isn't opened often, or after a version bump nobody manually
retested through every screen.

**Root cause**: at some point, a developer set the property via the VS Designer's Properties grid
(not in hand-written code) on a form/control instance. Because the property's value at design time
wasn't a simple literal the Designer could inline as C# source, it got serialized as a **binary blob**
into the form's `.resx`, with the generated `.Designer.cs` reading it back via
`resources.GetObject(...)`:

```csharp
// .Designer.cs
this.mSubDivisionsLD1.SubdivisionCategories =
    ((System.Collections.Generic.List<string>)(resources.GetObject("mSubDivisionsLD1.SubdivisionCategories")));
```

```xml
<!-- .resx -->
<data name="mSubDivisionsLD1.SubdivisionCategories" mimetype="application/x-microsoft.net.object.binary.base64">
  <value>
      AAEAAAD/////AQAAAAAAAAAMAgAAAJoBbXNjb3JsaWIs...
      ...TGlzdGAxW1tTeXN0ZW0uSW50MzI...   <!-- literally embeds "List`1[[System.Int32" -->
  </value>
</data>
```

That `.resx` entry is a **frozen, already-serialized `List<int>` object**, written whenever that
property was last touched in the Designer (here: an empty list, since the property was never
meaningfully configured on this particular form). Retyping the property's declared type to
`List<string>` in the source (and updating the `.Designer.cs` cast text to match, which is needed just
to compile) does **not** re-serialize this blob — `resources.GetObject(...)` still deserializes the
*old* binary data at runtime, producing a real `List<int>` instance, and the compiled cast to
`List<string>` then fails exactly like casting any other wrong-typed object would. The cast expression
in the `.cs` file is free to say whatever type you like; it has no effect on what type the blob
actually deserializes to.

This is the same underlying mechanism as the already-documented "designer serializes `SelectedRows`
state junk" gotcha in `../../Topics/SmartGridConventions.md` (a stale binary blob nobody meant to keep), just
triggered here by an upstream **type change** instead of stale **selection state**, and in a plain
`UserControl` property with no grid involved at all — so it's a general WinForms/resx risk, not
something specific to SmartGrid.

**Fix**: delete both halves of the stale entry - don't just retype the cast:
1. The `<data name="control.Property" ...>` element in the `.resx`.
2. The `this.control.Property = ((CastType)(resources.GetObject("control.Property")));` line in the
   paired `.Designer.cs`.

If the property was never meaningfully configured on that particular form (as here - an empty list),
removing both is a pure no-op behaviorally: the property just keeps its own field-initializer default
(`= new List<string>()`) instead of being force-fed a deserialized blob. If a form genuinely did
configure a non-default value at design time, that value must be re-entered (in code, or by resetting
the property in the Designer and setting it again so a fresh, correctly-typed blob gets generated) -
don't just delete the entry and assume the default is equivalent without checking what was actually
configured there.

**Checklist / rule**:
- Treat any type change to a property that's `public`/settable and exposed on a `Form`/`UserControl`
  as a potential breaking change for **every** consumer, not just the ones that fail to compile.
  Compiling is necessary but not sufficient - a stale Designer-serialized blob compiles fine and only
  fails at the exact moment that specific form is constructed.
- After such a type change, grep every consuming repo for the property name (or the containing
  control's field name, e.g. `mSubDivisionsLD1`) across **both** `.Designer.cs` and `.resx` files, not
  just `.cs` - a `.resx`-only hit (no matching plain-code assignment) is the tell that the value came
  from the Properties grid, not a line someone wrote by hand, and is exactly where this bites.
- A `mimetype="application/x-microsoft.net.object.binary.base64"` `<data>` entry that decodes to (or
  visibly contains, as readable ASCII inside the base64) the *old* type name is confirmation, not just
  suspicion - the fix is deleting the entry, not patching a cast.

Found in GlobalForms + CWATIS, 2026-08-26: `MC_GlobalForms.Forms.CategorizedSubdivisions.View.CategorizedSubdivisionsForm.AllowedCategories`
changed from `List<int>` (category ids) to `List<string>` (category codes). CWATIS's
`CWATIS.Controls.Subdivisions.View.SubdivisionPickerControl.SubdivisionCategories` (which forwards
straight into `AllowedCategories`) was updated to match and compiled fine everywhere - but
`ContractsForm.Designer.cs`/`.resx` (`mSubDivisionsLD1`, one of two consumers of the control in
CWATIS) had a Designer-set (empty) value baked in from before the type change, and threw this
exact `InvalidCastException` on every `ContractsForm` open. The sibling consumer,
`ContractAddForm`, never had a Designer-set value for this property at all, so it was unaffected -
a reminder that this class of bug can affect some consumers of a shared control and not others, even
within the same repo.
