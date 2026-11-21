# Migration from Stock lilToon

Moving existing materials from stock lilToon to a custom shader is a
reassignment, not a conversion: the custom shader is generated first,
with its full variant set, and materials are pointed at the
corresponding custom variant. Stock and custom shaders coexist; only
the materials that need the custom behaviour have to move.

## What a material carries across a reassignment

A Unity material serialises three things beside its shader
reference, and all three survive assigning a different shader:

- Property values, stored by name. A value carries when the new
  shader declares a property of the same name (and a compatible
  type); a property the new shader does not declare stays in the
  file, unused, and takes effect again if the material is moved
  back.
- The enabled keyword set.
- The render queue.

For lilToon, keywords are not an independent migration item. The
feature system is project-wide defines (lilToon Specification), and
the keywords a material does carry are derived state: the editor's
setup recomputes them from property values (the `IsFeatureOn*` /
`SetShaderKeywords` path in lilMaterialUtils, with
`RemoveShaderKeywords` as the clearing counterpart). Move the
properties and the keywords regenerate to match. The exception is
the Multi family, where material keywords are the mechanism itself —
keywords borrowed from Unity's standard shaders, set from animated
properties — and those live on the material and carry with it,
because the custom Multi variant keeps the same scheme.

What does not carry: anything the new shader expresses differently —
render state and tags come from the new shader, and a variant kind
the custom set does not contain has nowhere to land.

## Requirements

- The custom shader is generated from the template of the *installed*
  lilToon version (P-01). The macro surface is version-bound; a
  template from another version is a mismatch, not an upgrade.
- The full 68-container set is generated before any material moves.
  Trimming (P-03) happens after the move, never before: a trimmed
  variant is a landing place that no longer exists.
- Variant kinds correspond: a material on the stock Full variant
  lands on the custom Full variant, Lite on Lite, Multi on Multi,
  and so on. The property sets differ per kind, and values only
  carry where the kind matches.
- Project-wide lilToon settings need no migration: the same settings
  drive generation and feature defines for custom shaders.
- Distribution gains a dependency: a project receiving the materials
  also needs the custom shader package (P-11).

## Procedure

The steps are P-12 in the Procedure Index. In outline: decide the
scope (which materials, which variant kinds); generate the full set
and implement the custom behaviour; pass the compile verification
(P-10) before touching a material; reassign each material to the
corresponding custom variant — a material is a shared asset, so one
reassignment covers every user of it; verify that values carried
(same-name properties), that the inspector shows the custom sections
and drives them, and that the keyword state regenerated as expected;
only then trim unused variants.

## When the target is a different package

Reassignment works because the template keeps lilToon's property
names. Between different packages the names differ, a plain
reassignment loses most settings, and migration becomes a converter.
Unlit_WF's material converter (WF_MaterialConverter) is the measured
example of what that takes:

- Flatten material variants first, and delete properties that would
  mislead classification.
- Classify the source material from several signals in priority
  order — a mode property, the RenderType tag, shader-name
  substrings, render-queue ranges, the presence of mask properties —
  to decide the rendering kind and feature presence (such as
  outline).
- Select the target shader from the classification, reassign, then
  copy the render queue from the source explicitly.
- Move properties through a rename table: case-insensitive,
  many-to-one (for example `_Cull` and `_Culling` both land on
  `_CullMode`; three ZWrite spellings land on one target).
- Synthesise values the source never had (WF derives its toon shadow
  colours from the main colour's hue).
- Snapshot the material first; on failure, abort and restore the
  whole material from the snapshot.

WF's converter has no lilToon-specific branch: lilToon materials
convert through the generic property names they share with Unity
Standard and UTS. A converter written for a lilToon source would
start from the same parts — classification, selection, queue
preservation, a rename table, synthesis, restore — with lilToon's
property names as the table's source side.
