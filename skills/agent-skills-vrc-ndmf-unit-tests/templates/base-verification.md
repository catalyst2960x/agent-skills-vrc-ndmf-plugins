# Template: Test base verification cases

## When it applies

Every plugin carries these cases once, against its own test
base: the fixture the whole suite stands on builds clean,
tests do not leak into each other, and the base can switch
the platform it builds for. A defect in the base invalidates
every case built on it while looking like a product defect,
so the base is verified before the cases that use it. The
samples run against the schematic `NDMFComponent`, a
document-local stand-in that generates one FX layer per
component named `NDMFComponent [<target name>]`.

## Skeleton

```csharp
// [Editor: unit tests]
// Stand-ins to implement (document-local; contracts in the introduction): NDMFComponent
using System.Linq;
using nadena.dev.ndmf;
using NUnit.Framework;
using UnityEditor;
using UnityEngine;

public class BaseVerificationTests : PluginTestBase
{
    // BASE-01 (the test base): the minimal avatar builds with no report at all.
    [Test]
    public void MinimalAvatarBuildsClean()
    {
        var root = CreateMinimalAvatar();

        var built = Build(root, out var errors);

        Assert.NotNull(built);
        Assert.IsEmpty(errors);
    }

    // BASE-02: after teardown the fixture and the temp folder are gone, and a rebuilt fixture gives the same result.
    [Test]
    public void BuildsDoNotLeakIntoEachOther()
    {
        var root = CreateMinimalAvatar();
        CreateChild(root, "Toggle").AddComponent<NDMFComponent>().Target = CreateChild(root, "Target");

        var first = Build(root, out _);
        var firstLayers = FindFxController(first).layers.Select(l => l.name).ToArray();
        var rootName = root.name;

        TearDown();
        Assert.IsNull(GameObject.Find(rootName));
        Assert.IsFalse(AssetDatabase.IsValidFolder(TempAssetPath));

        var secondRoot = CreateMinimalAvatar();
        CreateChild(secondRoot, "Toggle").AddComponent<NDMFComponent>().Target = CreateChild(secondRoot, "Target");
        var second = Build(secondRoot, out _);
        var secondLayers = FindFxController(second).layers.Select(l => l.name).ToArray();

        CollectionAssert.AreEqual(firstLayers, secondLayers);
    }
}
```
