# Template: Editor operation cases

## When it applies

An editor-support plugin carries these cases: an operation
the tool performs is undoable, and applied to a prefab
instance it remains as an override on the instance without
touching the asset; a setup operation run twice changes
nothing the second time; and inference from names decides
confusable input by the designed rule instead of picking a
wrong match.

The samples run against document-local stand-ins:
`SetupOperation`, the tool's one operation, which assigns
the component's target by the designed rule — an exact name
match in the avatar, else the first candidate in hierarchy
order — recording its edits with Undo; and the schematic
`NDMFComponent` it operates on. `FixturePrefabPath` is the
adopting product's constant naming its fixture prefab
asset, an avatar carrying the component unconfigured.

## Skeleton

```csharp
// [Editor: unit tests]
// Stand-ins to implement (document-local; contracts in the introduction): NDMFComponent, SetupOperation, FixturePrefabPath
using NUnit.Framework;
using UnityEditor;
using UnityEngine;

public class EditorOperationTests : PluginTestBase
{
    // EDIT-01 (editor operations): the edit is one undo step; undoing restores the prior state.
    [Test]
    public void OperationsAreUndoable()
    {
        var root = CreateMinimalAvatar();
        var target = CreateChild(root, "Target");
        var component = CreateChild(root, "Toggle").AddComponent<NDMFComponent>();

        SetupOperation.AssignTarget(component);
        Assert.AreSame(target, component.Target);

        Undo.PerformUndo();
        Assert.IsNull(component.Target);
    }

    // EDIT-01, prefab side: on a prefab instance the change stays an override; the asset is unchanged.
    [Test]
    public void PrefabInstancesKeepOverrides()
    {
        var instance = LoadPrefabFixture(FixturePrefabPath);
        var component = instance.GetComponentInChildren<NDMFComponent>();

        SetupOperation.AssignTarget(component);

        Assert.IsTrue(PrefabUtility.IsPartOfPrefabInstance(instance));
        Assert.NotNull(component.Target);
        var asset = AssetDatabase.LoadAssetAtPath<GameObject>(FixturePrefabPath);
        Assert.IsNull(asset.GetComponentInChildren<NDMFComponent>().Target);
    }

    // EDIT-02: running the operation twice changes nothing the second time.
    [Test]
    public void SetupIsIdempotent()
    {
        var root = CreateMinimalAvatar();
        CreateChild(root, "Target");
        var component = CreateChild(root, "Toggle").AddComponent<NDMFComponent>();

        SetupOperation.AssignTarget(component);
        var afterFirst = EditorJsonUtility.ToJson(component);
        SetupOperation.AssignTarget(component);
        var afterSecond = EditorJsonUtility.ToJson(component);

        Assert.AreEqual(afterFirst, afterSecond);
    }

    // EDIT-03: inference picks the designed candidate (first in hierarchy order), not a later partial match.
    [Test]
    public void InferenceDecidesConfusableInputByRule()
    {
        var root = CreateMinimalAvatar();
        var first = CreateChild(root, "Target");
        CreateChild(root, "TargetExtra");
        var component = CreateChild(root, "Toggle").AddComponent<NDMFComponent>();

        SetupOperation.AssignTarget(component);

        Assert.AreSame(first, component.Target);
    }
}
```
