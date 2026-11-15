# Template: A Choose layer driven by AnyState equality transitions

## When it applies

An Int parameter selects one of N options, and each option is a
full set of values across many items. Wiring per-option condition
edges between every pair of states grows with the square of the
option count and every new option rewires the whole layer.
AnyState transitions on parameter equality keep the wiring at one
transition per option, independent of which state is current.

## Requirements

- One clip per option, holding constant values at time zero for
  every item. An option an item does not specify falls back to
  zero for that item, or to a null material for a material item,
  so every clip writes every item.
- One state per option, plus an idle state carrying option 0's
  clip. All states have Write Defaults off.
- AnyState transitions on parameter equality, one per option,
  never to the state the machine is already in (no
  self-transition), with the menu's transition duration as the
  transition duration.
- The menu side is a submenu with one Toggle control per option,
  each setting the Int parameter to the option index.

## Skeleton

```csharp
// [Editor: generator]
// One clip and one state per option; the fill loop is the scaling
// side, writing every item into every option's clip.
AnimatorController BuildChooseController(string name, string parameter,
    IReadOnlyList<IReadOnlyList<ItemValue>> options, float seconds)
{
    var controller = new AnimatorController();
    controller.AddParameter(parameter, AnimatorControllerParameterType.Int);
    controller.AddLayer(name);
    var sm = controller.layers[0].stateMachine;

    var states = new List<AnimatorState>();
    for (var i = 0; i < options.Count; i++)
    {
        var clip = new AnimationClip { name = $"{name}_{i}" };
        foreach (var value in options[i]) value.WriteConstant(clip);
        var state = sm.AddState($"{name}_{i}");
        state.motion = clip;
        state.writeDefaultValues = false;
        states.Add(state);
    }
    sm.defaultState = states[0];

    for (var i = 0; i < states.Count; i++)
    {
        var edge = sm.AddAnyStateTransition(states[i]);
        edge.AddCondition(AnimatorConditionMode.Equals, i, parameter);
        edge.hasExitTime = false;
        edge.duration = seconds;
        edge.canTransitionToSelf = false;
    }
    return controller;
}
```
