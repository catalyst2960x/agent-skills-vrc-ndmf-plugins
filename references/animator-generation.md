# Animator Generation Reference

How a pass turns component configuration into an AnimatorController.
Construction uses the raw UnityEditor.Animations APIs
(AnimatorController, AnimatorStateMachine, AnimatorStateTransition)
directly. For larger or more repetitive state graphs, wrap those calls in
the product's own internal helper layer rather than taking a construction
library as a dependency: neither dependency case includes one. Keep the
bookkeeping (asset containers, state identity) in that one layer.

## The layer/state pattern

- One AnimatorController per pass, one layer per component or setting.
  Layer name identifies the source (for example
  `SoundEffect [<GameObject name>]`); set the layer's defaultWeight
  to 1 or the layer silently contributes nothing.
- States: `idle` (the default state), `active`, and optionally `pre_active`
  when a two-stage condition is wanted. Generated states set
  writeDefaultValues = false and share one empty Motion (a Direct
  Blend Tree layer is the exception: VRChat Avatar Platform,
  Playable Layers).
- The empty Motion is a single AnimationClip holding one curve with two
  zero keys (at time 0 and 1/60) on a dummy path; it exists so the states are
  valid without animating anything, and so no state carries a null
  Motion: with Write Defaults off, a state with no Motion can stall
  animation on other layers. The SDK behaviour on the state does the
  real work.
- The behaviour instance goes on `active` only, and carries both the enter
  and the exit settings; there is no second behaviour for exit.

## Transitions and condition reversal

- Forward transitions carry all conditions of their set (AND), with
  hasExitTime = false, hasFixedDuration = true, duration = 0, exitTime = 0.
  With a pre_active stage, idle → pre_active carries the pre-conditions and
  pre_active → active carries the main conditions. The reference pattern
  creates no pre_active → idle return; once the pre-conditions have fired,
  the layer waits in pre_active for the main conditions.
- Return to idle, two options:
  - **Reversed conditions**: one transition per condition, each carrying
    that condition reversed. Separate transitions make the return an OR of
    the negations, which is the correct negation of the forward AND. A
    single transition carrying all reversed conditions would wait for every
    negation at once and is wrong.
  - **Free return**: one condition-less transition with hasExitTime = true,
    exitTime = 0, for designs that should drop back immediately.
- Condition reversal (the negation logic the product's own condition type
  implements; the parameter's type is an input to the reversal because Int
  thresholds shift):

| Forward | Reversed |
| --- | --- |
| If | IfNot |
| IfNot | If |
| Equals | NotEqual |
| NotEqual | Equals |
| Greater | Less; for Int parameters the threshold is +1 |
| Less | Greater; for Int parameters the threshold is −1 |

  The ±1 for Int converts between strict and non-strict comparisons
  (value > t negates to value ≤ t, expressed as value < t+1). Float
  thresholds are unchanged. The parameter type comes from ParameterInfo,
  not from the condition itself.
- Only parameters that exist in the build-time parameter set may be added
  to the generated controller as AnimatorControllerParameters, each once
  (see Parameters).
- An empty condition set is a real configuration, not an error: the
  forward transition carries no conditions and fires unconditionally, and
  the reversed set is empty, so no return transitions are generated and
  the layer stays active. Decide whether that is the product's intent
  (for an always-on effect it is); if a return is required, the design
  needs conditions or the free return.

## Direct Blend Tree layers

The alternative shape for a product with many independent toggles,
selectors, and graded values: one layer replaces the
one-layer-per-component state machines, and there are no transitions
at all.

Build the layer in this order:

1. Register the driver parameter on the controller: a Float with
   default 1, under a name the controller does not already use.
   Every root child's directBlendParameter is set to it in step 5,
   so every sub-tree is evaluated at full weight.
2. Create the root: a BlendTree with blendType Direct,
   blendParameter set to the driver parameter, and automatic
   thresholds off. Build the layer explicitly: a state machine whose
   only state holds the root as its Motion, with writeDefaultValues
   = true — the exception to the avatar-wide rule (VRChat Avatar
   Platform, Playable Layers) — and is set as the state machine's
   default state; and a layer with that state machine, blendingMode
   Override, and defaultWeight 1, added with
   controller.AddLayer(layer).
3. For each feature, build a Simple1D blend tree keyed on the
   feature's own parameter and add it to the root:
   - Toggle: automatic thresholds off. Add the two clips with
     explicit thresholds: the default-state clip at the threshold
     equal to the parameter's declared default (0 or 1) and the
     changed-state clip at the other. Explicit thresholds remove the
     dependence on when automatic threshold recomputation runs,
     which varies with the Unity version and the order children are
     added.
   - Selector: automatic thresholds off; add one clip per value with
     AddChild(clip, i) for value i.
   - Graded value: place each clip at the parameter value it
     corresponds to.
   - Several conditions on one target: nest Simple1D trees, one
     level per condition. End a branch that fails its condition at
     the inactive clip; only the path where every condition passes
     reaches the active clip.
4. Register each feature parameter on the controller once, guarded
   by a name check: a Float whose defaultFloat is the feature's
   declared default.
5. After every sub-tree is added, set each root child's
   directBlendParameter to the driver parameter. ChildMotion is a
   struct: read root.children into an array, set the field on each
   element, and assign the array back.

The same construction as code; the toggle sub-tree's thresholds are
the explicit form from step 3:

```csharp
using System.Linq;
using UnityEditor.Animations;
using UnityEngine;

if (!controller.parameters.Any(p => p.name == driverName))
    controller.AddParameter(new AnimatorControllerParameter
    {
        name = driverName,
        type = AnimatorControllerParameterType.Float,
        defaultFloat = 1f,
    });

var root = new BlendTree
{
    name = "Root",
    blendType = BlendTreeType.Direct,
    blendParameter = driverName,
    useAutomaticThresholds = false,
};
var state = new AnimatorState
{
    name = "Root",
    motion = root,
    writeDefaultValues = true,
};
var stateMachine = new AnimatorStateMachine { name = "Root" };
stateMachine.AddState(state, Vector3.zero);
stateMachine.defaultState = state;
controller.AddLayer(new AnimatorControllerLayer
{
    name = layerName,
    stateMachine = stateMachine,
    blendingMode = AnimatorLayerBlendingMode.Override,
    defaultWeight = 1f,
});

var toggle = new BlendTree
{
    name = featureName,
    blendType = BlendTreeType.Simple1D,
    blendParameter = featureParameter,
    useAutomaticThresholds = false,
};
toggle.AddChild(defaultClip, 0f);
toggle.AddChild(changedClip, 1f);
root.AddChild(toggle);
if (!controller.parameters.Any(p => p.name == featureParameter))
    controller.AddParameter(new AnimatorControllerParameter
    {
        name = featureParameter,
        type = AnimatorControllerParameterType.Float,
        defaultFloat = featureDefault,
    });

var children = root.children;
for (int i = 0; i < children.Length; i++)
    children[i].directBlendParameter = driverName;
root.children = children;
```

- Blend trees blend on floats, so every parameter in the layer is
  registered on the controller as a Float with its default as the
  default value — two-state parameters included (0/1). A name shared
  with the Expression Parameters is declared Float there as well.
- Because the state runs with Write Defaults on, every property a
  sub-tree drives must be present in both of its clips: the
  default-state clip carries the resting values, and a property
  animated by only one side falls back to its scene default whenever
  its weight is 0.
- Cost: the whole product is one layer regardless of feature count,
  and every driven property is evaluated every frame.

## Mapping component fields onto an SDK behaviour

- Convert at generation time, not in the component: clamp ranges to the
  behaviour's documented limits (and swap min/max after clamping so x ≤ y),
  filter null clips, turn an object reference into the path the behaviour
  expects (a play-audio behaviour, for example, takes the
  AudioSource's transform path relative to the avatar root, empty
  string for the root itself).
- Compute relative paths by walking parents to the avatar root; validate
  first that the target is the root or a descendant, so the walk is
  guaranteed to terminate at the root.
- The path basis must match the merge: when the controller is merged
  with a relative path mode, a behaviour's path field (a Play Audio
  Source Path, for example) is interpreted relative to the merge root
  and rewritten on merge, and nothing auto-corrects a path written
  against a different root. Compute behaviour paths against the same
  root the merge will use.
- SDK behaviour field and enum names must be verified against code that
  compiles against the SDK (NEVER #8 in SKILL.md).

## Merging into the avatar

The route depends on the dependency case (SKILL.md, Design
Decisions).

### Case 2: emit a merge component

- Add one ModularAvatarMergeAnimator to the avatar root with the generated
  controller: animator = the controller, layerType = FX (or the layer the
  product targets), matchAvatarWriteDefaults = true when the generated
  states assume Modular Avatar's write-defaults handling.
- Modular Avatar consumes merge components in its own passes, which is why
  the generating pass is ordered before it (Framework reference).
- Remove the source components only after the merge component exists,
  and remove every found instance, including ones whose generation
  failed (P-19).

### Case 1: edit the avatar's controllers through virtual controllers

NDMF's AnimatorServicesContext exposes the avatar's own controllers
as virtual controllers: editable representations that NDMF commits
back into build-owned copies when the context deactivates. The
plugin never clones a controller asset and never reassigns the
descriptor itself. The context is available since NDMF 1.7.0
(Framework, Version floors).

1. Declare the dependency on the pass,
   [DependsOnContext(typeof(AnimatorServicesContext))], and read the
   context in Execute: ctx.Extension<AnimatorServicesContext>().
2. Take the target controller from
   asc.ControllerContext.Controllers. On a VRChat avatar the
   playable layers are keyed by VRCAvatarDescriptor.AnimLayerType
   (FX, Gesture, and the rest); a scene Animator's controller is
   keyed by the Animator itself.
3. Edit the virtual controller. SetParameter registers a parameter.
   AddLayer(LayerPriority, name) adds a layer; the priority orders
   it against layers other passes add, without index arithmetic.
   Build states on the layer's StateMachine (AddState), and
   virtualize every real Motion with
   asc.ControllerContext.CloneContext.Clone(motion) before assigning
   it.
4. Register every newly created object that animations will
   reference with asc.ObjectPathRemapper. While the context is
   active, NDMF tracks avatar object renames and rewrites animation
   paths accordingly; the remapper is how new objects join that
   tracking.

```csharp
using nadena.dev.ndmf;
using nadena.dev.ndmf.animator;
using UnityEditor.Animations;
using UnityEngine;
using VRC.SDK3.Avatars.Components;

[DependsOnContext(typeof(AnimatorServicesContext))]
class AddFxLayerPass : Pass<AddFxLayerPass>
{
    protected override void Execute(BuildContext ctx)
    {
        var asc = ctx.Extension<AnimatorServicesContext>();
        var fx = asc.ControllerContext.Controllers[
            VRCAvatarDescriptor.AnimLayerType.FX];
        fx.SetParameter("MyToggle", new AnimatorControllerParameter
        {
            name = "MyToggle",
            type = AnimatorControllerParameterType.Bool,
            defaultBool = false,
        });
        var layer = fx.AddLayer(LayerPriority.Default, "MyPlugin");
        layer.DefaultWeight = 1;
        var state = layer.StateMachine.AddState("Active",
            asc.ControllerContext.CloneContext.Clone(activeClip));
        state.WriteDefaultValues = false;
        layer.StateMachine.DefaultState = state;
    }
}
```

A plugin that instead replaces a whole playable-layer controller
clones the avatar's controller, edits the clone, and assigns the
clone to the descriptor's layer; the shared asset is never mutated.
That route is the fallback for edits the virtual objects do not
expose.

## Attaching SDK state behaviours

- Attach a behaviour by adding the SDK type to the state and setting
  its fields. The catalogue of behaviours and their platform
  semantics live in VRChat Avatar Platform (State Behaviour
  Semantics).
- A generator sets every field it relies on: defaults differ
  depending on how the behaviour instance was created, and an unset
  field (an apply mode, for example) silently changes the outcome.
- One Parameter Driver per state collects all of that state's
  parameter changes; do not stack duplicates. Its sync behaviour is
  part of the design, not a detail: changes can be restricted to the
  local player, and random changes can be marked unsynced.
- Multiple Play Audio instances may coexist on one state, one per
  AudioSource. For Tracking Control, set per element; unset elements
  keep the SDK default.
- VRChat's built-in animator parameters are ordinary parameter names
  from the generator's point of view: treat the names as constants
  defined in one place in the product, verified against the SDK
  documentation, and validate them through ParameterInfo like any
  other name (Parameters).

## Output routes, including the destructive one

| Route | Dependency case | When it fits | Cost |
| --- | --- | --- | --- |
| New controller + MergeAnimator | Case 2 (NDMF + MA) | Plugin products distributed to other authors | Output is additive and removable; the pass must run before MA |
| Plugin-managed controller | Case 1 (NDMF only) | The plugin must add animator output without MA | The plugin clones the target controller inside the pass and owns all bookkeeping; never mutate the shared asset |
| Destructive, into the descriptor's controllers | Neither (editor tooling, outside a plugin build) | Tooling the avatar's own author runs deliberately on their own project, especially over hand-authored controllers | Mutates the author's assets; needs its own undo/backup story |

The destructive route, as a pattern: the generator binds itself to the
avatar descriptor, writes one principal layer per playable layer type
(FX, Gesture, Action, Additive as idle, Base as locomotion) inside the
descriptor's own controllers, and puts auxiliary layers under a name
suffix the generator owns. Regeneration is delete-then-recreate: remove
the principal layers and every layer carrying the owned suffix, then
write them again. It is repeatable only because the suffix namespace is
owned and nothing else writes into it. Never run this inside an NDMF
build pass (SKILL.md NEVER #9): the build operates on a clone precisely
so the author's assets survive it.
