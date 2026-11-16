# Template: Error reporting cases

## When it applies

Every plugin carries the severity cases. Most of them ride
inside the component-processing cases — an unset input
reporting Information, an invalid value reporting NonFatal —
because the severity is asserted on the same build that
asserts the generation. This file carries the case that
stands alone: contention between components. The contract
throughout is that a case asserts severities, never message
text, and that no case produces an Error from the plugin.

The samples run against the test base and the schematic
`NDMFComponent`, whose designed contention rule is: two
components on the same target object — the first in
processing order generates, the other reports NonFatal and
generates nothing. This is distinct from the same layer name
on different objects, which is numbering (generated output).
The component reports its problems with itself attached as
a context object, which the build report window shows; the
captured context a test receives exposes the reporting
plugin and the pass name, and does not hand the context
objects back, so a case identifies a report by plugin and
pass. The schematic plugin's qualified name is `NDMFPlugin`,
and its generating pass reports under the name `Generate`.

## Skeleton

```csharp
// [Editor: unit tests]
// Stand-ins to implement (document-local; contracts in the introduction): NDMFComponent
using System.Linq;
using nadena.dev.ndmf;
using NUnit.Framework;
using UnityEngine;

public class ErrorReportingTests : PluginTestBase
{
    // ERR-02 (error reporting): a report is identifiable by its plugin and pass.
    [Test]
    public void ReportsIdentifyTheirPluginAndPass()
    {
        var root = CreateMinimalAvatar();
        CreateChild(root, "Toggle").AddComponent<NDMFComponent>();

        Build(root, out var errors);

        var report = errors.Single(e => e.TheError.Severity == ErrorSeverity.Information);
        Assert.AreEqual("NDMFPlugin", report.Plugin.QualifiedName);
        Assert.AreEqual("Generate", report.PassName);
    }

    // ERR-04: two components with the same problem produce one report each.
    [Test]
    public void ReportCountsMatchProblemCounts()
    {
        var root = CreateMinimalAvatar();
        CreateChild(root, "Toggle").AddComponent<NDMFComponent>();
        CreateChild(root, "Toggle").AddComponent<NDMFComponent>();

        Build(root, out var errors);

        Assert.AreEqual(2, errors.Count(e => e.TheError.Severity == ErrorSeverity.Information), ReportSummary(errors));
    }

    // ERR-03: two components on one target — the first generates, the second reports NonFatal only.
    [Test]
    public void SameTargetContentionReportsNonFatal()
    {
        var root = CreateMinimalAvatar();
        var target = CreateChild(root, "Target");
        CreateChild(root, "Toggle").AddComponent<NDMFComponent>().Target = target;
        CreateChild(root, "Toggle").AddComponent<NDMFComponent>().Target = target;

        var built = Build(root, out var errors);
        var fx = FindFxController(built);

        RequireLayer(fx, "NDMFComponent [Target]");
        AssertLayerAbsent(fx, "NDMFComponent [Target] (2)");
        Assert.IsTrue(errors.Any(e => e.TheError.Severity == ErrorSeverity.NonFatal));
    }
}
```
