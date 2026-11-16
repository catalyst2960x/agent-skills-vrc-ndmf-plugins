# Template: Hierarchy restructuring cases

## When it applies

A plugin that merges, chains, or cleans up hierarchy —
typically a mesh-processing or restructuring plugin —
carries these cases: reference cycles are detected and
reported instead of looping, a chain that matches only
partially is left alone, and objects the retention rules
protect survive a cleanup that removes the unprotected.
Each case runs both sides: the shape that must fail or be
kept, and the shape that must succeed or be removed.

The samples run against document-local stand-ins:
`NDMFChainComponent`, whose contract is to merge its target
chain into its anchor when every link of the chain matches,
following `Next` references; and `NDMFCleanupComponent`,
whose contract is to remove helper objects under its root
except end bones and objects an animation references.

## Skeleton

```csharp
// [Editor: unit tests]
// Stand-ins to implement (document-local; contracts in the introduction): NDMFChainComponent, NDMFCleanupComponent
using System.Linq;
using nadena.dev.ndmf;
using NUnit.Framework;
using UnityEngine;

public class HierarchyRestructuringTests : PluginTestBase
{
    // HIER-01 (hierarchy restructuring): a reference cycle reports NonFatal and merges nothing; without it, the same shapes merge.
    [Test]
    public void ReferenceCyclesAreDetected()
    {
        var cyclic = CreateMinimalAvatar();
        var a = CreateChild(cyclic, "A");
        var b = CreateChild(cyclic, "B");
        a.AddComponent<NDMFChainComponent>().Next = b;
        b.AddComponent<NDMFChainComponent>().Next = a;

        var builtCyclic = Build(cyclic, out var cyclicErrors);

        Assert.IsTrue(cyclicErrors.Any(e => e.TheError.Severity == ErrorSeverity.NonFatal));
        Assert.NotNull(builtCyclic.transform.Find("A"));
        Assert.NotNull(builtCyclic.transform.Find("B"));

        var acyclic = CreateMinimalAvatar();
        var first = CreateChild(acyclic, "A");
        var second = CreateChild(acyclic, "B");
        first.AddComponent<NDMFChainComponent>().Next = second;

        Build(acyclic, out var acyclicErrors);

        Assert.IsFalse(acyclicErrors.Any(e => e.TheError.Severity == ErrorSeverity.NonFatal));
    }

    // HIER-02: a chain whose last link does not match is not merged at all.
    [Test]
    public void PartialChainsAreNotMerged()
    {
        var root = CreateMinimalAvatar();
        var a = CreateChild(root, "A");
        var b = CreateChild(root, "B");
        var chain = a.AddComponent<NDMFChainComponent>();
        chain.Next = b;
        chain.ExpectedLinks = 3;

        var built = Build(root, out _);

        Assert.NotNull(built.transform.Find("A"));
        Assert.NotNull(built.transform.Find("B"));
    }

    // HIER-03: cleanup removes an unprotected helper and keeps both protected kinds.
    [Test]
    public void CleanupKeepsProtectedObjects()
    {
        var root = CreateMinimalAvatar();
        var cleanup = CreateChild(root, "Cleanup").AddComponent<NDMFCleanupComponent>();
        var helper = CreateChild(root, "Helper");
        var endBone = CreateChild(root, "EndBone");
        var animated = CreateChild(root, "Animated");
        cleanup.Root = root;
        cleanup.EndBones = new[] { endBone };
        cleanup.Animated = new[] { animated };

        var built = Build(root, out _);

        Assert.IsNull(built.transform.Find("Helper"));
        Assert.NotNull(built.transform.Find("EndBone"));
        Assert.NotNull(built.transform.Find("Animated"));
    }
}
```
