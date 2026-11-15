# Template: Installing a generated menu without an installer

## When it applies

A generated menu is installed through Modular Avatar components,
and no ModularAvatarMenuInstaller is present on the GameObject —
including the case where a build pass stores the bundle itself
and cannot assume an installer. Requiring the author to add an
installer instead breaks setups where the component already sits
inside an existing menu structure; decomposing the menu into
Menu Items places it where the component is.

## Requirements

- If the GameObject has a ModularAvatarMenuInstaller, the
  generated VRCExpressionsMenu asset — or its parent menu, for a
  menu that has one — is assigned as the installer's menu to
  append.
- Without an installer, the menu is decomposed into
  ModularAvatarMenuItem components whose Control fields are deep
  copies of the generated menu's controls:
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
  another menu generator does not need an installer; anywhere
  else without an installer draws a warning that no menu will be
  installed.

## Skeleton

```csharp
// [Editor]
// Decomposition when no installer is present. This block covers the
// three cases: single control, submenu over children, and group.
void InstallMenu(GameObject target, Bundle bundle)
{
    if (target.TryGetComponent<ModularAvatarMenuInstaller>(out var installer))
    {
        installer.menuToAppend = bundle.ParentMenu ?? bundle.Menu;
        return;
    }
    if (bundle.ParentMenu == null && bundle.Menu.controls.Count > 1)
    {
        target.GetOrAddComponent<ModularAvatarMenuGroup>();
        foreach (var control in bundle.Menu.controls) AddChildItem(target, control);
        return;
    }
    var item = target.GetOrAddComponent<ModularAvatarMenuItem>();
    item.Control = (bundle.ParentMenu ?? bundle.Menu).controls[0].DeepCopy();
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
    child.AddComponent<ModularAvatarMenuItem>().Control = control.DeepCopy();
}
```
