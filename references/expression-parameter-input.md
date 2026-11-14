# Expression Parameter Input Reference

How a plugin's inspector lets the author configure parameters:
fields that hold parameter names, suggestions drawn from the
avatar's parameter set, a default name the plugin supplies when a
field is empty, suggestions derived from components that generate
parameters, and assignments that write a name into another
component's field at build time.

The scope is the edit-time side. A name the author enters is
input and is not validated; that principle is a design decision
(SKILL.md, Loose coupling to referenced sets) and is not restated
here.

## The name field

A field that holds a parameter name is a free-text field.

- The field accepts any name, including a name that resolves
  to nothing. Entering such a name is not an error, and the
  component is not skipped for it.
- In general the field shows, next to the input, the type the
  current name resolves to. The display resolves against the
  same enumeration the suggestions use (below), so the label
  and the list cannot disagree.
- When the name resolves to a parameter whose type is
  undefined, the display shows the type as undefined; a guessed
  type is never presented as resolved.
- When the name resolves to nothing and the plugin defines such
  a name itself with a fixed type, in general the display shows
  that type: it is the type the name will have at build time.
- The display never alters the stored name.

## Suggestions from the UI parameter view

A field may offer the avatar's existing parameters as
suggestions. The enumeration is the UI view of the parameter
set, not the build-time set of P-05: an editor has no build
context, and `ParameterInfo.ForUI` is the surface provided for
this use.

Build the list in this order:

1. Enumerate `ParameterInfo.ForUI.GetParametersForObject` for
   the avatar root.
2. Expand every entry through `SubParameters()`, so a derived
   name appears as its own entry.
3. Exclude entries whose `EffectiveName` is empty, and exclude
   hidden entries.
4. Exclude animator-only entries when the name must denote a
   parameter defined in Expression Parameters: an animator-only
   parameter is not registered there. Other fields in general
   exclude them as well, unless the field can consume an
   animator-only parameter.
5. Collapse duplicates by `EffectiveName` to one entry.
6. Sort the entries by name, with an ordinal comparison.

Presentation:

- The list writes the chosen name into the field's serialized
  property. The field remains free text afterwards; a name that
  is not in the list remains valid input.
- In general each entry shows its type next to the name, in
  the same notation as the field's type display, including the
  undefined marker. The author chooses with the type visible.
  The type shown is the enumerated type; the platform converts
  values between types on some paths, so the display informs
  the choice and does not act as a constraint the field
  enforces.
- In general the list can be searched.
- In general the enumeration is refreshed when the list is
  opened, not only when the inspector is enabled, so parameters
  added since the inspector opened are present.
- When the enumeration is empty, the list in general shows a
  note to that effect instead of entries, and likewise when a
  search matches nothing. An empty list is indistinguishable
  from a failure; the note separates the two.
- The plugin never pre-checks a chosen name against existing
  definitions. A name declared elsewhere with a different type
  conflicts through the normal merge rules (P-13).

## A supplied default name

When a field's name is empty and the feature requires a name,
the plugin may supply a default name derived from data the
component already has, such as the name of the GameObject it
sits on.

- The inspector writes the supplied name into the field itself
  before displaying it, so the name in effect is visible in
  the field and stored in the serialized property. A name that
  exists only inside the build cannot be discovered or changed
  by the author. The fill happens when the field is displayed
  in a configuration that requires a name and the stored name
  is empty.
- The build resolves the same effective name without depending
  on the inspector having run: the resolution (the entered name
  when non-empty, otherwise the supplied name) is implemented
  once, in code shared by the inspector and the build pass, and
  the build never writes the supplied name back into the
  author's component.
- An entered name and a supplied name are treated alike in
  every place the name is used. Components that resolve to the
  same effective name share the parameter.
- The form of a supplied name follows the platform's parameter
  name form (VRChat Avatar Platform, Parameter name form).

## Suggestions derived from generating components

A plugin may scan an object the author selects, and its
children, for components that generate parameters, and offer
the parameters those components will generate as suggestions
for a referencing field, such as a condition.

- Each suggestion derives from the component's own generation
  rule: the prefix or parameter value the component is
  configured with, joined with the suffix and the type the
  component generates for it. A component whose generating
  field is empty generates nothing and produces no suggestion.
- In general the results are ordered by hierarchy path and
  grouped under the component each parameter derives from.
- Applying a suggestion appends a new entry to the referencing
  field; it never replaces or edits an existing entry. When
  the referencing construct has a per-type default form, in
  general the appended entry starts from the form for the
  suggested parameter's type, with a threshold chosen against
  the parameter's value range rather than set to one constant
  for every parameter.
- When a generating component type exists only from a certain
  SDK version, references to it are guarded, for example by an
  assembly version define, so the editor still compiles
  against an older SDK.

## Assignments into another component's field

A plugin may record an assignment: a target component and a
parameter name, meaning that the build writes the name into the
target component's own parameter field. This lets a referencing
field point at a parameter the target will generate once its
field is set, without the author setting two fields by hand.

- The assignment is recorded on the author's component and
  applied by a pass on the build copy. The applying pass runs
  before the build-time parameter set is collected from the
  target components, so the assigned value is part of the set
  (P-05); a pass in the Resolving phase satisfies this.
- The write is conditional: it applies only when the target is
  inside the avatar being built and the target field is still
  empty. A non-empty field is never overwritten. The author's
  scene objects are never the write target; the pass works on
  the build copy.
- A write that cannot apply is skipped and reported as NonFatal
  (P-06): the target outside the avatar, and an assignment whose
  name is already taken by an earlier assignment of the same
  component. For conflicting assignments of one component, the
  first wins. A skip is never silent: the author recorded an
  assignment expecting it to take effect.
- At edit time, when the name an assignment would write is
  already taken, in general the editor offers a deduplicated
  name and never presents the assignment as applicable while
  the conflict stands. The name bookkeeping behind this (the
  names a component generates, each assignment's current base
  name, the taken names, and the deduplication) is in general
  implemented once and shared by the editor and the build pass,
  so the name the editor offers is the name the build writes.
- When a referencing field's name comes from an assignment the
  component records, in general the field's row displays the
  assignment's source component, so the author can find where
  the name is set.
