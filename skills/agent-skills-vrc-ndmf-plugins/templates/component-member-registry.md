# Template: A registry of animatable component members

## When it applies

A generator lets the author pick properties of arbitrary
components on target objects and animate them — a menu generator
driving component settings, for example. Without a
registry, each component type's animatable properties are handled
ad hoc: the editor reads a value through the C# member, the curve
binding needs the serialized name, and the two drift apart; a type
with no hand-written handling silently offers nothing, not even
its enable toggle. The registry is the single place where the
naming systems meet, so a member is described once and the
editor, the value reader, and the curve writer all use the same
description.

## Requirements

- An entry binds four things: a reflection handle (the field or
  property), used to read the current value and to build the
  editor control; the animation binding name, the serialized name
  the animation system uses, which often differs from the C# name;
  an optional UI label, for a member whose meaning differs from
  its name; and an optional nested path, for a member inside a
  sub-structure of the component.
- A component type whose class has an `enabled` property gains an
  `enabled` member automatically, ahead of its registry entries,
  binding to the serialized enabled field. A type with no registry
  entry but with `enabled` still exposes exactly one item; a type
  with neither exposes nothing.
- The component types offered for a target are the types actually
  on the target object, filtered through the VRChat SDK's avatar
  component whitelist, so the UI never offers a component VRChat
  would strip from an avatar.
- Members whose type is float, Vector3, Quaternion, or Color are
  suitable for continuous transitions; all other member types are
  discrete-only.
- A member inside a repeated sub-structure is registered once per
  possible index: the reflection handle stays the leaf field's,
  and the nested path carries the position (the path walks the
  serialized list to that element's field). The label names the
  index.
- Which members a registry exposes per component type is the
  adopting product's data, not part of this template.

## Skeleton

```csharp
// [Editor]
// The registry entry and the two completion rules, in minimal form.
// This block demonstrates several features: entry shape, the
// automatic enabled member, and whitelist filtering of candidates.
public sealed class MemberEntry
{
    public MemberInfo Handle;      // reflection: read values, build controls
    public string BindingName;     // serialized name used in curve bindings
    public string Label;           // optional UI label (null = member name)
    public string NestedPath;      // optional path inside a sub-structure
}

public static IReadOnlyList<MemberEntry> MembersOf(Type componentType)
{
    var members = new List<MemberEntry>();
    if (HasEnabledProperty(componentType))
        members.Add(new MemberEntry { BindingName = "m_Enabled" });
    if (Registry.TryGetValue(componentType, out var registered))
        members.AddRange(registered);
    return members;
}

public static IEnumerable<Type> OfferedComponentTypes(GameObject target) =>
    target.GetComponents<Component>()
        .Select(c => c.GetType())
        .Where(t => AvatarWhitelist.Contains(t));
```

```csharp
// [Editor]
// Registering one member per element of a repeated sub-structure.
IEnumerable<MemberEntry> SourceWeights(Type constraintType) =>
    Enumerable.Range(0, MaxSourceCount).Select(i => new MemberEntry
    {
        Handle = SourceWeightField,
        BindingName = SourceWeightField.Name,
        NestedPath = $"Sources.source{i}.Weight",
        Label = $"Sources[{i}].Weight",
    });
```
