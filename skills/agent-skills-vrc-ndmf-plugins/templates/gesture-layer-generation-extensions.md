# Template: Gesture layer generation — variants, force mode, hand lock, ramp endpoints

## When it applies

The base gesture layer generation (gesture-layer-generation) is in
use, and the product needs the variations a production gesture
controller carries: several motions for one gesture selected by a
parameter, a parallel state set that takes the fingers over from
tracking, a lock that freezes the hand, and a Fist ramp whose two
ends are assigned independently of each other and of the other
gesture slots.

This template extends the base generation. It does not replace it:
the controller, the hand layers, the masks, the discrete entry
conditions, and the motion-time wiring are the base template's.

## Requirements

- Variant selection: a gesture may carry an array of variant clips
  and the name of an Int selector parameter. Variant 0 is the base
  form. A variant above 0 is entered under the gesture equality
  plus the selector equality for its index. The base form's entry
  carries no selector condition and is added after the variant
  entries, so a selector value with no variant falls back to the
  base form.
- Force mode: when a mode parameter name is set, the hand's state
  set is generated twice. The normal set's entries carry the mode
  parameter's IfNot condition; the force set's entries carry its
  If condition, and the force states carry an Animator Tracking
  Control behaviour that sets that hand's fingers to Animation.
  Force state names are the base names plus a suffix.
- Hand lock: when a lock parameter name and a lock clip are set,
  each hand gains a lock state holding the lock clip, with no
  motion-time parameter. Its entry condition is the lock
  parameter's If. Every other entry in the layer gains the lock
  parameter's IfNot condition as a guard, including the force
  set's entries and the variant entries.
- Independent ramp endpoints: the Fist ramp's start and end are
  separate clip assignments per hand. The start motion contributes
  the pose at its first frame; the end motion contributes the pose
  at its last frame. Either endpoint may be any clip — a dedicated
  pose, a clip shared with another gesture slot, or a timed clip.
  The ramp's bindings come from the end motion; a binding the
  start motion does not have starts from the end motion's
  first-frame value.
- Entry conditions are assembled in one place: the gesture
  equality first, then the guard, the mode condition, and the
  selector equality, so every entry in a layer carries the same
  guard and mode shape.
- The configured selector, mode, and lock parameters are declared
  in the generated controller alongside the built-ins (Int for a
  selector, Bool for mode and lock), so the generated layers can
  read them. Registering them in Expression Parameters, when the
  player is to drive them from a menu, is the product's own step.

## Skeleton

```csharp
// [Runtime: authoring component]
// The extension fields, per hand and per set. VictoryVariants[0]
// is the base form; the base template's single Victory clip is the
// array of length 1. FistRampStart and FistRampEnd replace the
// base template's FistRelaxed and Fist as the ramp's endpoints.
[Serializable]
public sealed class GestureHandClipsExt
{
    public AnimationClip Idle;
    public AnimationClip FistRampStart;
    public AnimationClip FistRampEnd;
    public AnimationClip HandOpen;
    public AnimationClip FingerPoint;
    public AnimationClip[] VictoryVariants;
    public string VictorySelector;      // null: no variants
    public AnimationClip RockNRoll;
    public AnimationClip HandGun;
    public AnimationClip ThumbsUp;
    public bool ForceMode;              // generate the force set
}

public sealed class GestureSetExt : MonoBehaviour
{
    public GestureHandClipsExt Left;
    public GestureHandClipsExt Right;
    public float FistRampSeconds = 1f / 3f;
    public string ForceParameter;       // null: no force mode
    public string LockParameter;        // null: no hand lock
    public AnimationClip LockClip;
}
```

```csharp
// [Editor: generator]
// One entry transition, assembled from the gesture equality and
// the extra conditions the extensions add. Every entry in a layer
// goes through here, so the guard and mode conditions cannot be
// forgotten on one state.
static void AddEntry(AnimatorStateMachine sm, AnimatorState destination,
    string gestureParameter, int value, params AnimatorCondition[] extra)
{
    var transition = sm.AddAnyStateTransition(destination);
    transition.AddCondition(AnimatorConditionMode.Equals, value, gestureParameter);
    foreach (var condition in extra) transition.AddCondition(
        condition.mode, condition.threshold, condition.parameter);
    transition.hasExitTime = false;
    transition.duration = 0f;
    transition.canTransitionToSelf = false;
}

static AnimatorCondition Condition(AnimatorConditionMode mode, float threshold, string parameter)
    => new AnimatorCondition { mode = mode, threshold = threshold, parameter = parameter };

// The conditions every normal entry carries, and every force
// entry: the lock guard when a lock is configured, then the mode
// condition for the set being generated.
static AnimatorCondition[] EntryExtras(GestureSetExt source, bool forceSet)
{
    var extras = new List<AnimatorCondition>();
    if (source.LockParameter != null)
        extras.Add(Condition(AnimatorConditionMode.IfNot, 0f, source.LockParameter));
    if (source.ForceParameter != null)
        extras.Add(Condition(forceSet ? AnimatorConditionMode.If : AnimatorConditionMode.IfNot,
            0f, source.ForceParameter));
    return extras.ToArray();
}
```

```csharp
// [Editor: generator]
// The gesture states of one set (normal or force), with variant
// selection on Victory. Called twice for a force-mode hand: once
// with an empty suffix and IfNot extras, once with the force
// suffix, If extras, and Tracking Control on every state.
static void AddGestureStates(AnimatorStateMachine sm, GestureHandClipsExt clips,
    string gestureParameter, string weightParameter, float fistRampSeconds,
    AnimatorCondition[] extras, string suffix, bool tracking)
{
    bool leftHand = weightParameter == "GestureLeftWeight";

    // Every state of the set goes through here, so a force set
    // carries Tracking Control on all of its states.
    AnimatorState S(string name, AnimationClip clip, string weight)
    {
        var state = AddGestureState(sm, name, clip, weight);
        if (tracking) AddTrackingControl(state, leftHand);
        return state;
    }
    void Enter(AnimatorState state, int value, params AnimatorCondition[] more)
        => AddEntry(sm, state, gestureParameter, value, extras.Concat(more).ToArray());

    var idle = S("Idle" + suffix, clips.Idle, null);
    if (suffix == "") sm.defaultState = idle;
    Enter(idle, 0);

    Enter(S("Fist" + suffix,
        SynthesizeRamp(clips.FistRampStart, clips.FistRampEnd, fistRampSeconds), weightParameter), 1);
    Enter(S("HandOpen" + suffix, clips.HandOpen, weightParameter), 2);
    Enter(S("FingerPoint" + suffix, clips.FingerPoint, weightParameter), 3);

    // Variants first, the base form last with no selector
    // condition, so unlisted selector values fall back to it.
    var variants = clips.VictoryVariants;
    for (int i = 1; i < variants.Length; i++)
        Enter(S("Victory" + i + suffix, variants[i], weightParameter), 4,
            Condition(AnimatorConditionMode.Equals, i, clips.VictorySelector));
    Enter(S("Victory" + suffix, variants[0], weightParameter), 4);

    Enter(S("RockNRoll" + suffix, clips.RockNRoll, weightParameter), 5);
    Enter(S("HandGun" + suffix, clips.HandGun, weightParameter), 6);
    Enter(S("ThumbsUp" + suffix, clips.ThumbsUp, weightParameter), 7);
}

// The lock state: entered on the lock parameter alone, ahead of
// every guarded entry. No gesture equality, no motion time.
static void AddLockState(AnimatorStateMachine sm, GestureSetExt source)
{
    var lockState = AddGestureState(sm, "HandLock", source.LockClip, null);
    var transition = sm.AddAnyStateTransition(lockState);
    transition.AddCondition(AnimatorConditionMode.If, 0f, source.LockParameter);
    transition.hasExitTime = false;
    transition.duration = 0f;
    transition.canTransitionToSelf = false;
}
```

```csharp
// [Editor: generator]
// The Fist ramp with independent endpoints: the start motion is
// sampled at its first frame, the end motion at its last frame.
// Bindings come from the end motion; a binding the start motion
// lacks starts from the end motion's first-frame value.
static AnimationClip SynthesizeRamp(AnimationClip startMotion, AnimationClip endMotion, float seconds)
{
    var ramp = new AnimationClip { name = endMotion.name + " Ramp" };
    foreach (var binding in AnimationUtility.GetCurveBindings(endMotion))
    {
        var endCurve = AnimationUtility.GetEditorCurve(endMotion, binding);
        var startCurve = AnimationUtility.GetEditorCurve(startMotion, binding);
        float start = startCurve != null ? startCurve.Evaluate(0f) : endCurve.Evaluate(0f);
        float end = endCurve.Evaluate(endMotion.length);
        AnimationUtility.SetEditorCurve(ramp, binding,
            new AnimationCurve(new Keyframe(0f, start), new Keyframe(seconds, end)));
    }
    return ramp;
}
```
