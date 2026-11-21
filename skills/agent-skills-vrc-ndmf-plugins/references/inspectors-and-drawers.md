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

The inspectors and drawers this reference describes are IMGUI. A
UIElements inspector is a different form — it builds a visual
tree and binds it to the serialized object instead of drawing
controls — and is outside this reference's scope; a UIElements
inspector additionally composes its font from OS fonts, because
the default font renders some languages' glyphs incorrectly
(Japanese among them).

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
- One test decides whether a value lives on the component or
  in the drawer: if the built avatar requires the value to be
  what it is, it is component state and is serialized; if the
  built avatar is the same without it — the list selection, an
  edit mode, a cache — it is drawer state and is not serialized
  on the component.

## Shared presentation

Similar operations use the same UI in every plugin: a list is
edited the same way, an element is drawn the same way, so an
author who learned one inspector reads the others. When a new
inspector needs an operation an existing one has, reuse its
form.

### Lists

- A list of entries is a ReorderableList, drawn and edited
  through serialized properties. ReorderableList lives in
  UnityEditorInternal; in a file that imports it,
  AnimatorControllerParameterType is written fully qualified
  (Anti-Patterns 18).
- In general a list accepts drops of the objects it holds.
  Draw the list with an explicit rect and use that rect as the
  drop area; while a drag carries at least one acceptable
  object, show the copy visual mode over it. Dropped objects
  are appended at the end through the serialized property, so
  several objects dropped together register in one operation.
  Dragged objects of other types are ignored.
- An element added through a SerializedProperty arrives
  uninitialized — the element class's field initializers do
  not run for it — so the fields an entry needs are set
  explicitly on append, from the product's declared defaults.
  What else appending does — whether an object already in the
  list is appended again, and the like — is the product's
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
- A drawer that keeps state — nested drawers, a computed
  layout — is cached against its property and reused only
  while both hold: the managed reference it was built for is
  the same object, and the property's serialized contents are
  still equal (SerializedProperty.EqualContents). When either
  fails, the drawer is disposed and a fresh one created.
  Disposal cascades to the drawers it owns and releases what
  it acquired — scene-editing state, temporary objects — so a
  discarded drawer leaves nothing engaged.
- A GUIContent instance shared between controls is a scratch
  buffer: set its text for each control as it is drawn, and
  never let one control's label ride into another through a
  retained instance.
- A field that commits on focus loss (a delayed text or float
  field) in a list whose selection can change during the edit
  first takes a passive control ID hashed from the property
  path; without it, a pending edit can land on the element
  the selection moved to.
- A tool that enumerates a component's members by walking its
  SerializedObject walks with a single iterator: a copied
  iterator (Copy()) logs an invalid-iteration error. The walk
  does not surface the component's enabled flag (m_Enabled);
  a tool that offers it presents it through FindProperty
  explicitly.

### Suggestions and conflicts

- A suggestion row shows the suggested name with its type as a
  suffix and a fixed-width apply button; applying appends and
  does not edit existing entries (Expression Parameter Input).
- A name conflict is presented as a Warning HelpBox carrying a
  button that offers the deduplicated name (Expression
  Parameter Input): not as an error, and never resolved
  silently.

## Localized labels

- A product that localizes its inspector uses NDMF's
  localization framework (nadena.dev.ndmf.localization), not a
  vendored library: one Localizer, held statically in the
  Editor assembly, constructed with the default language
  (en-US) and a loader that returns the product's
  LocalizationAssets. Creating it statically registers its
  locales with NDMF's global language registry when the
  assembly is first touched.
- The loader enumerates each PO asset explicitly by GUID
  (AssetDatabase.GUIDToAssetPath and LoadAssetAtPath); there
  is no directory discovery, so adding a locale means adding
  its asset's GUID to the loader. Translations are PO files
  named by locale code, imported as LocalizationAsset; the
  header's Language field sets the asset's locale.
- Every locale file carries the same key set, and every key
  the UI uses exists at least in the default language's file:
  a key found in no table renders as <key>, so a missing
  entry is a visible defect, not a silent one.
- A key is a kebab-case identifier naming the element and its
  owner, never display text; the default language's file maps
  keys to text through the same table. The framework performs
  no formatting: an entry that needs values carries {0}-style
  placeholders, and the caller applies string.Format.
- Lookup resolves against the globally selected language in
  this order: the exact locale, then every loaded locale
  sharing its base language, then this Localizer's default.
  The selection is NDMF's single global preference: the
  product draws no language picker, keeps no preference of
  its own, and never writes the global value.
- Labels are fetched through the Localizer at draw time, so
  a language change applies on the next repaint without
  rebuilding the control tree.
- Error messages use the same framework, through the
  Localizer member of the error type (Error Reporting); the
  passthrough form described there is what a product uses
  when it ships no PO assets.
