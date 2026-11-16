# Template: Installing a generated menu by decomposition

## When it applies

A generated menu is installed through Modular Avatar components,
and the product cannot assume an author-placed installer —
including the case where a build pass stores the bundle itself.
Requiring the author to add an installer instead breaks setups
where the component already sits inside an existing menu
structure; decomposing the menu into Menu Items places it where
the component is, and the decomposition supplies the installer
that makes Modular Avatar visit them.

## Requirements

- Menu installation is entered through a
  ModularAvatarMenuInstaller: with no installer anywhere on the
  avatar, menu components are not visited at all. Every route
  therefore ends with an installer on the target GameObject.
- If the GameObject has a ModularAvatarMenuInstaller, the
  generated VRCExpressionsMenu asset — or its parent menu, for a
  menu that has one — is assigned as the installer's menu to
  append.
- Otherwise, the menu is decomposed into
  ModularAvatarMenuItem components whose Control fields are deep
  copies of the generated menu's controls, and an installer with
  no menu to append is placed on the same GameObject: the
  installer makes Modular Avatar visit the menu source — the
  items, or the group — on its GameObject.
  - A single-control menu becomes one Menu Item on the GameObject
    itself.
  - A menu with a parent makes the GameObject's own Menu Item a
    submenu over children: its control is the parent's single
    submenu control, its menu source is Children, and each
    control becomes a child GameObject with its own Menu Item.
  - A multi-control menu without a parent puts a
    ModularAvatarMenuGroup on the GameObject and one child Menu
    Item per control.
- The inspector mirrors this logic when advising the author: a
  component placed under a Menu Item submenu, a Menu Group, or
  another menu generator is visited through that structure and
  needs no installer of its own; anywhere else, the generated
  installation supplies the installer, so the author adds
  nothing.

## Skeleton

```csharp
// [Editor]
// Installation: the asset route when an installer is present,
// decomposition otherwise. The decomposition covers the three
// cases: single control, submenu over children, and group.
void InstallMenu(GameObject target, Bundle bundle)
{
    if (target.TryGetComponent<ModularAvatarMenuInstaller>(out var installer))
    {
        installer.menuToAppend = bundle.ParentMenu ?? bundle.Menu;
        return;
    }
    // The decomposition route still ends with an installer:
    // with nothing to append, it makes Modular Avatar visit
    // the items and the group on this GameObject.
    if (target.GetComponent<ModularAvatarMenuInstaller>() == null)
        target.AddComponent<ModularAvatarMenuInstaller>();
    if (bundle.ParentMenu == null && bundle.Menu.controls.Count > 1)
    {
        if (target.GetComponent<ModularAvatarMenuGroup>() == null)
            target.AddComponent<ModularAvatarMenuGroup>();
        foreach (var control in bundle.Menu.controls) AddChildItem(target, control);
        return;
    }
    var item = target.GetComponent<ModularAvatarMenuItem>()
        ?? target.AddComponent<ModularAvatarMenuItem>();
    item.Control = CopyControl((bundle.ParentMenu ?? bundle.Menu).controls[0]);
    if (bundle.ParentMenu != null)
    {
        item.MenuSource = SubmenuSource.Children;
        foreach (var control in bundle.Menu.controls) AddChildItem(target, control);
    }
}

void AddChildItem(GameObject parent, VRCExpressionsMenu.Control control)
{
    var child = new GameObject(control.name);
    child.transform.SetParent(parent.transform);
    child.AddComponent<ModularAvatarMenuItem>().Control = CopyControl(control);
}

// The copy is the product's own helper: Control has no copy
// method in the SDK or the frameworks.
VRCExpressionsMenu.Control CopyControl(VRCExpressionsMenu.Control source)
{
    return new VRCExpressionsMenu.Control
    {
        name = source.name,
        icon = source.icon,
        type = source.type,
        parameter = source.parameter == null ? null
            : new VRCExpressionsMenu.Control.Parameter { name = source.parameter.name },
        value = source.value,
        style = source.style,
        subMenu = source.subMenu,
        subParameters = source.subParameters?.ToArray(),
        labels = source.labels?.ToArray(),
    };
}
```
