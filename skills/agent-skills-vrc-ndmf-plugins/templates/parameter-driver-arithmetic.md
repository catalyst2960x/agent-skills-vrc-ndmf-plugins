# Template: Parameter Driver arithmetic

## When it applies

A parameter must be computed from a constant or from one other
parameter at a discrete moment — a menu selection, a state
entry, a reset — rather than continuously: assigning a mode,
stepping a counter, scaling a synced value into another
parameter's range, drawing a random value. The Avatar Parameter
Driver (a VRChat state behaviour) performs the computation on
state entry, so no blend trees are built for it. Continuous
tracking of a changing source, and any operation combining two
parameters, are outside its operations; those are the blend
tree route (the AAP templates).

## The operations

One driver holds a list of operations, executed in list order,
once, on entry to its state. `name` is the destination
parameter.

| Computation | Operation | Form |
| --- | --- | --- |
| Constant assignment | Set | dest := value |
| Add or subtract a constant | Add | dest := dest + value (a negative value subtracts) |
| Multiply or divide by a constant | Copy with Convert Range | dest := src × (dest range) / (source range); with both ranges starting at 0, dest := src × k, and k below 1 divides |
| Scale, then offset (affine) | Copy with Convert Range, then Add, in one driver | dest := src × k, then dest := dest + c |
| Copy, with type conversion | Copy | bool reads as 0 or 1; converting to bool, 0 is false and anything else is true; converting to int rounds down |
| Random | Random | Float and int: a value in [valueMin, valueMax]; bool: true with probability chance. An int Random offers a prevent-repeats setting |

## Rules

- Set, Add, Random, and Copy apply to float and int; bool
  takes Set, Random, and Copy (no Add).
- A destination that is a synced parameter (defined in
  Expression Parameters) is clamped when the driver writes it:
  float to [-1, 1], int to [0, 255]. A parameter defined only
  in the controller is not clamped, so an intermediate result
  that exceeds the synced range is held in a controller-only
  parameter and copied into the synced one in range.
- A VRChat-defined parameter cannot be a destination, and a
  built-in parameter does not work as a Copy source.
- Add and Random run on every instance's own entry and can
  diverge between instances. The documented form is a synced
  destination with the driver set to Local Only, so only the
  wearer's instance computes and the result reaches the others
  through sync.
- No operation combines two parameters: Copy overwrites its
  destination, and Add takes a constant. Parameter plus
  parameter, parameter times parameter, and parameter divided
  by parameter have no driver form.

## Skeleton

The driver construction, with one compute state per
computation. The state is entered when the value is needed and
exited at once; the driver has run on entry.

```csharp
// [Editor: generator]
// The operation constructors. name is the destination.
static VRC_AvatarParameterDriver.Parameter Set(string name, float value) =>
    new VRC_AvatarParameterDriver.Parameter
    { type = VRC_AvatarParameterDriver.ChangeType.Set, name = name, value = value };

static VRC_AvatarParameterDriver.Parameter Add(string name, float value) =>
    new VRC_AvatarParameterDriver.Parameter
    { type = VRC_AvatarParameterDriver.ChangeType.Add, name = name, value = value };

static VRC_AvatarParameterDriver.Parameter Remap(
    string source, float sourceMin, float sourceMax,
    string name, float destMin, float destMax) =>
    new VRC_AvatarParameterDriver.Parameter
    {
        type = VRC_AvatarParameterDriver.ChangeType.Copy, name = name, source = source,
        convertRange = true,
        sourceMin = sourceMin, sourceMax = sourceMax, destMin = destMin, destMax = destMax
    };

static VRC_AvatarParameterDriver.Parameter RandomRange(string name, float valueMin, float valueMax) =>
    new VRC_AvatarParameterDriver.Parameter
    { type = VRC_AvatarParameterDriver.ChangeType.Random, name = name, valueMin = valueMin, valueMax = valueMax };

static VRC_AvatarParameterDriver.Parameter RandomChance(string name, float chance) =>
    new VRC_AvatarParameterDriver.Parameter
    { type = VRC_AvatarParameterDriver.ChangeType.Random, name = name, chance = chance };

static void AddDriver(AnimatorState state, bool localOnly,
    params VRC_AvatarParameterDriver.Parameter[] operations)
{
    var driver = state.AddStateMachineBehaviour<VRCAvatarParameterDriver>();
    driver.localOnly = localOnly;
    foreach (var operation in operations) driver.parameters.Add(operation);
}

// Constant assignment and a counter step: Mode := 3, Count += 1.
AddDriver(compute, false,
    Set("Mode", 3),
    Add("Count", 1));

// Multiplication and division by a constant, as range
// conversions: Scaled := Speed * 2, Halved := Speed / 2.
AddDriver(compute, false,
    Remap("Speed", 0f, 1f, "Scaled", 0f, 2f),
    Remap("Speed", 0f, 1f, "Halved", 0f, 0.5f));

// An affine form in one driver: Out := In * 0.5 + 0.25. The Add
// runs after the Copy because operations run in list order.
AddDriver(compute, false,
    Remap("In", 0f, 1f, "Out", 0f, 0.5f),
    Add("Out", 0.25f));

// Random, run locally; the synced destinations carry the
// results to the other instances.
AddDriver(compute, true,
    RandomRange("Seed", 0f, 1f),
    RandomChance("CoinFlip", 0.5f));
```

```csharp
// [Editor: generator]
// The compute state: an idle state, the compute state holding
// the drivers, an entry transition on the condition that needs
// the value, and an immediate return.
var idle = layer.stateMachine.AddState("Idle", new Vector3(0, 0, 0));
var compute = layer.stateMachine.AddState("Compute", new Vector3(250, 0, 0));
layer.stateMachine.defaultState = idle;
compute.motion = emptyClip;

var enter = idle.AddTransition(compute);
enter.hasExitTime = false;
enter.duration = 0f;
enter.AddCondition(AnimatorConditionMode.If, 0f, "ComputeNow");

var back = compute.AddTransition(idle);
back.hasExitTime = false;
back.duration = 0f;
```
