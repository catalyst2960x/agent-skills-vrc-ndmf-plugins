# Inspectors and Drawers Reference

How the inspector of a plugin's component is structured and
presented. An inspector for an NDMF plugin edits a declaration:
the component on the author's avatar states what the build will
generate, apply, or drive, and every effect materializes later,
on the build copy. That split decides what the inspector does
and does not do; the presentation patterns the plugins share
follow from it. Letting the author configure parameters without
typing names from scratch is Expression Parameter Input; this
reference covers the inspector around those fields.

## Edit time and build time

- Problems with a declaration surface as build reports (P-06),
  not as inspector state. The inspector does not mirror build
  validation: a check whose inputs other passes keep changing —
  the parameter set, the hierarchy — cannot be reproduced at
  edit time, and a mirrored check that disagrees with the build
  misleads in both directions. The inspector's correctness
  surface is the input itself: offer the candidates it can
  detect as suggestions, and keep the fields editable (SKILL.md,
  Loose coupling to referenced sets).
- Reading the avatar at edit time goes through the edit-time
  views (ParameterInfo.ForUI); an editor has no build context.
  What an edit-time view cannot see — a parameter a later pass
  will add, an object a later pass will move — is not an error
  state for the inspector to display.
- The inspector writes the author's component only, through
  serialized properties, so Undo covers every change it makes.
  A change that must land on another component is recorded as
  data on the author's component and applied by a pass on the
  build copy (Expression Parameter Input, assignments).

## Shared presentation

Similar operations use the same UI in every plugin: a list is
edited the same way, an element is drawn the same way, so an
author who learned one inspector reads the others. When a new
inspector needs an operation an existing one has, reuse its
form.

### Lists

- A list of entries is a ReorderableList, drawn and edited
  through serialized properties.
- In general a list accepts drops of the objects it holds.
  Draw the list with an explicit rect and use that rect as the
  drop area; while a drag carries at least one acceptable
  object, show the copy visual mode over it. Dropped objects
  are appended at the end through the serialized property, so
  several objects dropped together register in one operation.
  Dragged objects of other types are ignored.
- An added element starts from the array's copy behaviour, so
  the fields an entry needs are set explicitly on append. What
  else appending does — the defaults it sets, whether an object
  already in the list is appended again — is the product's
  rule, stated where the list is defined.

### Groups and foldouts

- A section heading uses EditorStyles.boldLabel. A group of
  related entries is an EditorStyles.helpBox vertical layout
  with its own boldLabel heading. When several groups of one
  kind would carry the same heading — several components of
  one kind on one object — append the group's ordinal among
  its kind so the headings differ.
- Settings most authors never change go in a foldout after the
  main fields, indented one level. Its initial state is
  decided when the inspector opens: open when any member
  differs from its default, closed while all members hold
  their defaults. A Reset button at the end of the group
  restores the members' defaults, through serialized
  properties.

### Element drawers

- An element type drawn in several places gets one
  PropertyDrawer, reused. The element draws on one line, with
  fixed widths for its controls; the flexible field — a name —
  takes the remaining width.
- A control the element's current mode makes inapplicable is
  omitted, and the width it frees goes to the controls that
  remain.

### Suggestions and conflicts

- A suggestion row shows the suggested name with its type as a
  suffix and a fixed-width apply button; applying appends and
  does not edit existing entries (Expression Parameter Input).
- A name conflict is presented as a Warning HelpBox carrying a
  button that offers the deduplicated name (Expression
  Parameter Input): not as an error, and never resolved
  silently.
