# Animator Generation Reference

How a pass turns component configuration into an AnimatorController.
Construction uses the raw UnityEditor.Animations APIs
(AnimatorController, AnimatorStateMachine, AnimatorStateTransition)
directly. For larger or more repetitive state graphs, wrap those calls in
the product's own internal helper layer rather than taking a construction
library as a dependency: neither dependency case includes one. Keep the
bookkeeping (asset containers, state identity) in that one layer.

## The layer/state pattern

Choose the shape per feature before building, and weigh both costs.
A feature that needs transitions or behaviour on state entry takes a
state-machine layer from this section. A value-driven setting — a
toggle, a selector, a graded value — that needs neither can go in
the Direct Blend Tree shape (Direct Blend Tree layers): the product
stays at one layer however many such features it has, at the price
of evaluating every driven property every frame, so the tree's cost
grows with the feature count. State-machine layers cost a layer per
feature instead, and a layer resting in its idle state evaluates
almost nothing.

- One AnimatorController per pass. The pattern's default is one
  layer per component or setting; a component whose design separates
  concerns — different blending or weight, a behaviour to isolate —
  may build several layers. Layer name identifies the source (for
  example
  `<product name> [<GameObject name>]`); set the layer's defaultWeight
  to 1 or the layer silently contributes nothing. Set it where the
  layer is constructed, or write the layer back: a layer taken from
  a controller's layers array is a copy (the element is a struct),
  so assigning defaultWeight on the retrieved value changes nothing
  in the controller — assign the modified layer back into the
  array. A consumer that merges controllers normalizes the first
  layer's weight to 1, which masks the omission in a one-layer
  controller; a controller with several layers is where it
  surfaces. Names within one
  generated controller are distinct: when two components would yield
  the same name, the generator makes them distinct by its own
  deterministic rule, assigned in collection order so repeated
  builds produce the same names.
- Generated clips are named by their role within their container.
  The layer and state structure already identifies the feature and
  the object, so the clip's name carries the role: a toggle's two
  motions by their roles, a clip with no curves as Empty. A clip
  left unnamed keeps Unity's default name, and every generated
  clip in the built controller then reads the same. When the same
  role repeats inside one container, append a discriminator — an
  index, or the name of the parameter that drives it. When clips
  for many objects share one container with no per-object
  structure — a shared layer, a blend tree — the name carries the
  object as well, because the container no longer supplies it.
- States: `idle` (the default state), `active`, and optionally an
  intermediate state when a two-stage condition is wanted: the
  intermediate state is entered on a preliminary condition set and
  latches — it is held while the main condition set is awaited, with
  no return to idle. State names are the product's; the roles are
  what the pattern fixes. Generated states set
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
  With an intermediate stage, idle → intermediate carries the preliminary
  conditions and intermediate → active carries the main conditions. The
  reference pattern creates no intermediate → idle return; once the
  preliminary conditions have fired,
  the layer waits in the intermediate state for the main conditions.
- Return to idle, two options:
  - **Reversed conditions**: one transition per condition, each carrying
    that condition reversed. Separate transitions make the return an OR of
    the negations, which is the correct negation of the forward AND (for
    Float comparisons, exact except at the threshold point itself; see the
    note under the reversal table). A
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
  thresholds are unchanged, and for Float the reversal is not exact:
  a Float parameter supports only the Greater and Less modes, so no
  condition expresses value ≤ t, and at value == threshold neither
  the forward nor the return transition fires — the layer holds its
  current state. The Int shift has no Float analogue because the
  domain has no smallest step. The case that bites is a threshold at
  a value the parameter rests at (0 for Voice or a PhysBone's
  Stretch, for example): once the layer has entered active, returning
  to exactly that value does not return it. A design that must reset
  at the threshold uses a Bool-typed condition instead (If and IfNot
  are exact), or the free return. The parameter type comes from ParameterInfo,
  not from the condition itself — for a built-in parameter, from the
  platform catalogue (Built-in parameters in generated output).
- Reversal and thresholds are computed in the parameter's source type.
  If the generated controller registers a different type, VRChat
  converts the value before the conditions evaluate it (VRChat Avatar
  Platform, Parameter Type Conversion) and the arithmetic above no
  longer describes the behaviour: a Float parameter read as Int has
  already rounded, so a threshold at 0.5 is evaluated against 0 or 1.
  Generated output registers the source type (Parameters).
- Only parameters that exist in the build-time parameter set may be added
  to the generated controller as AnimatorControllerParameters, each once
  (see Parameters). Built-in parameters are the exception: register them
  with their catalogue type (Built-in parameters in generated output).
- An empty condition set is a real configuration, not an error: the
  forward transition carries no conditions and fires unconditionally, and
  the reversed set is empty, so no return transitions are generated and
  the layer stays active. Decide whether that is the product's intent
  (for an always-on effect it is); if a return is required, the design
  needs conditions or the free return.

## Built-in parameters in generated output

VRChat's built-in parameters (VRChat Avatar Platform, Built-in
Animator Parameters) can carry generated conditions, and they differ
from declared parameters in three ways:

- They are not in the build-time parameter set. No parameter
  provider supplies them, so ParameterInfo never returns them, and
  their types cannot come from it either. Take a built-in's type
  from the platform catalogue — condition reversal needs it. The
  name itself needs no validation; names are accepted as entered.
- Their names and types are fixed by the platform. Register a used
  built-in on the generated controller with the catalogue's type,
  once, like any used parameter. Never declare a built-in as the
  plugin's own parameter, in Expression Parameters or through a
  parameter provider, and never rename one. Modular Avatar's
  parameter handling rests on the same basis: its parameter probing
  skips the SDK-predefined names and does not offer them as
  remappable, and that set includes IsAnimatorEnabled.
- IsAnimatorEnabled carries a timing property the others do not:
  it goes false one frame before the avatar's animator is disabled
  on a client, and while the animator is disabled every layer
  freezes in its current state. A layer whose active state must
  not persist frozen — one holding a one-shot behaviour or
  driver-written values, for example — can add one more return
  transition carrying only IsAnimatorEnabled IfNot, next to its
  per-condition reversed transitions, so the layer settles to idle
  in that final frame. The disable it warns of is a viewing client
  culling a remote avatar, distance-hidden being the standing case:
  the wearer's own client does not observe it, the parameter is not
  synced, and its value differs per client, so the response has to
  be built into the layer itself. The critical cases are the ones
  whose frozen residue persists or whose completion never runs: a
  state holding Parameter Driver-written values (they stay written
  while the animator is off), a state whose cleanup runs on exit
  (the exit never runs once disabled), and a state entered for a
  momentary effect (it persists as if permanent). A toggle whose
  active state is a stable configuration needs none of this.
  Whether a layer needs this is a per-layer design decision.

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

- Blend trees blend on floats, so every parameter the layer blends on
  is registered on the controller as a Float with its default as the
  default value — two-state parameters included (0/1). That
  controller-side Float is the standing exception to registering the
  source type (P-08): VRChat converts the value per controller, and a
  Bool or Int source converts to exactly 0.0 or 1.0 (VRChat Avatar
  Platform, Parameter Type Conversion), so the thresholds describe
  the behaviour. The Expression Parameters declaration is not
  changed: it carries the parameter's own type, and a two-state
  parameter stays Bool — declared as Float it would cost 8 sync bits
  instead of 1.
- Because the state runs with Write Defaults on, every property a
  sub-tree drives must be present in both of its clips: the
  default-state clip carries the resting values, and a property
  animated by only one side falls back to its scene default whenever
  its weight is 0.
- Cost: the whole product is one layer regardless of feature count,
  and every driven property is evaluated every frame.
- Merging (dependency case 2): emit the merge component with
  matchAvatarWriteDefaults = false for this layer. Matching rewrites
  state Write Defaults to the target controller's convention and
  would turn this state's on setting off (Merging into the avatar).

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
  with a relative path mode, a behaviour's path field
  is interpreted relative to the merge root
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
  product targets), matchAvatarWriteDefaults per the generated output's
  shape. Matching rewrites each merged state's Write Defaults to the
  target controller's convention: set it true for the layer/state
  pattern, whose states are built to that convention, and false for a
  Direct Blend Tree layer, whose state must keep Write Defaults on
  (Direct Blend Tree layers) — matching would rewrite it off.
- Modular Avatar consumes merge components in its own passes, which is why
  the generating pass is ordered before it (Framework reference).
- The merge carries a controller's curves for the parameters that
  controller declares; a curve for a parameter the controller does
  not declare is dropped in the merge. Writing curves into a
  controller another system owns therefore fails mechanically as
  well as by rule (NEVER #9) when the parameter is not declared
  there.
- Remove the source components only after the merge component exists,
  and remove every found instance, including ones whose generation
  failed (P-21).

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
3. Edit the virtual controller. Register a parameter by handing a
   fully constructed AnimatorControllerParameter to the controller:
   `fx.Parameters = fx.Parameters.SetItem(name, parameter)`, or
   `fx.SetParameter(name, parameter)`, which performs the same
   assignment. Since NDMF 1.9.0 the framework clones a parameter
   when it is set, and changing a parameter retrieved from the map
   in place is detected and will become an error, so every field —
   type and default included — is set on the new object before it
   is handed over. AddLayer(LayerPriority, name) adds a layer; the priority orders
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
        fx.Parameters = fx.Parameters.SetItem("MyToggle",
            new AnimatorControllerParameter
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

- Attach a behaviour by adding the SDK type to the state
  (AnimatorState.AddStateMachineBehaviour) or with
  ScriptableObject.CreateInstance, and set its fields on the
  attached instance. A StateMachineBehaviour is a ScriptableObject:
  an instance created with `new` is not engine-managed and does not
  survive into the built avatar. The catalogue of behaviours and
  their platform semantics live in VRChat Avatar Platform (State
  Behaviour Semantics).
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
  documentation. Their validation and registration differ from
  declared parameters (Built-in parameters in generated output).

## Output routes, including the destructive one

| Route | Dependency case | When it fits | Cost |
| --- | --- | --- | --- |
| New controller + MergeAnimator | Case 2 (NDMF + MA) | Plugin products distributed to other authors | Output is additive and removable; the pass must run before MA |
| Virtual controllers (AnimatorServicesContext) | Case 1 (NDMF only) | The plugin must add animator output without MA | Edits commit to build-owned copies when the context deactivates; the plugin clones no controller asset and owns no controller bookkeeping |
| Clone and assign | Case 1 (fallback) | The edit replaces a whole playable-layer controller, or the virtual objects do not expose the edit | The plugin owns the cloned controller's bookkeeping; never mutate the shared asset |
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
