# Template: Menu generation cases

## When it applies

A plugin that emits menus carries these cases: the menu
installs by the designed route — into an installer when one
is present on the avatar, decomposed onto menu items when
none is; cyclic menu references terminate with the designed
result; deduplication applies per installation, so
installing the same menu twice is two installations; and
the generated structure — a submenu's child controls, a
control installed under a group — is present in the built
menu as designed.

The assertions read the built avatar's expressions menu —
the descriptor's `expressionsMenu` asset — never the menu
components. Modular Avatar resolves installers and items
into that asset during the build, so installed items do
not remain on the built avatar as components to count; a
case that counts them asserts a state the framework does
not produce. Both installation routes converge on the one
menu asset, and the cases assert the controls in it: name,
type, parameter, value, and sub-menu.

The samples run against a document-local stand-in,
`NDMFMenuComponent`, whose contract the intro states: it
carries one toggle control named after its target and
installs it into the avatar's expressions menu, through
an installer's menu to append when the avatar carries an
installer, and otherwise as its own menu item, visited by
an installer the stand-in places with nothing to append.
Menu components are visited only through an installer, so
an item with no installer anywhere on the avatar is never
installed. A component can name another component's
group as its installation target; following those
references, a cycle is cut at the second visit to a group,
and the control installs at the first group of the chain.
With child entries set, the stand-in installs one submenu
control carrying them as its controls, with their names
and values. With a group object named, the control
installs into the sub-menu the group forms, which carries
the group's name.

## Skeleton

```csharp
// [Editor: unit tests]
// Stand-ins to implement (document-local; contracts in the introduction): NDMFMenuComponent
using System.Collections.Generic;
using System.Linq;
using nadena.dev.modular_avatar.core;
using NUnit.Framework;
using UnityEngine;
using VRC.SDK3.Avatars.Components;
using VRC.SDK3.Avatars.ScriptableObjects;

public class MenuGenerationTests : PluginTestBase
{
    static VRCExpressionsMenu BuiltMenu(GameObject built)
        => built.GetComponent<VRCAvatarDescriptor>().expressionsMenu;

    static IEnumerable<VRCExpressionsMenu.Control> AllControls(VRCExpressionsMenu menu)
    {
        if (menu == null) yield break;
        foreach (var control in menu.controls)
        {
            yield return control;
            if (control.subMenu == null) continue;
            foreach (var child in AllControls(control.subMenu)) yield return child;
        }
    }

    // MENU-01 (menus): the route follows the avatar — through the installer when present, as the component's own item when not.
    [Test]
    public void InstallationFollowsTheDesignedRoute()
    {
        var withInstaller = CreateMinimalAvatar();
        CreateChild(withInstaller, "Installer").AddComponent<ModularAvatarMenuInstaller>();
        CreateChild(withInstaller, "Menu").AddComponent<NDMFMenuComponent>().Target =
            CreateChild(withInstaller, "Target");

        var installed = AllControls(BuiltMenu(Build(withInstaller, out _)))
            .Where(c => c.name == "Target").ToArray();
        Assert.AreEqual(1, installed.Length);
        Assert.AreEqual(VRCExpressionsMenu.Control.ControlType.Toggle, installed[0].type);
        Assert.AreEqual("NDMFComponent/Active", installed[0].parameter.name);

        var withoutInstaller = CreateMinimalAvatar();
        CreateChild(withoutInstaller, "Menu").AddComponent<NDMFMenuComponent>().Target =
            CreateChild(withoutInstaller, "Target");

        var own = AllControls(BuiltMenu(Build(withoutInstaller, out _)))
            .Where(c => c.name == "Target").ToArray();
        Assert.AreEqual(1, own.Length);
        Assert.AreEqual("NDMFComponent/Active", own[0].parameter.name);
    }

    // MENU-02: an installation cycle terminates; the control is installed once, at the chain's first group.
    [Test]
    public void CyclicGroupsTerminate()
    {
        var root = CreateMinimalAvatar();
        var target = CreateChild(root, "Target");
        var a = CreateChild(root, "MenuA").AddComponent<NDMFMenuComponent>();
        var b = CreateChild(root, "MenuB").AddComponent<NDMFMenuComponent>();
        a.Target = target;
        b.Target = target;
        a.InstallInto = b;
        b.InstallInto = a;

        var built = Build(root, out var errors);
        var installed = AllControls(BuiltMenu(built)).Count(c => c.name == "Target");

        Assert.AreEqual(1, installed);
        Assert.IsFalse(errors.Any(e => e.TheError.Severity == nadena.dev.ndmf.ErrorSeverity.Error),
            ReportSummary(errors));
    }

    // MENU-03: deduplication is per installation — the same control from two components installs twice.
    [Test]
    public void DeduplicationIsPerInstallation()
    {
        var root = CreateMinimalAvatar();
        CreateChild(root, "Menu").AddComponent<NDMFMenuComponent>().Target = CreateChild(root, "Target");
        CreateChild(root, "Menu").AddComponent<NDMFMenuComponent>().Target = CreateChild(root, "Target");

        var built = Build(root, out _);
        var count = AllControls(BuiltMenu(built)).Count(c => c.name == "Target");

        Assert.AreEqual(2, count);
    }

    // MENU-04: a generated submenu installs as one submenu control carrying the designed children.
    [Test]
    public void SubmenuCarriesItsChildren()
    {
        var root = CreateMinimalAvatar();
        var menu = CreateChild(root, "Menu").AddComponent<NDMFMenuComponent>();
        menu.Target = CreateChild(root, "Target");
        menu.Children = new[]
        {
            new NDMFMenuComponent.Child { Name = "First", Value = 1f },
            new NDMFMenuComponent.Child { Name = "Second", Value = 2f },
        };

        var parent = AllControls(BuiltMenu(Build(root, out _)))
            .Single(c => c.type == VRCExpressionsMenu.Control.ControlType.SubMenu);
        Assert.NotNull(parent.subMenu);
        var children = parent.subMenu.controls;
        Assert.AreEqual(2, children.Count);
        Assert.AreEqual("First", children[0].name);
        Assert.AreEqual(1f, children[0].value);
        Assert.AreEqual("Second", children[1].name);
        Assert.AreEqual(2f, children[1].value);
    }

    // MENU-05: a control installed into a group lands in the sub-menu the group forms.
    [Test]
    public void GroupInstallationLandsUnderTheGroup()
    {
        var root = CreateMinimalAvatar();
        var menu = CreateChild(root, "Menu").AddComponent<NDMFMenuComponent>();
        menu.Target = CreateChild(root, "Target");
        menu.Group = CreateChild(root, "Group");

        var groupControl = AllControls(BuiltMenu(Build(root, out _)))
            .Single(c => c.type == VRCExpressionsMenu.Control.ControlType.SubMenu
                         && c.name == "Group");
        var landed = groupControl.subMenu.controls.Where(c => c.name == "Target").ToArray();
        Assert.AreEqual(1, landed.Length);
    }
}
```
