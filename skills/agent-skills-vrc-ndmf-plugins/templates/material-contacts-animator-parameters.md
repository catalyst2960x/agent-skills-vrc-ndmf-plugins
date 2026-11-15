# Template: Contact Receivers writing Animator parameters

## When it applies

A generated feature reads contact detection inside the
animator: a Receiver writes its detection result into a named
parameter, and generated layers branch on it. The Receiver is
configured in code (below) or placed by the author; either
way, the controller that reads the value declares the same
name and type itself, and no Expression Parameters entry is
needed.

## Skeleton

The component is `VRCContactReceiver` in the
`VRC.SDK3.Dynamics.Contact.Components` namespace, and the shape
type is `ContactBase.ShapeType` in `VRC.Dynamics`.

Writing a Bool with Constant — the value holds while any
contact is present and resets when none is:

```csharp
// [Editor: generator]
var receiver = obj.AddComponent<VRCContactReceiver>();
receiver.shapeType = ContactBase.ShapeType.Sphere;
receiver.radius = 0.1f;
receiver.receiverType = ContactReceiver.ReceiverType.Constant;
receiver.parameter = "ExampleTouch";
receiver.allowSelf = true;
receiver.allowOthers = true;
receiver.collisionTags = new List<string> { "ExampleTag" };
```

Writing a Float with Proximity — 0.0 to 1.0, closest contact
wins:

```csharp
// [Editor: generator]
var receiver = obj.AddComponent<VRCContactReceiver>();
receiver.shapeType = ContactBase.ShapeType.Sphere;
receiver.radius = 0.3f;
receiver.receiverType = ContactReceiver.ReceiverType.Proximity;
receiver.parameter = "ExampleNear";
receiver.allowSelf = false;
receiver.allowOthers = true;
receiver.collisionTags = new List<string> { "ExampleTag" };
```

The declarations on the controller side. Of the Advanced
Parameters, this also reads the other party's type:

```csharp
// [Editor: NDMF pass]
controller.AddParameter("ExampleNear", AnimatorControllerParameterType.Float);
controller.AddParameter("ExampleNear_SenderType", AnimatorControllerParameterType.Int);
```
