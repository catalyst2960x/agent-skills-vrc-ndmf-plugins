# Template: Generated parameter entries from a base name

## When it applies

Several SDK components generate their animator parameters from
one base name the author enters. Anything that reasons about
names — suggestions, collision checks, type resolution — must
expand the base name into the full set of generated entries, with
types, from the component kind. One case is prefix-shaped: a
VRCRaycast's `Parameter` field is not a parameter name but a
prefix, and the component generates three parameters from it.

## Requirements

- The expansion is a pure function of (component kind, base
  name); an empty base name generates nothing.
- The table is the platform's generation rules, kept in one
  place:
  - PhysBone: `{p}_IsGrabbed` (Bool), `{p}_IsPosed` (Bool),
    `{p}_Angle` (Float), `{p}_Stretch` (Float), `{p}_Squish`
    (Float).
  - Contact Receiver: `{p}` itself — Float for a Proximity
    receiver, Bool otherwise.
  - VRCRaycast: `{p}_Hit` (Bool), `{p}_Ratio` (Float),
    `{p}_Distance` (Float).
- VRCRaycast exists only in SDK 3.10.3 and later; the entries and
  every use sit behind a version define on the SDK package.

## Skeleton

```csharp
// [Editor: NDMF pass]
public static List<(string Name, AnimatorControllerParameterType Type)> GeneratedEntries(Component component, string baseName)
{
    var result = new List<(string Name, AnimatorControllerParameterType Type)>();
    if (component == null || string.IsNullOrEmpty(baseName)) return result;
    if (component is VRCPhysBone)
    {
        result.Add((baseName + "_IsGrabbed", AnimatorControllerParameterType.Bool));
        result.Add((baseName + "_IsPosed", AnimatorControllerParameterType.Bool));
        result.Add((baseName + "_Angle", AnimatorControllerParameterType.Float));
        result.Add((baseName + "_Stretch", AnimatorControllerParameterType.Float));
        result.Add((baseName + "_Squish", AnimatorControllerParameterType.Float));
    }
    else if (component is VRCContactReceiver receiver)
    {
        result.Add((baseName, IsProximity(receiver)
            ? AnimatorControllerParameterType.Float
            : AnimatorControllerParameterType.Bool));
    }
#if VRCSDK_3_10_3_OR_NEWER
    else if (component is VRCRaycast)
    {
        result.Add((baseName + "_Hit", AnimatorControllerParameterType.Bool));
        result.Add((baseName + "_Ratio", AnimatorControllerParameterType.Float));
        result.Add((baseName + "_Distance", AnimatorControllerParameterType.Float));
    }
#endif
    return result;
}
```
