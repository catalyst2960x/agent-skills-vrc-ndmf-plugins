# Inspectors and Drawers Reference

How a plugin's components are drawn in the Unity Inspector: the
CustomEditor classes and PropertyDrawers that present a component's
serialized fields to the author. The build-time behaviour of the
component is specified by the procedures; this reference covers the
editing surface only.

## Placement and shape

- Inspector and drawer classes live in the product's Editor assembly
  (P-01), one CustomEditor per component type.
- A field type that appears in more than one component — a condition
  row, a parameter-name field, an assignment row — gets one
  PropertyDrawer, written once and shared by every component that
  contains the type. The same data is drawn the same way everywhere
  it appears, in every product.
- Sections appear in one order across the product's components:
  the fields the effect acts on, the settings, the actions, the
  validation display. Headings and groupings use one style
  throughout.

## SerializedProperty discipline

- Draw and write only through SerializedObject and
  SerializedProperty: Update the serialized object, draw with
  PropertyField or the property's drawer, then
  ApplyModifiedProperties. Undo, prefab overrides, and multi-object
  editing all ride on this path.
- Never assign a component's fields directly from inspector code,
  and never mutate the component while drawing. A change the
  serialized path does not see is a change undo cannot reach and a
  rebuild may silently lose.
- Array size and element order change only through the serialized
  array property; lists are drawn as reorderable lists with the
  add/remove controls in the list footer.

## Suggestions and free text

- Where a value can be derived from the avatar — a component of a
  known type on the same object, a parameter the avatar already
  provides — offer it as an explicit action in the inspector (a
  button or a suggestion list) that writes through the serialized
  property. The author sees and accepts each suggestion; the
  inspector never fills a field silently, and never overwrites a
  value the author already set.
- Fields whose values are names stay free text with suggestions
  beside them, never a closed dropdown: a name the suggestions
  cannot know must remain enterable by hand (Loose coupling to
  referenced sets, in SKILL.md).

## Validation display

- The inspector mirrors the build's validation (Error Reporting):
  the same defects, at the same severities, drawn as help boxes on
  the component they belong to. An author who fixes what the
  inspector shows meets no new defect at build time.
- The build report stays authoritative. The inspector does not fix
  values silently, does not block saving, and does not promise a
  defect is absent — it shows the defects its checks can see, and
  inapplicability draws nothing (Error Reporting, Validation
  policy).

## Edit-time actions

- A button that generates at edit time writes exactly what the
  build pass writes, through the same code path, with the same
  presence check, so the edit-time and build-time routes compose
  (Expression Parameters and Expressions Menu, Edit-time
  pre-generation). Two code paths that write the same output drift
  apart; one path, called from both, does not.
