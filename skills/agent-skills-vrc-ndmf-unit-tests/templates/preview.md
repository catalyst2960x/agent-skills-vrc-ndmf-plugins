# Template: Preview cases

## When it applies

A reactive plugin carries these cases: the preview shows
what the build produces, for the effective and the
ineffective state alike; the value copies the preview state
is built from compare equal to their source and detect a
change on either side; and running the preview leaves the
scene as it was. The defect class is staleness and leakage:
the preview keeps showing the old state because a copy
compared equal when it was not, or the preview writes into
the scene it only observes.

The samples run against document-local stand-ins: the
reactive component `NDMFReactiveComponent`, whose preview
applies its target's designed active state to a proxy
renderer while the build produces the same state on the
built avatar; `ObservedState`, the plain-data state its
filter observes, with exact value equality and a deep
`Clone()`; and `PreviewProbe`, the adopting product's entry
point for running one filter frame outside a preview
session and reading the proxy it produced.

## Skeleton

```csharp
// [Editor: unit tests]
// Stand-ins to implement (document-local; contracts in the introduction): NDMFReactiveComponent, ObservedState, PreviewProbe
using NUnit.Framework;
using UnityEngine;

public class PreviewTests : PluginTestBase
{
    // PREV-01 (preview) and PREV-02: the proxy state after one frame matches the built state, effective or not.
    [Test]
    public void PreviewMatchesTheBuildForBothStates()
    {
        foreach (var designed in new[] { true, false })
        {
            var root = CreateMinimalAvatar();
            var target = CreateChild(root, "Target");
            target.SetActive(!designed);
            var component = CreateChild(root, "Reactive").AddComponent<NDMFReactiveComponent>();
            component.Target = target;
            component.DesignedActive = designed;

            var proxy = PreviewProbe.RunFrame(component, target);
            var built = Build(root, out _);
            var builtActive = built.transform.Find("Target").gameObject.activeSelf;

            Assert.AreEqual(builtActive, proxy.activeSelf);
            Assert.AreEqual(designed, proxy.activeSelf);
            TearDown();
        }
    }

    // PREV-03: a value copy equals its source, detects changes on either side, and shares nothing with it.
    [Test]
    public void ValueCopiesDetectChangesOnEitherSide()
    {
        var state = new ObservedState { IsActive = true, Level = 0.5f };

        var copy = state.Clone();
        Assert.AreEqual(state, copy);

        copy.Level = 0.9f;
        Assert.AreNotEqual(state, copy);
        Assert.AreEqual(0.5f, state.Level);

        state.IsActive = false;
        Assert.AreNotEqual(state, copy);
    }

    // PREV-04: running the preview leaves the scene's original values unchanged.
    [Test]
    public void PreviewLeavesTheSceneUnchanged()
    {
        var root = CreateMinimalAvatar();
        var target = CreateChild(root, "Target");
        target.SetActive(false);
        var component = CreateChild(root, "Reactive").AddComponent<NDMFReactiveComponent>();
        component.Target = target;
        component.DesignedActive = true;

        PreviewProbe.RunFrame(component, target);

        Assert.IsFalse(target.activeSelf);
    }
}
```
