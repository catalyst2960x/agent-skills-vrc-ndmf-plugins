# Template: Non-destructiveness cases

## When it applies

Every plugin carries these cases: the build writes to the
build clone only, and the author's scene objects, assets,
and out-of-scope components keep their values across a
build. The assertions run in the opposite direction from the
other areas — the scene side is compared, not the built
side — because unchanged is the claim.

The samples run against the test base and the schematic
`NDMFComponent`, with a target object and a numeric `Level`
setting whose field values are compared across the build.

## Skeleton

```csharp
// [Editor: unit tests]
// Stand-ins to implement (document-local; contracts in the introduction): NDMFComponent
using System.Linq;
using NUnit.Framework;
using UnityEditor.Animations;
using UnityEngine;
using VRC.SDK3.Avatars.Components;

public class NonDestructivenessTests : PluginTestBase
{
    // ND-01 (non-destructiveness): the build writes to the clone only; the scene component keeps its values.
    [Test]
    public void AuthorComponentIsUnchangedByBuild()
    {
        var root = CreateMinimalAvatar();
        var target = CreateChild(root, "Target");
        var component = CreateChild(root, "Toggle").AddComponent<NDMFComponent>();
        component.Target = target;
        component.Level = 0.25f;

        Build(root, out _);

        Assert.AreSame(target, component.Target);
        Assert.AreEqual(0.25f, component.Level);
    }

    // ND-02: the author's FX controller asset is not written; the merge lands on the clone's copy.
    [Test]
    public void AuthorControllerAssetIsUnchanged()
    {
        var root = CreateMinimalAvatar();
        var descriptor = root.GetComponent<VRCAvatarDescriptor>();
        var authorsFx = descriptor.baseAnimationLayers
            .Single(l => l.type == VRCAvatarDescriptor.AnimLayerType.FX).animatorController;
        CreateChild(root, "Toggle").AddComponent<NDMFComponent>().Target = CreateChild(root, "Target");

        Build(root, out _);

        Assert.AreEqual(0, ((AnimatorController)authorsFx).layers.Length);
    }

    // ND-04: every layer the build added carries the plugin's naming form.
    [Test]
    public void GeneratedOutputIsIdentifiable()
    {
        var root = CreateMinimalAvatar();
        CreateChild(root, "Toggle").AddComponent<NDMFComponent>().Target = CreateChild(root, "Target");
        CreateChild(root, "Toggle").AddComponent<NDMFComponent>().Target = CreateChild(root, "Other");

        var built = Build(root, out _);
        var fx = FindFxController(built);

        Assert.AreEqual(2, fx.layers.Length);
        var offenders = fx.layers.Where(l => !l.name.StartsWith("NDMFComponent [")).Select(l => l.name).ToArray();
        Assert.IsEmpty(offenders,
            "layers without the plugin's naming form: " + string.Join(", ", offenders) +
            "; all layers: " + string.Join(", ", fx.layers.Select(l => l.name)));
    }

    // ND-03: an out-of-scope component and what it carries are unchanged on the built avatar.
    [Test]
    public void OutOfScopeComponentsAreUnchanged()
    {
        var root = CreateMinimalAvatar();
        var other = CreateChild(root, "Other");
        other.AddComponent<AudioSource>().volume = 0.5f;
        CreateChild(root, "Toggle").AddComponent<NDMFComponent>().Target = CreateChild(root, "Target");

        var built = Build(root, out _);
        var builtOther = built.transform.Find("Other").GetComponent<AudioSource>();

        Assert.AreEqual(0.5f, builtOther.volume);
    }
}
```
