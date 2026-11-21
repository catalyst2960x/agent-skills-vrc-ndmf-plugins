# Template: Gating a driven layer by its weight

## When it applies

A generated layer drives a value proportionally — a 1D Blend
Tree over a Float parameter — and the drive must contribute
nothing while a condition set does not hold, with no partial
contribution in between. The gate is the driven layer's weight,
set by Animator Layer Control behaviours on the states of a
second generated layer that evaluates the conditions. No
parameter is spent on the gate: the gate's state is the layer
weight itself, and every client evaluates the same conditions
from the same input parameters, so the weight agrees wherever
the inputs agree.

## Requirements

- The driven layer's default weight in the generated controller
  is 0, and the gate is the only writer of its weight.
- The driven layer is not the first layer of its own generated
  controller: a controller's first layer always has weight 1
  regardless of what is serialized, and when Modular Avatar
  merges the controller it writes that 1 into the layer —
  transplanted out of the first position, the serialized 0
  would otherwise take effect — which would hold the gate
  open. In this template's shape the gating layer is
  generated first, so the driven layer never stands first.
- Every state of the gating layer carries the control: the
  idle-side states with goal weight 0, the active state with
  goal weight 1. A weight a behaviour sets persists after the
  state exits, so a state without the control leaves the
  previous weight in place.
- The control's layer index is the driven layer's index in the
  generated controller; the framework rewrites it when the
  merged controller is committed (Animator Generation,
  Attaching SDK state behaviours). The control's playable is
  the playable the driven layer belongs to.
- The blend duration is 0: the gate switches. A fade is a
  different design, in which the duration is part of the
  product's timing.

## Skeleton

```csharp
// [Editor: generator]
// The driven layer starts at weight 0; only the gate raises it.
var driven = new AnimatorControllerLayer
{
    name = drivenLayerName,
    defaultWeight = 0f,
    stateMachine = drivenMachine,
};
var layers = controller.layers;
ArrayUtility.Add(ref layers, driven);
controller.layers = layers;
var drivenIndex = layers.Length - 1;

// Every state of the gating layer carries the control, so the
// weight is rewritten on whichever state the layer occupies.
void AddGate(AnimatorState state, float goalWeight)
{
    var control =
        state.AddStateMachineBehaviour<VRCAnimatorLayerControl>();
    control.playable = drivenPlayable;
    control.layer = drivenIndex;
    control.goalWeight = goalWeight;
    control.blendDuration = 0f;
}

AddGate(idleState, 0f);
AddGate(preActiveState, 0f);
AddGate(activeState, 1f);
```
