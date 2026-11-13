# Manipulation Techniques Reference

How a plugin makes an avatar change at runtime — objects appearing and
moving, blendshape weights, and shader (material) property values — plus
the static build-time variant. Runtime change is always parameter-driven
and must be generated: there is no runtime code. Two generation routes
exist, and they compose per property, not per product:

- **Reactive route (dependency case 2).** Emit a Modular Avatar (MA)
  Reactive Component describing the desired end state; MA compiles all reactive
  components into animation during its Transforming passes, arbitrating
  when several components touch the same property. A reactive component
  applies its effect while it is active — activation follows the active
  state of the GameObject it sits on, as driven by menus and animations —
  and every reactive component carries an Inverted option that flips
  that. The base class is not publicly extensible (Modular Avatar
  Integration), so the built-in components below are the vocabulary.
- **Manual route (both cases).** Generate AnimationClips holding the
  property bindings below, and drive them with the layer/state pattern
  or a Direct Blend Tree layer (Animator Generation) — blend trees
  also carry the discrete values there. The
  plugin owns arbitration: two layers animating the same property fight,
  and layer order decides the winner.

Static manipulation is the third option for changes that do not need a
parameter: the pass edits the build-time clone directly (activate,
reparent, delete, replace materials), registering replacements with the
ObjectRegistry so later passes and other plugins see the new objects.

## Direct editing and the framework surface

Editing mesh data (vertices, normals, UVs, blend shape frames, bone
weights and bind poses), texture pixels, or a cloned material's
values has no NDMF or Modular Avatar API. The framework surface for
such work is the two NDMF services and the Modular Avatar components
named here; the data editing itself is Unity API work on the cloned
asset.

- NDMF: register the replacement with ObjectRegistry and persist
  created assets through the build's asset saver (Framework;
  Converting shared assets in a pass). Nothing in NDMF reads or
  writes mesh, texture, or material data.
- Modular Avatar's manipulation components are the case 2 route.
  ModularAvatarMeshCutter removes the vertices its built-in filters
  select (by axis, bone, mask texture, blend shape, or UV tile)
  during reactive processing; the filtering internals are internal
  to Modular Avatar, so a plugin emits the component and does not
  register filters of its own. ModularAvatarBoneProxy retargets the
  bone a binding follows. ModularAvatarMeshSettings sets a mesh's
  root bone and bounds. Material changes use the material reactive
  components (Modular Avatar Integration).

## Converting shared assets in a pass

The recurring shape of a conversion pass (a material swap is the
common case):

- Collect every distinct original across BOTH live references
  (renderer materials, component fields) and animation curves, in one
  de-duplicating cache keyed by the original object, so the same
  original always maps to the same clone. Then clone, change the
  clone, and rewrite every reference through that one mapping;
  register each replacement with the ObjectRegistry (Framework).
  Rewriting only the live values leaves the animated overrides
  pointing at the original, and the avatar snaps back the first time
  such an animation plays.
- The animation side goes through the animator-services layer:
  activate AnimatorServicesContext ([DependsOnContext] on the pass,
  or ActivateExtensionContextRecursive when a context is built by
  hand) and use AnimationIndex.RewriteObjectCurves with the same
  mapping for object-reference curves. Numeric properties are float
  curves, which RewriteObjectCurves does not touch; a value the pass
  forces that a float curve also drives needs the curve neutralized
  through VirtualClip, or the animator keeps overriding it.
- A clone of a Material Variant is still a variant: changes the
  variant system locks (such as the shader) are silently ignored on
  it. Clear the clone's parent first, which bakes its resolved
  values into a standalone material, then change it.
- Never round-trip a property through a getter that resolves several
  underlying states into one value in order to "preserve" it across
  a change (Material.renderQueue is the standard example): the
  resolved value comes back as an explicit override and silently
  replaces what the new configuration would have chosen. The copy
  constructor already carries the real override state, including
  "no override".

## Objects

| Technique | Binding / component | Notes |
| --- | --- | --- |
| Toggle active state | Reactive: ModularAvatarObjectToggle, one entry per object with its desired active state. Manual: a float curve on the GameObject, property `m_IsActive` | Define the property in every state of the controlling layer; a state that does not animate it inherits whatever another layer last set |
| Enable/disable a component | Manual: a float curve on the component's object, type = the component type, property `m_Enabled` | Works for renderers, lights, constraints, audio sources; the component keeps existing, only its enabled flag moves |
| Move an object | Reactive: ModularAvatarReactiveMove | Moves the object to a target position/parent while active; combine with ObjectToggle patterns rather than animating transforms by hand |
| Scale compensation | ModularAvatarScaleAdjuster (not reactive) | Bakes a scale correction at build time; a static manipulation, listed here because authors reach for it alongside toggles |
| Remove geometry | Reactive: ModularAvatarMeshCutter | Build-time mesh processing while active, not an animation |

## BlendShapes

| Technique | Binding / component | Notes |
| --- | --- | --- |
| Set a weight while active | Reactive: ModularAvatarShapeChanger, one entry per shape (renderer, shape name, change type Set, target value). Manual: a float curve on the SkinnedMeshRenderer, property `blendShape.<ShapeName>` | Weights run 0..100, not 0..1 — the most common unit mistake in generated clips. The component also carries a Threshold used when the controlling value is analog |
| Delete a shape while active | ShapeChanger with change type Delete | Build-time mesh processing: the shape is removed from the mesh, not animated to zero |
| Mirror another mesh's shape | ModularAvatarBlendshapeSync: a reference mesh + shape name bound to a local shape, with an optional remap curve | For outfits and add-ons that must follow the base body's expressions; MA evaluates the binding during the build |

Shape names are per-mesh: the same name on a different renderer is a
different binding, and a name that does not exist on the target mesh
generates a curve that silently does nothing. Validate names against the
target mesh's actual blendshape list during the pass and report misses
through ErrorReport.

## Shader parameters (material properties)

A shader parameter is animated through the renderer that uses the
material, never through the material asset itself:

| Technique | Binding / component | Notes |
| --- | --- | --- |
| Animate a float/color/vector property | Manual: a curve on the renderer's path, type = the renderer's type, property name `material.<PropertyName>`; colors and vectors animate per component (`material.<PropertyName>.r`, `.g`, `.b`, `.a`) | The property name must match the shader's declared property exactly, including the leading underscore. Driven by a state (on/off values) or by a blend tree over a float parameter for continuous control — the pattern menu generators use for radial-driven shader values |
| Set a material while active | Reactive: ModularAvatarMaterialSetter, one entry per object (object, material slot index, material) | Slot-level replacement; the rest of the renderer's materials are untouched |
| Swap materials while active | Reactive: ModularAvatarMaterialSwap, a root object plus from→to material pairs | Every occurrence of the From material under the root is replaced |

Two conditions decide whether a generated material animation works at
all. First, the animated material must be the avatar's own: if the same
material asset is shared with other objects or other avatars, the
animation still binds per renderer, but build-time material processing
by other plugins may clone it — resolve the material through the
ObjectRegistry at generation time rather than caching the asset.
Second, nothing about a shader parameter is synced by itself: sync is
inherited from the animator parameter driving the state or blend tree,
so the driving parameter's declaration (Expression Parameters and Expressions Menu) is part of
the design.

Modular Avatar has no general-purpose reactive component for arbitrary
shader properties in the pinned version; continuous shader control is
manual-route work. What the reactive route covers is material set and
swap, above.

## Choosing per property

Decide per manipulated property, and record the choice: reactive when a
built-in component expresses the end state exactly (toggle, shape set,
material set/swap), because MA then owns arbitration, preview, and the
generated layers; manual when the value is continuous (a dial sweeping a
blendshape weight or a shader float), when the target is a component's
enabled flag, or in dependency case 1 where reactive components do not
exist. Mixing routes on the same property double-drives it — the manual
layer and MA's generated layer both write it, and the result depends on
layer order, which neither side documents as a contract.
