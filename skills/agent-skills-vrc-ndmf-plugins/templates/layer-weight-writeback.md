# Template: Layer weight write-back

## When it applies

A pass adds a layer to a generated `AnimatorController` and sets
its weight to 1. `AnimatorControllerLayer` is a struct: reading
`controller.layers[i]` yields a copy, so assigning `defaultWeight`
to the retrieved value changes nothing in the controller, and the
layer enters the merge with weight 0. The defect is masked in a
one-layer controller because the merge side forces the first
layer's weight to 1; it surfaces as soon as a controller has
several layers.

## Requirements

- After setting any field on a retrieved layer, assign the whole
  layers array back to the controller.
- Apply this to every retrieved-layer edit, not only weights: the
  same copy semantics apply to the state machine reference and
  every other field.

## Skeleton

```csharp
// [Editor: generator]
controller.AddLayer(layerName);
var layers = controller.layers;
layers[layers.Length - 1].defaultWeight = 1f;
controller.layers = layers; // the write-back is the fix
var machine = layers[layers.Length - 1].stateMachine;
```
