# Modular Avatar Integration Reference

For products in dependency case 2, where Modular Avatar (MA) is a
requirement and not merely a merge target. Three distinct kinds of access
exist: emitting MA components for MA to process, cooperating with MA's
Reactive Components, and matching MA's editor preview. Modular Avatar
facts are those of the pinned version (SKILL.md, Sources).

## Emitting Modular Avatar components from code

A plugin whose output is expressed as MA components gains MA's processing
for free, at the price of the ordering contract: the components must exist
before the MA passes that consume them, so emission happens in Generating,
ordered BeforePlugin("nadena.dev.modular-avatar") (see Framework).

The implementation route this skill describes is the direct component
APIs: AddComponent<ModularAvatarMergeAnimator>() etc., setting the
serialized fields (animator, layerType, pathMode,
matchAvatarWriteDefaults, deleteAttachedAnimator on MergeAnimator). The
plugin owns every field, and Modular Avatar itself is the only dependency
involved.

What a complete code-driven emission sets, per component type:

- ModularAvatarParameters: one entry per parameter the product
  introduces — nameOrPrefix, the syncType (which also fixes the synced
  value type), localOnly and saved as designed, an explicit default
  value when the design needs one (what an entry without one falls
  back to: Writing parameters), and, when the design uses them,
  remapTo, the Animator
  Only and Prefix entry kinds, bitWidth, representation, and the lerp
  duration. Leaving the settings at their defaults when the design
  needs non-default sync or persistence is a silent bug. Entry
  semantics: Expression Parameters and Expressions Menu (Writing
  parameters).
- ModularAvatarMergeAnimator: the generated controller, the target layer
  type, the path mode, and the write-defaults matching flag.
- ModularAvatarMenuItem: a complete Control (name, icon, control type
  — toggle, button, radial, and the other SDK menu control types —
  parameter, value, and the sub-parameters a puppet binds), the
  MenuSource (child items, a menu asset, or another object's
  children), the label (empty is meaningful: Writing menus), and the
  automatic-creation flags isSynced, isSaved, isDefault, and
  automaticValue (Writing menus, Automatic parameter creation). A
  toggle and a button over
  the same parameter differ only in the control type and value semantics,
  so build both from one description rather than duplicating the wiring.

Reading MA state: prefer the framework surfaces over MA internals —
ParameterInfo (Parameters) for parameters, and the components
themselves for configuration. MA renames and reorganizes during its own
passes; anything read after MA has run must be resolved through the
BuildContext/ObjectRegistry, not through references captured earlier.

## Reactive Components

MA's reactive system turns declarative components into generated animation
at build time; its mesh-processing members are instead evaluated into
the mesh during the same processing (Manipulation Techniques). For the per-target techniques (objects, blendshapes, shader
parameters) and the choice between emitting a reactive component and
generating the animation manually, see Manipulation Techniques. The base type is `ReactiveComponent : AvatarTagComponent`
(Modular Avatar Runtime/ReactiveComponent.cs); the built-in concrete
components include ModularAvatarObjectToggle, ModularAvatarShapeChanger,
ModularAvatarMaterialChanger, ModularAvatarMaterialSetter,
ModularAvatarMaterialSwap, ModularAvatarMeshCutter and
ModularAvatarReactiveMove.

The tracking that drives these components — a parameter that
follows an object's active state — is Modular Avatar's internal
service and not public. A plugin that needs the same input without
emitting reactive components provides its own tracking (Reactive
Tracking): its component is then a generator of its own animator
output, not a reactive component, and MA's reactive system neither
analyzes nor drives it.

Rules for a plugin working alongside or on top of them:

- If the effect a product wants is exactly what a reactive component
  expresses (toggle an object, change a shape key, swap a material), emit
  the reactive component instead of hand-building layers and clips. MA
  generates, optimizes and previews it; a hand-rolled equivalent duplicates
  all three and drifts from MA's behaviour.
- Do not hand-edit the animations MA generates from reactive components,
  and do not read a reactive component's state after MA's reactive passes
  have consumed it. Treat the components as inputs that exist up to MA's
  Transforming-phase processing.
- Parameter names a reactive component responds to are ordinary avatar
  parameters: the same ParameterInfo validation applies to them as to any
  condition parameter.

## Preview

NDMF has a preview system (namespace nadena.dev.ndmf.preview) that renders
avatar effects in the editor without a build:

- A preview is an `IRenderFilter`: `GetTargetGroups(ComputeContext)`
  returns the RenderGroups (sets of renderers processed together) it wants
  to affect, `Instantiate(...)` returns an `IRenderFilterNode` that applies
  the visual change to proxy renderers, and `IsEnabled(ComputeContext)`
  gates it. Set `CanEnableRenderers` when the filter may enable otherwise
  disabled renderers.
- Modular Avatar ships one preview filter per reactive component
  (ObjectTogglePreview, ShapeChangerPreview, ReactiveMovePreview,
  MeshDeleterPreview, and others under Editor/ReactiveObjects), so effects
  expressed as reactive components are visible in the editor with no extra
  work.
- Decide the preview story for every visible effect a product generates
  (SKILL.md, Design Decisions). In order of preference: express the effect
  as an MA reactive component and inherit MA's preview; implement an
  IRenderFilter for the product's component; or state explicitly that
  the product has no preview. "No preview" is
  a legitimate choice for non-visual output (audio, parameter driving),
  but it is a choice to state, not a default to fall into.
- Preview code runs outside the build, repeatedly, on the live scene: it
  must not mutate the author's objects. Work on the proxy renderers and
  materials the filter is given, and dispose of anything created in the
  node's Dispose.
- Three failure modes produce a preview that does nothing while
  raising no error:
  - The proxy renderer is re-copied from the original every frame
    before the filter chain runs, so a material assigned only in
    Instantiate or Refresh is reverted on the next frame. A filter
    that swaps materials must re-apply the already-computed material
    references in the node's OnFrame; creating material instances
    there is what the OnFrame guidance prohibits, not re-assignment.
  - GetTargetGroups must return only MeshRenderer and
    SkinnedMeshRenderer. If even one returned renderer is a type the
    proxy system cannot replace (ParticleSystemRenderer,
    TrailRenderer, LineRenderer, ...), NDMF discards ALL of the
    filter's groups, for every avatar, with only a console error.
  - Reactivity is registered only by queries made through the
    ComputeContext (GetAvatarRoots, GetComponent(s)InChildren,
    Observe). A lookup done with a plain Unity API inside
    GetTargetGroups is invisible to the pipeline, so later edits
    never recompute the preview. This includes finding the filter's
    own settings component: locate it through the context, because
    the avatar-root list itself does not change when a component is
    added to an existing root.
