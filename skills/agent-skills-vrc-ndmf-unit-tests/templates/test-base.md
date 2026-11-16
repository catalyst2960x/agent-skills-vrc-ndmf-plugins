# Template: Test base

## When it applies

A plugin's requirements are verified as editor tests (NUnit)
against build output: the generated controllers, clips, and
reports, read statically after a build. Without a shared base,
each test re-derives the same setup and leaves its objects and
assets behind, so one test's residue fails the next. The base
provides four things: tracked cleanup of everything a test
creates, a minimal avatar, the two build entry points (a direct
BuildContext for one pass, and the full pipeline), and
inspection helpers that read the built avatar's FX controller.

The tests live in an Editor assembly that references the test
framework. The folder name carries a trailing `~` in a
distributed package, so the tests are not imported by the
package's users; the adopting project links or copies the
folder in for its own development.

The test assembly definition (JSON). The assembly names
`Example.Plugin.Tests`, `Example.Plugin.Runtime`, and
`Example.Plugin.Editor` are stand-ins to replace with the
adopting plugin's own assembly names. The SDK types the
fixtures and assertions name (the avatar descriptor, the
state behaviours, the expression parameters) ship in the
SDK package as precompiled DLLs — VRCSDK3A.dll and
VRCSDKBase.dll, with their Editor counterparts — not as
source assemblies, so `overrideReferences` stays false and
the DLLs are referenced automatically; a project that sets
it true must enumerate those DLLs in precompiledReferences,
or the SDK types do not resolve (CS0234, CS0246) while the
product code still compiles:

```json
{
    "name": "Example.Plugin.Tests",
    "references": [
        "UnityEngine.TestRunner",
        "UnityEditor.TestRunner",
        "nadena.dev.ndmf",
        "nadena.dev.ndmf.vrchat",
        "nadena.dev.modular-avatar.core",
        "nadena.dev.modular-avatar.core.editor",
        "VRC.SDK3A",
        "VRC.SDK3A.Editor",
        "VRC.SDKBase",
        "VRC.SDKBase.Editor",
        "Example.Plugin.Runtime",
        "Example.Plugin.Editor"
    ],
    "includePlatforms": [ "Editor" ],
    "overrideReferences": false,
    "precompiledReferences": [ "nunit.framework.dll" ],
    "autoReferenced": false,
    "defineConstraints": [ "UNITY_INCLUDE_TESTS" ]
}
```

## Skeleton

```csharp
// [Editor: unit tests]
using System.Collections.Generic;
using nadena.dev.ndmf;
using NUnit.Framework;
using UnityEditor;
using UnityEditor.Animations;
using UnityEngine;
using VRC.SDK3.Avatars.Components;
using VRC.SDK3.Avatars.ScriptableObjects;
using VRC.SDKBase;

// The test base: tracked cleanup, a minimal avatar, the build
// entry points, and the FX inspection helpers.
public abstract class PluginTestBase
{
    protected const string TempAssetPath = "Assets/ZZZ_TestTemp";

    private readonly List<Object> _tracked = new List<Object>();

    // Everything a test creates goes through Track, so teardown
    // can remove all of it.
    protected T Track<T>(T obj) where T : Object
    {
        if (EditorUtility.IsPersistent(obj))
            throw new System.Exception("Tracked objects must not be persistent assets");
        _tracked.Add(obj);
        return obj;
    }

    [TearDown]
    public void TearDown()
    {
        // The asset flag is required: tracked controllers and
        // clips are asset objects, and destroying one without
        // it throws in teardown, failing tests whose assertions
        // passed. Track has rejected persistent assets, so the
        // flag reaches only objects the test itself created.
        foreach (var obj in _tracked)
            if (obj != null) Object.DestroyImmediate(obj, true);
        _tracked.Clear();
        AssetDatabase.DeleteAsset(TempAssetPath);
    }

    // A minimal VRChat avatar: an Animator, a descriptor with
    // populated playable layers (FX carries an empty
    // controller), and empty Expression Parameters.
    protected GameObject CreateMinimalAvatar(string name = "Avatar")
    {
        var root = Track(new GameObject(name));
        root.AddComponent<Animator>();
        var descriptor = root.AddComponent<VRCAvatarDescriptor>();
        descriptor.customizeAnimationLayers = true;
        var fx = Track(new AnimatorController());
        fx.name = "FX";
        descriptor.baseAnimationLayers = new[]
        {
            new VRCAvatarDescriptor.CustomAnimLayer { type = VRCAvatarDescriptor.AnimLayerType.Base },
            new VRCAvatarDescriptor.CustomAnimLayer { type = VRCAvatarDescriptor.AnimLayerType.Additive },
            new VRCAvatarDescriptor.CustomAnimLayer { type = VRCAvatarDescriptor.AnimLayerType.Gesture },
            new VRCAvatarDescriptor.CustomAnimLayer { type = VRCAvatarDescriptor.AnimLayerType.Action },
            new VRCAvatarDescriptor.CustomAnimLayer
            {
                type = VRCAvatarDescriptor.AnimLayerType.FX,
                isEnabled = true,
                animatorController = fx,
            },
        };
        var expressionParameters = Track(ScriptableObject.CreateInstance<VRCExpressionParameters>());
        expressionParameters.parameters = new VRCExpressionParameters.Parameter[0];
        descriptor.expressionParameters = expressionParameters;
        return root;
    }

    // The prefab form of a fixture: a shared asset under the
    // test folder, instantiated per test and tracked.
    protected GameObject LoadPrefabFixture(string path)
    {
        var prefab = AssetDatabase.LoadAssetAtPath<GameObject>(path);
        Assert.NotNull(prefab, "fixture prefab not found: " + path);
        return Track(Object.Instantiate(prefab));
    }

    protected GameObject CreateChild(GameObject parent, string name)
    {
        var child = Track(new GameObject(name));
        child.transform.SetParent(parent.transform, false);
        return child;
    }

    // A direct context over a clone of the avatar, for running
    // one pass without the full pipeline.
    protected BuildContext CreateContext(GameObject root)
        => new BuildContext(root, TempAssetPath);

    // The full pipeline through NDMF's manual build entry, with
    // the reports made during the build captured for assertion.
    protected GameObject Build(GameObject root, out List<ErrorContext> errors)
    {
        GameObject built = null;
        errors = ErrorReport.CaptureErrors(
            () => built = AvatarProcessor.ManualProcessAvatar(root));
        if (built != null) Track(built);
        return built;
    }

    // The built avatar's FX controller, and lookup by name
    // within it.
    protected static AnimatorController FindFxController(GameObject builtAvatar)
    {
        var descriptor = builtAvatar.GetComponent<VRCAvatarDescriptor>();
        foreach (var layer in descriptor.baseAnimationLayers)
            if (layer.type == VRCAvatarDescriptor.AnimLayerType.FX)
                return layer.animatorController as AnimatorController;
        return null;
    }

    // AnimatorControllerLayer is a struct, so the finder
    // returns a nullable: a missing layer is null, not a
    // default layer with no name.
    protected static AnimatorControllerLayer? FindLayer(AnimatorController controller, string name)
    {
        foreach (var layer in controller.layers)
            if (layer.name == name) return layer;
        return null;
    }

    // Presence has dedicated helpers, so a case never
    // consumes the nullable finder itself.
    protected static bool HasLayer(AnimatorController controller, string name)
        => FindLayer(controller, name) != null;

    protected static void AssertLayerAbsent(AnimatorController controller, string name, string context = null)
        => Assert.IsNull(FindLayer(controller, name),
            "layer '" + name + "' should not exist; layers are: " +
            string.Join(", ", System.Array.ConvertAll(controller.layers, l => l.name)) +
            (context == null ? "" : "; " + context));

    // The asserting counterpart of FindLayer: on failure the
    // message names the layers the controller actually has.
    // It loops and fails directly, so the caller receives the
    // layer itself, never a nullable to unwrap. The context
    // argument carries call-site diagnostics (after a build,
    // the report summary).
    protected static AnimatorControllerLayer RequireLayer(AnimatorController controller, string name, string context = null)
    {
        foreach (var layer in controller.layers)
            if (layer.name == name) return layer;
        Assert.Fail("layer '" + name + "' not found; layers are: " +
            string.Join(", ", System.Array.ConvertAll(controller.layers, l => l.name)) +
            (context == null ? "" : "; " + context));
        return default;
    }

    // The avatar-relative path of an object in the built
    // hierarchy, built from its real names. Restructuring
    // renames what it moves — an armature merge appends a
    // suffix to the moved object's name — so an expected
    // binding path is resolved from the built object, never
    // written from the scene-side name.
    protected static string PathOf(GameObject obj)
    {
        var names = new List<string>();
        for (var t = obj.transform; t != null && t.parent != null; t = t.parent)
            names.Add(t.name);
        names.Reverse();
        return string.Join("/", names);
    }

    protected static AnimatorState FindState(AnimatorControllerLayer layer, string name)
    {
        foreach (var child in layer.stateMachine.states)
            if (child.state.name == name) return child.state;
        return null;
    }

    // The asserting counterpart of FindState: on failure the
    // message names the states the layer actually has, where
    // the plain finder surfaces downstream as a null reference.
    protected static AnimatorState RequireState(AnimatorControllerLayer layer, string name)
    {
        var state = FindState(layer, name);
        Assert.NotNull(state,
            "state '" + name + "' not found in layer '" + layer.name + "'; states are: " +
            string.Join(", ", System.Array.ConvertAll(layer.stateMachine.states, c => c.state.name)));
        return state;
    }

    // Every transition in a layer, one per line — source,
    // destination, and each condition's mode, parameter, and
    // threshold — for the failure messages of transition
    // assertions, whose mismatch is otherwise a bare count.
    protected static string TransitionSummary(AnimatorControllerLayer layer)
    {
        var parts = new List<string>();
        foreach (var child in layer.stateMachine.states)
        foreach (var transition in child.state.transitions)
        {
            var conditions = new List<string>();
            foreach (var condition in transition.conditions)
                conditions.Add(condition.mode + " " + condition.parameter + " " + condition.threshold);
            parts.Add(child.state.name + " -> " +
                (transition.destinationState != null ? transition.destinationState.name : "Exit") +
                " [" + string.Join("; ", conditions) + "]");
        }
        return parts.Count + " transition(s): " + string.Join(" | ", parts);
    }

    protected static AnimationClip FindClip(AnimatorState state)
        => state.motion as AnimationClip;

    // A one-line summary of a captured report set, for
    // assertion messages: each entry's severity, the pass
    // that reported it, and its message.
    protected static string ReportSummary(List<ErrorContext> errors)
    {
        var parts = new List<string>();
        foreach (var error in errors)
            parts.Add(error.TheError.Severity + " [" + error.PassName + "] " + error.TheError.ToMessage());
        return errors.Count + " report(s): " + string.Join("; ", parts);
    }
}
```
