# Template: Component processing cases

## When it applies

Every plugin carries these cases: how a pass treats the
components it finds — generation on a valid configuration,
the severity each problem class reports, sanitization of
serialized data, clamping at generation, and removal of every
found instance. Without them, a defect in the pass's basic
posture (generating on an unset input, stopping at the first
failure, leaving a source component behind) reaches the
author undetected.

The samples run against the test base and a schematic
component, `NDMFComponent`, with a target object, an
`ExtraTargets` array (the active clip curves every target),
a `Level` whose value the forward transition carries as
its threshold, clamped to 0..1 at generation, and a `Mode`
enum whose designed values are 0 to 2. A second stand-in,
`NDMFFailingComponent`, generates for its first target and
then fails mid-generation; its designed result is that no
part of its output remains. The adopting plugin substitutes
its own component and fields.

## Skeleton

```csharp
// [Editor: unit tests]
// Stand-ins to implement (document-local; contracts in the introduction): NDMFComponent, NDMFComponentMode, NDMFFailingComponent
using System.Linq;
using nadena.dev.ndmf;
using NUnit.Framework;
using UnityEditor;
using UnityEngine;

public class ComponentProcessingTests : PluginTestBase
{
    NDMFComponent AddToggle(GameObject root, GameObject target)
    {
        var component = CreateChild(root, "Toggle").AddComponent<NDMFComponent>();
        component.Target = target;
        return component;
    }

    // PASS-01 (component processing): a fully configured component generates, is removed, and reports nothing.
    [Test]
    public void ValidComponentGeneratesSilently()
    {
        var root = CreateMinimalAvatar();
        AddToggle(root, CreateChild(root, "Target"));

        var built = Build(root, out var errors);

        RequireLayer(FindFxController(built), "NDMFComponent [Target]");
        Assert.IsEmpty(built.GetComponentsInChildren<NDMFComponent>(true));
        Assert.IsEmpty(errors);
    }

    // PASS-02 (with ERR-01, error reporting): an unset required input generates nothing and reports Information.
    [Test]
    public void UnsetInputReportsInformationOnly()
    {
        var root = CreateMinimalAvatar();
        AddToggle(root, null);

        var built = Build(root, out var errors);

        Assert.AreEqual(0, FindFxController(built).layers.Length);
        Assert.IsTrue(errors.Any(e => e.TheError.Severity == ErrorSeverity.Information));
        Assert.IsFalse(errors.Any(e => e.TheError.Severity == ErrorSeverity.Error));
    }

    // PASS-03: a set-but-invalid value generates nothing, reports NonFatal, and does not stop the build.
    [Test]
    public void InvalidValueReportsNonFatal()
    {
        var root = CreateMinimalAvatar();
        var component = AddToggle(root, CreateChild(root, "Target"));
        component.Mode = (NDMFComponentMode)99;

        var built = Build(root, out var errors);

        Assert.AreEqual(0, FindFxController(built).layers.Length);
        Assert.IsTrue(errors.Any(e => e.TheError.Severity == ErrorSeverity.NonFatal));
        Assert.IsFalse(errors.Any(e => e.TheError.Severity == ErrorSeverity.Error));
    }

    // PASS-04: a target outside the avatar generates nothing and reports NonFatal.
    [Test]
    public void OutsideTargetReportsNonFatal()
    {
        var root = CreateMinimalAvatar();
        var outside = Track(new GameObject("Outside"));
        AddToggle(root, outside);

        var built = Build(root, out var errors);

        Assert.IsTrue(errors.Any(e => e.TheError.Severity == ErrorSeverity.NonFatal));
        Assert.IsFalse(errors.Any(e => e.TheError.Severity == ErrorSeverity.Error));
    }

    // PASS-05: every component is processed; a failing one stops neither the others nor their removal.
    [Test]
    public void AllComponentsProcessedAndRemoved()
    {
        var root = CreateMinimalAvatar();
        AddToggle(root, CreateChild(root, "First"));
        AddToggle(root, null);
        AddToggle(root, CreateChild(root, "Second"));

        var built = Build(root, out _);
        var fx = FindFxController(built);

        RequireLayer(fx, "NDMFComponent [First]");
        RequireLayer(fx, "NDMFComponent [Second]");
        Assert.IsEmpty(built.GetComponentsInChildren<NDMFComponent>(true));
    }

    // PASS-06: a null entry in the target list is skipped; the valid entries are still curved.
    [Test]
    public void NullEntriesAreSkipped()
    {
        var root = CreateMinimalAvatar();
        var component = AddToggle(root, CreateChild(root, "Target"));
        component.ExtraTargets = new[]
        {
            CreateChild(root, "Extra1"), null, CreateChild(root, "Extra2"),
        };

        var built = Build(root, out var errors);
        var layer = RequireLayer(FindFxController(built), "NDMFComponent [Target]");
        var clip = FindClip(RequireState(layer, "Active"));
        var paths = clip.GetCurveBindings().Select(b => b.path).ToArray();

        CollectionAssert.AreEquivalent(new[] { "Target", "Extra1", "Extra2" }, paths);
        Assert.IsFalse(errors.Any(e => e.TheError.Severity == ErrorSeverity.Error));
    }

    // PASS-08: a mid-generation failure leaves no partial output; the other component's output is untouched.
    [Test]
    public void MidGenerationFailureLeavesNoPartialOutput()
    {
        var root = CreateMinimalAvatar();
        AddToggle(root, CreateChild(root, "Target"));
        CreateChild(root, "Failing").AddComponent<NDMFFailingComponent>().Target =
            CreateChild(root, "FailTarget");

        var built = Build(root, out var errors);
        var fx = FindFxController(built);

        // No Error first, summaries on the assertions: a
        // skipped generation says why in the reports.
        Assert.IsFalse(errors.Any(e => e.TheError.Severity == ErrorSeverity.Error), ReportSummary(errors));
        RequireLayer(fx, "NDMFComponent [Target]", ReportSummary(errors));
        AssertLayerAbsent(fx, "NDMFFailingComponent [FailTarget]", ReportSummary(errors));
        Assert.IsTrue(errors.Any(e => e.TheError.Severity == ErrorSeverity.NonFatal), ReportSummary(errors));
    }

    // PASS-07: an out-of-range Level reaches the generated transition clamped to its range.
    [Test]
    public void OutOfRangeLevelIsClamped()
    {
        var root = CreateMinimalAvatar();
        AddToggle(root, CreateChild(root, "Target")).Level = 1.5f;

        var built = Build(root, out _);
        var layer = RequireLayer(FindFxController(built), "NDMFComponent [Target]");
        var idle = RequireState(layer, "Idle");
        var forward = idle.transitions.Single(t => t.destinationState.name == "Active");

        Assert.AreEqual(1f, forward.conditions[0].threshold);
    }
}
```
