# Template: A Toggle layer with timed transition clips

## When it applies

A Bool-driven menu whose change should take time — objects flip
in sequence, values ramp — rather than switching instantly on the
parameter edge. Without transition states the change is a hard
cut. Without the empty-transition handling, a transition in which
no item animates collapses to zero duration and the sequencing
the author configured disappears.

## Requirements

- Four clips: active, inactive, and the two transition clips
  (to-active, to-inactive). Curve construction inside the
  transition clips is the value families'
  (value-curve-families.md); each item also carries use flags
  deciding which of the four clips receive its curve.
- Three standing states — idle, active, inactive — where idle
  carries the inactive clip; instant condition edges on the Bool
  (If to active, IfNot to inactive). All states have Write
  Defaults off.
- With a transition duration, two intermediate states play the
  transition clips and leave by exit time. An object item's
  offset percent positions its flip inside the clip, so several
  objects can flip in sequence inside one transition.
- A transition clip that ends up with no bindings at all either
  drops its states (when the menu opts to skip empty transitions)
  or receives a dummy curve on a sentinel path, so the state
  still lasts the full duration.

## Skeleton

```csharp
// [Editor: generator]
// The Toggle layer skeleton: three standing states, two optional
// transition states, all Write Defaults off, instant condition edges.
AnimatorController BuildToggleController(string name, string parameter, Clips clips, float seconds)
{
    var controller = new AnimatorController();
    controller.AddParameter(parameter, AnimatorControllerParameterType.Bool);
    controller.AddLayer(name);
    var sm = controller.layers[0].stateMachine;

    var idle = sm.AddState(name + "_idle");       idle.motion = clips.Inactive;
    var active = sm.AddState(name + "_active");   active.motion = clips.Active;
    var inactive = sm.AddState(name + "_inactive"); inactive.motion = clips.Inactive;
    foreach (var s in sm.states) s.state.writeDefaultValues = false;
    sm.defaultState = idle;

    Edge(idle, active, If(parameter));
    Edge(idle, inactive, IfNot(parameter));
    if (seconds > 0)
    {
        var go = sm.AddState(name + "_activate");     go.motion = clips.Activate;
        var back = sm.AddState(name + "_inactivate"); back.motion = clips.Inactivate;
        go.writeDefaultValues = back.writeDefaultValues = false;
        Edge(inactive, go, If(parameter));   ExitByTime(go, active);
        Edge(active, back, IfNot(parameter)); ExitByTime(back, inactive);
    }
    else
    {
        Edge(inactive, active, If(parameter));
        Edge(active, inactive, IfNot(parameter));
    }
    return controller;
}
```

```csharp
// [Editor: generator]
// The dummy curve that keeps an otherwise empty transition state
// at its full duration: a constant curve on a path nothing uses.
void KeepDuration(AnimationClip transitionClip, float seconds)
{
    if (AnimationUtility.GetCurveBindings(transitionClip).Length > 0) return;
    transitionClip.SetCurve("__transition_dummy__", typeof(Transform), "m_LocalPosition.x",
        new AnimationCurve(new Keyframe(0f, 0f), new Keyframe(seconds, 0f)));
}
```
