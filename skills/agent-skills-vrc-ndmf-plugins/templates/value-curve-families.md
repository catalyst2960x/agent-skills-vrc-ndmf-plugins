# Template: Curve construction in two families, per value type

## When it applies

A generator writes many value types into transition clips — the
clips that play while a menu state changes. Without the family
split, curve construction is rewritten once per item kind, and
the return direction's timing is re-derived (and can be derived
differently) for every type. The methodology is: a value knows
its two endpoints and its timing; the clip set asks the value for
the curve of each direction, so construction is written once per
value type, not once per item kind.

## Requirements

- A value carries an active endpoint, an inactive endpoint, an
  offset rate, and a duration rate (rates are fractions of the
  transition duration).
- Two families:
  - Discrete: the value steps at the offset within the
    transition; the return direction mirrors the offset.
  - Continuous: the value ramps between the offset rate and the
    offset-plus-duration rate; the return direction mirrors both
    rates.
- Which family applies follows from the value type: interpolable
  types (float, Vector3, Quaternion, Color) can ramp; the rest
  step.
- Multi-component values write one curve per component, each from
  the same family construction applied to that component.

## Skeleton

```csharp
// [Editor: generator]
// One value type's curve construction, shown for a float value.
// Discrete values use StepCurve with the same signature.
public AnimationCurve RampCurve(bool toActive, float seconds, ToggleValue value)
{
    var startRate = toActive ? value.OffsetRate : 1f - value.OffsetRate - value.DurationRate;
    var endRate = startRate + value.DurationRate;
    var from = toActive ? value.Inactive : value.Active;
    var to = toActive ? value.Active : value.Inactive;
    return new AnimationCurve(
        new Keyframe(seconds * startRate, from),
        new Keyframe(seconds * endRate, to));
}

public AnimationCurve StepCurve(bool toActive, float seconds, ToggleValue value)
{
    var at = seconds * (toActive ? value.OffsetRate : 1f - value.OffsetRate);
    var from = toActive ? value.Inactive : value.Active;
    var to = toActive ? value.Active : value.Inactive;
    return new AnimationCurve(
        new Keyframe(0f, from),
        new Keyframe(at, from),
        new Keyframe(at, to));
}
```

```csharp
// [Editor: generator]
// The clip set asks each item's value for its curves; the loop is
// the scaling side, over every item of the menu.
void WriteTransitionCurves(ClipSet clips, IEnumerable<Item> items, float seconds)
{
    foreach (var item in items)
    {
        item.Value.WriteCurves(clips.ToActive, toActive: true, seconds, item);
        item.Value.WriteCurves(clips.ToInactive, toActive: false, seconds, item);
    }
}
```
