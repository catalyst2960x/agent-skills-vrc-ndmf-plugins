# Template: A display-only slider (a parameter meter)

## When it applies

A value already lives in an Animator parameter whose owner is
not the player — a parameter a PhysBone or a Contact derives,
a built-in the platform drives, or a value the avatar's own
layers compute — and the author wants that value visible in
the menu as a slider wheel. A Radial Puppet control has no
read-only form: it displays the parameter it binds and writes
it when dragged. Display-only behaviour is therefore never a
property of the control. It is a property of who owns the
parameter's value: when something rewrites the value
continuously, a write from the menu does not persist, and the
wheel is a meter.

Which form a slider takes is fixed by which parameter the
wheel binds and who writes it: the bound parameter has one
effective writer. If the writer is the player alone, the
wheel is an input slider; if it is a system — the component
that derives the parameter, or a layer generated to copy
into it — the wheel is a meter. This template builds the
meter. The input form is its mirror image: the bound
parameter and the driven value are separate parameters,
and the driven side is reached by reading the input (a
motion-time scrub, a parameter-driver copy, a blend), so
an external change never touches the player's value.

## Requirements

- The wheel is an ordinary Radial Puppet control whose single
  sub-parameter is the displayed parameter's name. Emission
  follows the menu forms (an item component, or an installer
  assignment); nothing about the control is special.
- The emission also places a menu installer on the host,
  carrying no menu to append. Menu processing enters only
  through installers: an item with no installer anywhere on
  the avatar is never installed into the built menu, so the
  item alone does not reach the menu.
- The displayed name is declared as an expression parameter
  (Float), so the menu can bind it. When the value is derived
  on each client — the usual case for a meter — the
  declaration is local-only: syncing a derived value would pay
  for the copy and fight its derivation.
- One of two ownership forms supplies the continuous writer:
  - **Platform-derived parameter.** The displayed name is the
    derived parameter's own name (a PhysBone's or Contact's
    generated parameter). The platform component recomputes
    the value as its inputs change, which is the continuous
    write. The declaration must sit where the name reaches
    the built avatar unchanged — a renaming scope that
    rewrites the name breaks the identity with the platform
    parameter, so the straightforward placement is outside
    any renaming scope, for example on the avatar root.
  - **Copy layer.** For any other Float source, including a
    built-in, the plugin owns a display parameter and a layer
    that copies the source into it every frame: a 1D blend
    tree whose blend parameter is the source, with two
    children at thresholds 0 and 1 whose clips write the
    display parameter as the constants 0 and 1. The tree
    interpolates between the constants by the source's value,
    so its output is the source's value, written continuously
    at every source value including 0. The source is declared
    in the generated controller with its own type, or Float
    when the type is unknown, so the tree can read it.
- The meter requires the shared form: the bound parameter
  and the driven value are one parameter, and the system
  side holds the exclusive write. A design that needs the
  wheel to stay operable uses the separated form described
  above and is outside this template.
- If nothing rewrites the displayed parameter, the wheel is
  an ordinary operable slider. The template's guarantee is
  the continuous writer, and a form without one must not be
  presented as display-only.

## Skeleton

```csharp
// [Editor: generator]
// The copy layer: a 1D blend tree over the source parameter
// whose two children write the display parameter as the
// constants 0 and 1, so the display parameter equals the
// source every frame.
AnimatorController BuildMeterController(string name, string sourceParameter, string displayParameter)
{
    var controller = new AnimatorController();
    controller.AddParameter(sourceParameter, AnimatorControllerParameterType.Float);
    controller.AddParameter(displayParameter, AnimatorControllerParameterType.Float);
    controller.AddLayer(name);
    var layers = controller.layers;
    layers[0].defaultWeight = 1f;
    controller.layers = layers;
    var layer = layers[0];

    // Two constant clips writing the display parameter itself
    // (an Animator binding), at 0 and at 1.
    AnimationClip ConstantParameterClip(string clipName, string parameter, float value)
    {
        var clip = new AnimationClip { name = clipName };
        clip.SetCurve("", typeof(Animator), parameter,
            new AnimationCurve(new Keyframe(0f, value)));
        return clip;
    }

    var tree = new BlendTree
    {
        name = name,
        blendType = BlendTreeType.Simple1D,
        blendParameter = sourceParameter,
        useAutomaticThresholds = false,
    };
    tree.AddChild(ConstantParameterClip(name + " 0", displayParameter, 0f), 0f);
    tree.AddChild(ConstantParameterClip(name + " 1", displayParameter, 1f), 1f);

    var state = layer.stateMachine.AddState(name);
    state.motion = tree;
    state.writeDefaultValues = false;
    layer.stateMachine.defaultState = state;
    return controller;
}
```

```csharp
// [Editor: NDMF pass]
// The meter's emission on the component's own GameObject:
// the parameter declaration that makes the displayed name
// bindable, the merge of the copy layer (copy form only),
// the Radial Puppet item, and the installer that gets the
// item installed. Four features in one block, annotated in
// the code: declaration, merge, item, installer.
void EmitMeter(GameObject host, string displayParameter, AnimatorController copyController)
{
    // Feature 1: declare the displayed parameter, local-only,
    // unsaved — a derived value has nothing to save.
    var parameters = host.AddComponent<ModularAvatarParameters>();
    parameters.parameters.Add(new ParameterConfig
    {
        nameOrPrefix = displayParameter,
        syncType = ParameterSyncType.Float,
        localOnly = true,
        saved = false,
        defaultValue = 0f,
        hasExplicitDefaultValue = true,
    });

    // Feature 2 (copy form only): merge the copy layer that
    // keeps the display parameter equal to its source.
    if (copyController != null)
    {
        var merge = host.AddComponent<ModularAvatarMergeAnimator>();
        merge.animator = copyController;
        merge.layerType = VRCAvatarDescriptor.AnimLayerType.FX;
        merge.matchAvatarWriteDefaults = true;
    }

    // Feature 3: the wheel itself — an ordinary Radial Puppet
    // item bound to the displayed parameter.
    var item = host.AddComponent<ModularAvatarMenuItem>();
    item.Control = new VRCExpressionsMenu.Control
    {
        name = displayParameter,
        type = VRCExpressionsMenu.Control.ControlType.RadialPuppet,
        parameter = new VRCExpressionsMenu.Control.Parameter { name = "" },
        value = 0f,
        subParameters = new[]
        {
            new VRCExpressionsMenu.Control.Parameter { name = displayParameter },
        },
        labels = new VRCExpressionsMenu.Control.Label[0],
    };

    // Feature 4: the installer. Menu processing enters only
    // through installers, so the item above reaches the built
    // menu only because an installer sits on the avatar; this
    // one is on the same GameObject as the item and carries
    // no menu to append.
    host.AddComponent<ModularAvatarMenuInstaller>();
}
```
