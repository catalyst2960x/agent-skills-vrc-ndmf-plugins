# Template: Non-destructive parameter assignment

## When it applies

The author wants a plugin to name the parameter of a component it
does not own (a PhysBone, a Contact Receiver, a Raycast) — for
example from a suggestion list — without the plugin ever writing
to the author's components in the editor or in their assets. The
component records the intent as data (target component + name);
a build pass applies it to the build copy only.

## Requirements

- The assignment is data on the plugin's own component:
  `{ Component target; string Name; }`. The editor records it and
  never writes the target's field.
- A Resolving-phase pass applies the assignments, so later phases
  see the names as if the author had entered them.
- An assignment applies only when all of these hold; each failure
  is a NonFatal report and a skip, never a silent write:
  - the target is inside the avatar;
  - no earlier assignment in the same build claimed the target
    (first assignment wins; a conflicting second one is reported);
  - the name does not collide with the generated names of any
    other generating component or assignment (the ledger below);
  - the target's parameter field is still empty. A field the
    author filled is never overwritten.
- The collision ledger is the set of names every generating
  component will produce from its base name (see
  generated-parameter-entries), collected over the whole avatar
  plus the other assignments. One exception: a name shared by
  Contact Receivers of the same generated type is not a conflict,
  because the platform merges same-name, same-type receivers.

## Skeleton

```csharp
// [Runtime component] Data on the plugin's own component:
[Serializable]
public class ParameterAssignment
{
    public Component Component;
    public string Name;
}

// [Editor: NDMF pass] Resolving phase:
void ApplyAssignments(BuildContext ctx)
{
    var found = ctx.AvatarRootObject.GetComponentsInChildren<NDMFComponent>(true);
    if (found.Length == 0) return;

    var root = ctx.AvatarRootObject.transform;
    var applied = new Dictionary<Component, string>();
    var taken = ParameterNameLedger.TakenNames(ctx.AvatarRootObject, null, includeAssignments: false);

    foreach (var component in found)
    {
        foreach (var assignment in component.ParameterAssignments ?? new ParameterAssignment[0])
        {
            if (assignment == null || assignment.Component == null || string.IsNullOrEmpty(assignment.Name))
                continue;
            var target = assignment.Component;
            if (target.transform != root && !target.transform.IsChildOf(root))
            {
                Warn(component, "A parameter assignment target is outside the avatar: " + target.gameObject.name);
                continue;
            }
            if (applied.TryGetValue(target, out var existing))
            {
                if (existing != assignment.Name)
                    Warn(component, "Conflicting parameter assignments for " + target.gameObject.name + "; the first one applies");
                continue;
            }
            var conflict = ParameterNameLedger.FindConflict(target, assignment.Name, taken);
            if (conflict != null)
            {
                Warn(component, "Parameter name already in use: " + conflict + "; the assignment for " + target.gameObject.name + " is skipped");
                continue;
            }
            if (ApplyAssignment(target, assignment.Name))
            {
                applied[target] = assignment.Name;
                ParameterNameLedger.AddEntries(taken, target, assignment.Name);
            }
        }
    }
}

// Applies only into a field that is still empty:
static bool ApplyAssignment(Component target, string name)
{
    switch (target)
    {
        case VRCPhysBone bone:
            if (!string.IsNullOrEmpty(bone.parameter)) return false;
            bone.parameter = name;
            return true;
        case VRCContactReceiver receiver:
            if (!string.IsNullOrEmpty(receiver.parameter)) return false;
            receiver.parameter = name;
            return true;
        case VRCRaycast raycast:
            if (!string.IsNullOrEmpty(raycast.Parameter)) return false;
            raycast.Parameter = name;
            return true;
        default:
            return false;
    }
}
```
