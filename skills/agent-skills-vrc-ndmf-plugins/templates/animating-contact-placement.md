# Template: Animating a contact's activation and placement

## When it applies

A generated layer opens, closes, or moves a contact: the
objects carrying Senders and Receivers are ordinary objects,
so the layer/state pattern's object curves control detection
itself. Disabling the object removes its contacts from
detection; moving the object moves the detection range with
it (a shape is evaluated at the object's root Transform plus
its offset).

## Skeleton

Toggling activation — a clip that drives the `m_IsActive` of
the contact's object:

```csharp
// [Editor: generator]
var clip = new AnimationClip();
var binding = EditorCurveBinding.FloatCurve(
    "Contacts/ExampleSender", typeof(GameObject), "m_IsActive");
AnimationUtility.SetEditorCurve(clip, binding, AnimationCurve.Constant(0f, 0f, 1f));
```

Moving the placement — a clip that drives the local position
of a Receiver's object:

```csharp
// [Editor: generator]
var clip = new AnimationClip();
var binding = EditorCurveBinding.FloatCurve(
    "Contacts/ExampleReceiver", typeof(Transform), "m_LocalPosition.z");
AnimationUtility.SetEditorCurve(clip, binding, AnimationCurve.Constant(0f, 0f, 0.2f));
```
