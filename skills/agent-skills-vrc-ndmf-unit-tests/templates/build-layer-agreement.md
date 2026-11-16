# Template: Build layer agreement case

## When it applies

Every plugin carries this case once: the same representative
configuration run at the two layers of test — one pass in a
direct BuildContext, and the full pipeline — produces the
same generated output. A divergence between the layers is
itself the finding: ordering or another pass's effect is
changing the result, and the case separates that from a
defect in the pass. Without it, a suite of pass-level cases
can pass while the integrated build differs.

The sample runs against the test base and the schematic
`NDMFComponent`; `NDMFPass` stands for the plugin's
generating pass, invoked directly for the pass-level run.

## Skeleton

```csharp
// [Editor: unit tests]
// Stand-ins to implement (document-local; contracts in the introduction): NDMFComponent, NDMFPass
using System.Linq;
using nadena.dev.modular_avatar.core;
using nadena.dev.ndmf;
using nadena.dev.ndmf.animator;
using NUnit.Framework;
using UnityEditor.Animations;
using UnityEngine;

public class BuildLayerAgreementTests : PluginTestBase
{
    // BASE-04 (the test base): the pass alone and the full pipeline generate the same layer set.
    [Test]
    public void PassAndPipelineAgree()
    {
        var root = CreateMinimalAvatar();
        var target = CreateChild(root, "Target");
        var holder = CreateChild(root, "Toggle");
        holder.AddComponent<NDMFComponent>().Target = target;

        var ctx = CreateContext(root);
        ctx.ActivateExtensionContext<AnimatorServicesContext>();
        ErrorReport.CaptureErrors(() => NDMFPass.Execute(ctx));
        var merge = ctx.AvatarRootObject
            .GetComponentInChildren<ModularAvatarMergeAnimator>(true);
        var direct = merge.animator as AnimatorController;

        var built = Build(root, out _);
        var pipeline = FindFxController(built);

        CollectionAssert.AreEquivalent(
            direct.layers.Select(l => l.name),
            pipeline.layers.Select(l => l.name));
    }
}
```
