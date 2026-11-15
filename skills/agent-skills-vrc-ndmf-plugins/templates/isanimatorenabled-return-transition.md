# Template: IsAnimatorEnabled return transition

## When it applies

A generated layer's active state must not stay frozen when the
avatar's animator is disabled (a viewing client culling a remote
avatar). The platform parameter `IsAnimatorEnabled` turns false
one frame before the animator is disabled, which is the layer's
last chance to settle: a dedicated return transition carrying
only that condition moves the layer back to idle on the final
frame, instead of leaving it parked in active to resume stale on
re-enable.

## Requirements

- The settle transition is separate from the per-condition return
  transitions: it carries only `IsAnimatorEnabled` IfNot, with no
  exit time and zero duration, like every other generated
  transition in the layer.
- The parameter is declared in the generated controller (Bool)
  even though the platform supplies its value.
- Whether a layer needs the settle transition is a property of
  the layer's effect: a layer whose active state is harmless to
  resume (a static toggle) may omit it; a layer whose state
  drives an ongoing effect (playback) takes it.

## Skeleton

```csharp
// [Editor: generator]
// The ordinary returns: one per chain proxy, Less 0.5.
foreach (var pair in proxies)
{
    var back = active.AddTransition(idle);
    back.hasExitTime = false;
    back.hasFixedDuration = true;
    back.duration = 0f;
    back.exitTime = 0f;
    back.AddCondition(AnimatorConditionMode.Less, 0.5f, pair.Key);
}

// The settle return: animator about to be disabled.
var settle = active.AddTransition(idle);
settle.hasExitTime = false;
settle.hasFixedDuration = true;
settle.duration = 0f;
settle.exitTime = 0f;
settle.AddCondition(AnimatorConditionMode.IfNot, 0f, "IsAnimatorEnabled");
```
