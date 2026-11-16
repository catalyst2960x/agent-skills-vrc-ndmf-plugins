# Template: Platform variant cases

## When it applies

A plugin that runs on more than the VRChat platform carries
these cases: a feature whose mechanism differs per platform
shows the designed form on each platform — the difference
itself is the case — and a platform-filtered component is
effective on its platform and disabled off it. A plugin
scoped to VRChat avatars only does not carry this file.

The samples run against document-local stand-ins:
`NDMFPlatformComponent`, whose contract is animator-driven
on a VRChat avatar (it generates its layer) and statically
applied on a generic avatar (it sets its target's active
state on the built root directly, generating no layer); and
`NDMFVrchatOnlyComponent`, filtered to the VRChat platform,
which generates its layer there and, off the platform, is
skipped with an Information report. The generic fixture
marks the root with the framework's own avatar root
component instead of the VRChat descriptor; which root a
fixture carries is the platform switch.

## Skeleton

```csharp
// [Editor: unit tests]
// Stand-ins to implement (document-local; contracts in the introduction): NDMFPlatformComponent, NDMFVrchatOnlyComponent
using System.Linq;
using nadena.dev.ndmf;
using NUnit.Framework;
using UnityEngine;

public class PlatformVariantTests : PluginTestBase
{
    GameObject CreateGenericAvatar()
    {
        var root = Track(new GameObject("GenericAvatar"));
        root.AddComponent<Animator>();
        root.AddComponent<NDMFAvatarRoot>();
        return root;
    }

    // BASE-03 (the test base) with PLAT-01 (platform and runtime): a layer on VRChat, a static application and no layer on generic.
    [Test]
    public void MechanismTakesItsPlatformForm()
    {
        var vrchat = CreateMinimalAvatar();
        var vrchatTarget = CreateChild(vrchat, "Target");
        vrchatTarget.SetActive(false);
        CreateChild(vrchat, "Platform").AddComponent<NDMFPlatformComponent>().Target = vrchatTarget;

        var generic = CreateGenericAvatar();
        var genericTarget = CreateChild(generic, "Target");
        genericTarget.SetActive(false);
        CreateChild(generic, "Platform").AddComponent<NDMFPlatformComponent>().Target = genericTarget;

        var builtVrchat = Build(vrchat, out _);
        RequireLayer(FindFxController(builtVrchat), "NDMFPlatformComponent [Target]");

        var builtGeneric = Build(generic, out _);
        Assert.IsTrue(builtGeneric.transform.Find("Target").gameObject.activeSelf);
    }

    // PLAT-02: off its platform the component is skipped with an Information report and no output.
    [Test]
    public void PlatformFilterDisablesOffPlatform()
    {
        var vrchat = CreateMinimalAvatar();
        CreateChild(vrchat, "Filtered").AddComponent<NDMFVrchatOnlyComponent>().Target =
            CreateChild(vrchat, "Target");

        var generic = CreateGenericAvatar();
        var genericTarget = CreateChild(generic, "Target");
        genericTarget.SetActive(false);
        CreateChild(generic, "Filtered").AddComponent<NDMFVrchatOnlyComponent>().Target = genericTarget;

        var builtVrchat = Build(vrchat, out var vrchatErrors);
        RequireLayer(FindFxController(builtVrchat), "NDMFVrchatOnlyComponent [Target]");
        Assert.IsFalse(vrchatErrors.Any(e => e.TheError.Severity == ErrorSeverity.Information));

        var builtGeneric = Build(generic, out var genericErrors);
        Assert.IsTrue(genericErrors.Any(e => e.TheError.Severity == ErrorSeverity.Information));
        Assert.IsFalse(genericErrors.Any(e => e.TheError.Severity == ErrorSeverity.Error));
        Assert.IsFalse(builtGeneric.transform.Find("Target").gameObject.activeSelf);
    }
}
```
