# Template: Generating a Gesture layer with weight-scrubbed Fist states

## When it applies

A plugin generates hand gesture content for the Gesture playable
layer: one layer per hand, states selected by the built-in gesture
parameters, and a Fist pose that follows the analog weight instead
of switching discretely. The same generation also has to answer
finger-tracking environments, where tracked fingers overwrite hand
animation unless the layer either writes nothing or takes the
fingers over explicitly.

The structure follows the SDK sample hands layer
(vrc_AvatarV3HandsLayer), which already wires each gesture state
this way; the sample's own clips are single poses, so the wiring is
latent there. This template keeps the sample's wiring and supplies
the clip form that makes it take effect.

## Requirements

- One generated AnimatorController, merged into the Gesture layer
  by a ModularAvatarMergeAnimator whose layer type is Gesture and
  whose matchAvatarWriteDefaults is true. The generating pass runs
  in the Generating phase, before Modular Avatar.
- The controller declares the four built-in parameters by name and
  type: GestureLeft and GestureRight (Int), GestureLeftWeight and
  GestureRightWeight (Float). Built-in parameters are read from the
  controller declaration only; they are never registered in
  Expression Parameters and never declared through
  ModularAvatarParameters.
- One layer per hand. The layer's mask is a generated AvatarMask
  with every humanoid body part inactive except that hand's
  fingers part (LeftFingers for the left hand, RightFingers for
  the right), and no transform elements — the shape the SDK
  sample's hand masks have.
- States: an Idle state as the default, plus one state per gesture
  value 1–7. Every state has Write Defaults off. Movement is by
  AnyState transitions on Int equality against the hand's gesture
  parameter: Idle on 0, each gesture state on its value.
- Every gesture state carries the hand's weight as its motion-time
  parameter (timeParameterActive, timeParameter). A state whose
  clip is a single pose is unaffected by the wiring; the wiring is
  what lets a timed clip be scrubbed by the weight.
- The Fist clip is a ramp synthesized from two author-supplied
  single-pose clips (relaxed hand, full fist): every curve of the
  two pose clips becomes a two-keyframe curve holding the relaxed
  value at time 0 and the fist value at the ramp length. The other
  gestures take the author's pose clips unchanged.
- Tracking answer, chosen per product: states with no motion leave
  the fingers to tracking; a parallel state set carrying an
  Animator Tracking Control behaviour that sets the fingers to
  Animation takes the fingers over, gated by an author parameter
  in the transition conditions.

## Skeleton

```csharp
// [Runtime: authoring component]
// The author's inputs: one pose clip per gesture, plus the relaxed
// pose the Fist ramp starts from. Poses are single-pose clips.
[Serializable]
public sealed class GestureHandClips
{
    public AnimationClip Idle;
    public AnimationClip FistRelaxed;
    public AnimationClip Fist;
    public AnimationClip HandOpen;
    public AnimationClip FingerPoint;
    public AnimationClip Victory;
    public AnimationClip RockNRoll;
    public AnimationClip HandGun;
    public AnimationClip ThumbsUp;
}

public sealed class GestureSet : MonoBehaviour
{
    public GestureHandClips Left;
    public GestureHandClips Right;
    public float FistRampSeconds = 1f / 3f;
}
```

```csharp
// [Editor: generator]
// The generation entry, called from a Generating pass. Builds the
// controller, declares the built-ins, adds one layer per hand, and
// emits the merge into the Gesture layer on the component's own
// GameObject. The component is destroyed on the build clone by the
// pass, as with any generating plugin.
static void Generate(GestureSet source)
{
    var controller = new AnimatorController { name = "Generated Gesture" };
    controller.AddParameter("GestureLeft", AnimatorControllerParameterType.Int);
    controller.AddParameter("GestureRight", AnimatorControllerParameterType.Int);
    controller.AddParameter("GestureLeftWeight", AnimatorControllerParameterType.Float);
    controller.AddParameter("GestureRightWeight", AnimatorControllerParameterType.Float);

    if (controller.layers.Length == 0) controller.AddLayer("Base Layer");
    AddHandLayer(controller, 0, "Left Hand", AvatarMaskBodyPart.LeftFingers,
        "GestureLeft", "GestureLeftWeight", source.Left, source.FistRampSeconds);
    controller.AddLayer("Right Hand");
    AddHandLayer(controller, controller.layers.Length - 1, "Right Hand",
        AvatarMaskBodyPart.RightFingers,
        "GestureRight", "GestureRightWeight", source.Right, source.FistRampSeconds);

    var merge = source.gameObject.AddComponent<ModularAvatarMergeAnimator>();
    merge.animator = controller;
    merge.layerType = VRCAvatarDescriptor.AnimLayerType.Gesture;
    merge.matchAvatarWriteDefaults = true;
}

// One hand layer: renames the layer, installs the generated mask,
// and fills the state machine. A new AnimatorController has no
// layers; the caller adds them first (AddLayer returns no index,
// so a later layer's index is layers.Length - 1 after adding).
static void AddHandLayer(AnimatorController controller, int layerIndex, string name,
    AvatarMaskBodyPart fingers, string gestureParameter, string weightParameter,
    GestureHandClips clips, float fistRampSeconds)
{
    var layers = controller.layers;
    layers[layerIndex].name = name;
    layers[layerIndex].avatarMask = MakeHandMask(fingers);
    controller.layers = layers;

    var sm = layers[layerIndex].stateMachine;
    var idle = AddGestureState(sm, "Idle", clips.Idle, null);
    sm.defaultState = idle;
    AddGestureState(sm, "Fist",
        SynthesizeRamp(clips.FistRelaxed, clips.Fist, fistRampSeconds), weightParameter);
    AddGestureState(sm, "HandOpen", clips.HandOpen, weightParameter);
    AddGestureState(sm, "FingerPoint", clips.FingerPoint, weightParameter);
    AddGestureState(sm, "Victory", clips.Victory, weightParameter);
    AddGestureState(sm, "RockNRoll", clips.RockNRoll, weightParameter);
    AddGestureState(sm, "HandGun", clips.HandGun, weightParameter);
    AddGestureState(sm, "ThumbsUp", clips.ThumbsUp, weightParameter);

    // Entry by gesture value: Idle on 0, each gesture on its value.
    var byValue = new[] { idle };
    AddEntry(sm, idle, gestureParameter, 0);
    var states = sm.states;
    for (int value = 1; value <= 7; value++)
        AddEntry(sm, states[value].state, gestureParameter, value);
}

static AnimatorState AddGestureState(AnimatorStateMachine sm, string name,
    AnimationClip clip, string weightParameter)
{
    var state = sm.AddState(name);
    state.motion = clip;
    state.writeDefaultValues = false;
    if (weightParameter != null)
    {
        state.timeParameterActive = true;
        state.timeParameter = weightParameter;
    }
    return state;
}

static void AddEntry(AnimatorStateMachine sm, AnimatorState destination,
    string gestureParameter, int value)
{
    var transition = sm.AddAnyStateTransition(destination);
    transition.AddCondition(AnimatorConditionMode.Equals, value, gestureParameter);
    transition.hasExitTime = false;
    transition.duration = 0f;
    transition.canTransitionToSelf = false;
}

// The hand mask: every body part inactive except one hand's
// fingers, and no transform elements.
static AvatarMask MakeHandMask(AvatarMaskBodyPart fingers)
{
    var mask = new AvatarMask();
    for (var part = AvatarMaskBodyPart.Root; part < AvatarMaskBodyPart.LastBodyPart; part++)
        mask.SetHumanoidBodyPartActive(part, part == fingers);
    return mask;
}

// The Fist ramp: each curve of the two pose clips becomes a
// two-keyframe curve, relaxed value at time 0 and fist value at
// the ramp length. Both pose clips must animate the same bindings.
static AnimationClip SynthesizeRamp(AnimationClip relaxed, AnimationClip fist, float seconds)
{
    var ramp = new AnimationClip { name = fist.name + " Ramp" };
    foreach (var binding in AnimationUtility.GetCurveBindings(fist))
    {
        float start = AnimationUtility.GetEditorCurve(relaxed, binding).Evaluate(0f);
        float end = AnimationUtility.GetEditorCurve(fist, binding).Evaluate(0f);
        AnimationUtility.SetEditorCurve(ramp, binding,
            new AnimationCurve(new Keyframe(0f, start), new Keyframe(seconds, end)));
    }
    return ramp;
}
```

```csharp
// [Editor: generator]
// The tracking takeover variation: a state that sets the fingers
// to Animation through an Animator Tracking Control behaviour.
// A parallel state set built with these states, entered under an
// author parameter's condition, takes the fingers over; states
// with no motion and no behaviour leave them to tracking.
static void AddTrackingControl(AnimatorState state, bool leftHand)
{
    var behaviour = state.AddStateMachineBehaviour<VRCAnimatorTrackingControl>();
    if (leftHand) behaviour.trackingLeftFingers = VRC_AnimatorTrackingControl.TrackingType.Animation;
    else behaviour.trackingRightFingers = VRC_AnimatorTrackingControl.TrackingType.Animation;
}
```
