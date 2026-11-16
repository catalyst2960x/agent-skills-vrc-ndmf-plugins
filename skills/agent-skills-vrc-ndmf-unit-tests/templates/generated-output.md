# Template: Generated output cases

## When it applies

A generating plugin carries these cases: the structure of
what it generates, read from the built avatar's FX
controller — layer names and their numbering, layer weights,
clip curves down to their values, transition conditions and
their parameter declarations, the initial state the defaults
determine, and Write Defaults uniformity. Structure that
matches while values differ is a defect; these cases compare
values.

The samples run against the test base and a schematic
component, `NDMFComponent`, whose plugin generates one FX
layer per component named `NDMFComponent [<target name>]`,
with an Idle and an Active state, a condition parameter, a
constant `m_IsActive` curve on the target, and the
document-local `MarkerBehaviour` attached to the Active
state with its `Tag` field set to the state's name. A
second stand-in, `NDMFEmptyComponent`, generates a layer
whose clips carry no real curves by design; where a
transition needs a clip, the design substitutes a clip
carrying one dummy curve on the component's own object.

## Skeleton

```csharp
// [Editor: unit tests]
// Stand-ins to implement (document-local; contracts in the introduction): NDMFComponent, NDMFEmptyComponent, MarkerBehaviour
using System.Linq;
using nadena.dev.ndmf;
using NUnit.Framework;
using UnityEditor;
using UnityEditor.Animations;
using UnityEngine;
using VRC.SDK3.Avatars.Components;

public class GeneratedOutputTests : PluginTestBase
{
    NDMFComponent AddToggle(GameObject root, GameObject target)
    {
        var component = CreateChild(root, "Toggle").AddComponent<NDMFComponent>();
        component.Target = target;
        return component;
    }

    // GEN-02 (generated output): an added layer's weight is 1 (the layers array was written back).
    [Test]
    public void AddedLayerHasWeightOne()
    {
        var root = CreateMinimalAvatar();
        AddToggle(root, CreateChild(root, "Target"));

        var built = Build(root, out _);
        var layer = RequireLayer(FindFxController(built), "NDMFComponent [Target]");

        Assert.AreEqual(1f, layer.defaultWeight);
    }

    // GEN-01: duplicate layer names get distinct names by the deterministic rule.
    [Test]
    public void DuplicateLayerNamesAreNumbered()
    {
        var root = CreateMinimalAvatar();
        AddToggle(root, CreateChild(root, "Target"));
        AddToggle(root, CreateChild(root, "Target"));

        var built = Build(root, out _);
        var fx = FindFxController(built);

        RequireLayer(fx, "NDMFComponent [Target]");
        RequireLayer(fx, "NDMFComponent [Target] (2)");
    }

    // GEN-03: the active clip carries the target's m_IsActive curve with the configured value.
    [Test]
    public void ActiveClipCurvesTargetActive()
    {
        var root = CreateMinimalAvatar();
        AddToggle(root, CreateChild(root, "Target"));

        var built = Build(root, out _);
        var layer = RequireLayer(FindFxController(built), "NDMFComponent [Target]");
        var clip = FindClip(RequireState(layer, "Active"));

        var binding = EditorCurveBinding.FloatCurve("Target", typeof(GameObject), "m_IsActive");
        var curve = AnimationUtility.GetEditorCurve(clip, binding);
        Assert.NotNull(curve);
        Assert.AreEqual(1f, curve.keys[0].value);
    }

    // GEN-04 and GEN-05: the forward transition carries the designed condition, and its parameters are declared.
    [Test]
    public void TransitionConditionIsDeclared()
    {
        var root = CreateMinimalAvatar();
        AddToggle(root, CreateChild(root, "Target"));

        var built = Build(root, out _);
        var fx = FindFxController(built);
        var layer = RequireLayer(fx, "NDMFComponent [Target]");
        var idle = RequireState(layer, "Idle");

        var transition = idle.transitions.SingleOrDefault(t => t.destinationState.name == "Active");
        Assert.NotNull(transition, "no Idle -> Active transition; " + TransitionSummary(layer));
        Assert.AreEqual(1, transition.conditions.Length, TransitionSummary(layer));
        Assert.AreEqual(AnimatorConditionMode.Greater, transition.conditions[0].mode, TransitionSummary(layer));
        Assert.AreEqual(0.5f, transition.conditions[0].threshold, TransitionSummary(layer));

        var declared = fx.parameters.Select(p => p.name).ToList();
        foreach (var state in layer.stateMachine.states)
        foreach (var t in state.state.transitions)
        foreach (var condition in t.conditions)
            Assert.Contains(condition.parameter, declared);
    }

    // GEN-08: a designed behaviour is present on the built state, created through the state.
    [Test]
    public void AttachedBehaviourSurvivesTheBuild()
    {
        var root = CreateMinimalAvatar();
        AddToggle(root, CreateChild(root, "Target"));

        var built = Build(root, out _);
        var layer = RequireLayer(FindFxController(built), "NDMFComponent [Target]");
        var active = RequireState(layer, "Active");

        var behaviour = active.behaviours.OfType<MarkerBehaviour>().Single();
        Assert.AreEqual("Active", behaviour.Tag);
    }

    // GEN-09: empty generation material yields the designed layer and the dummy curve.
    [Test]
    public void EmptyMaterialUsesTheDesignedSubstitute()
    {
        var root = CreateMinimalAvatar();
        CreateChild(root, "Empty").AddComponent<NDMFEmptyComponent>();

        var built = Build(root, out _);
        var layer = RequireLayer(FindFxController(built), "NDMFEmptyComponent");
        Assert.NotNull(layer);

        var clip = FindClip(RequireState(layer, "Active"));
        var bindings = clip.GetCurveBindings();
        Assert.AreEqual(1, bindings.Length);
        Assert.AreEqual("Empty", bindings[0].path);
        Assert.AreEqual("m_IsActive", bindings[0].propertyName);
    }

    // GEN-06: the default state is the one the condition's default selects, read statically.
    [Test]
    public void DefaultStateFollowsInitialActiveState()
    {
        var activeRoot = CreateMinimalAvatar();
        var activeTarget = CreateChild(activeRoot, "Target");
        AddToggle(activeRoot, activeTarget);

        var inactiveRoot = CreateMinimalAvatar();
        var inactiveTarget = CreateChild(inactiveRoot, "Target");
        inactiveTarget.SetActive(false);
        AddToggle(inactiveRoot, inactiveTarget);

        var activeLayer = RequireLayer(FindFxController(Build(activeRoot, out _)), "NDMFComponent [Target]");
        var inactiveLayer = RequireLayer(FindFxController(Build(inactiveRoot, out _)), "NDMFComponent [Target]");

        Assert.AreEqual("Active", activeLayer.stateMachine.defaultState.name);
        Assert.AreEqual("Idle", inactiveLayer.stateMachine.defaultState.name);
    }

    // GEN-07: under mixed Write Defaults, every generated state carries the designed value (off).
    [Test]
    public void GeneratedStatesShareWriteDefaults()
    {
        var root = CreateMinimalAvatar();
        var descriptor = root.GetComponent<VRCAvatarDescriptor>();
        var fx = Track(new AnimatorController());
        var existing = new AnimatorControllerLayer { name = "Existing", defaultWeight = 1f };
        existing.stateMachine = Track(new AnimatorStateMachine());
        existing.stateMachine.AddState("ExistingState").writeDefaultValues = true;
        fx.layers = new[] { existing };
        for (var i = 0; i < descriptor.baseAnimationLayers.Length; i++)
            if (descriptor.baseAnimationLayers[i].type == VRCAvatarDescriptor.AnimLayerType.FX)
                descriptor.baseAnimationLayers[i].animatorController = fx;

        AddToggle(root, CreateChild(root, "Target"));
        var built = Build(root, out _);
        var layer = RequireLayer(FindFxController(built), "NDMFComponent [Target]");

        Assert.IsTrue(layer.stateMachine.states.All(s => !s.state.writeDefaultValues));
    }
}
```
