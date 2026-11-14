# Expression Parameters and Expressions Menu Reference

Reading and writing the two avatar-level surfaces a plugin shares with
every other plugin: the Expression Parameters and the Expressions Menu.
Facts are those of the SDK types and the pinned framework versions
(SKILL.md, Sources).

## Reading parameters

- **Build time: ParameterInfo is the only complete view** (Parameters).
  Parameters declared only in a ModularAvatarParameters component are
  visible through it in Generating, so a plugin can validate conditions
  against them before MA itself runs.
- **Supplying parameters before they exist.** A component whose parameters
  are only created during generation is invisible to validation — its own
  and other plugins' — unless it also supplies them ahead of time. NDMF's
  mechanism for that is a parameter provider: a class marked
  `[ParameterProviderFor(typeof(TheComponent))]` implementing
  `IParameterProvider` (both in `nadena.dev.ndmf`), whose
  GetSuppliedParameters returns one ProvidedParameter per parameter (name,
  the Animator parameter namespace, the component instance, the plugin
  instance, and the animator parameter type). Menu and toggle generators
  use exactly this so their not-yet-generated parameters can be validated
  and referenced in Generating like any other parameter. If a product's
  component introduces parameters, give it a provider.
- **Edit time: the descriptor's assets.** Tools that inspect or migrate an
  avatar outside a build read `expressionParameters` and `expressionsMenu`
  from the VRCAvatarDescriptor (VRChatAvatarDescriptor() on the build
  context, or the component directly in edit mode). What they read is the
  author's current assets only — not what any plugin will add — which is
  why this view is for tooling, never for build decisions (SKILL.md
  NEVER #2).

## Writing parameters

- **Dependency case 2: declare with ModularAvatarParameters.** Each entry
  is a ParameterConfig. Its syncType is the synced value type —
  NotSynced, Int, Float, or Bool — and a NotSynced entry is not
  registered in the Expression Parameters at all. Three entry kinds
  matter to a generator: a normal entry; an Animator Only entry
  (internalParameter), which is renamed like the others but never
  registered, for work parameters; and a Prefix entry (isPrefix), where
  one entry configures the whole family of parameters a PhysBone or a
  Raycast generates from that prefix, and is likewise not registered.
- **Entry settings beyond the basics.** A synced Int or Float costs its
  bitWidth in network bits (2 to 8, default 8), and its representation
  (Unsigned or Signed) fixes how the value maps onto that width; a
  custom lerp duration is what smooths a synced value — smoothing is a
  setting on the entry, not a sync type. A default value counts only
  when set explicitly: an entry without one inherits the Expression
  Parameters asset's value, or zero. Override Animator Defaults
  replaces the defaults inside animator controllers with the declared
  one (always applied for an Animator Only entry that has a default).
- **Renaming is scoped.** remapTo renames the parameter for everything
  outside the Parameters component's own subtree; references inside the
  subtree keep the original name. Parameters components nest, each
  scoping its renames to its own subtree — so a generator places the
  component on its own component's GameObject, or on the subtree that
  uses the original names.
- **Merging.** MA merges all declarations and resolves collisions: the
  same name declared with the same type merges into one parameter; the
  same name with a different type is a build error the author must fix.
  Between non-nested declarations of one name, the parameter is saved
  if any of them says so, and conflicting non-blank default values
  produce a warning. The order declarations merge in matters beyond
  this file: it is part of sync compatibility (VRChat Avatar
  Platform, Synchronization).
- **Budget.** Synced parameters share a 256-bit budget across the whole
  avatar (a Bool costs 1 bit; a synced Int or Float costs its bitWidth,
  8 by default — a smaller width for a small-range value is the
  standard way to spend less). A declaration is a claim
  on a scarce shared resource: prefer Bool over Int for two-state values,
  and mark work parameters localOnly or unsynced when remote players do
  not need them — local-only parameters cost no sync budget. A product
  that declares many parameters should say so in its documentation.
- **Dependency case 1: no declaration service exists.** The plugin
  registers the parameters it needs on the controllers it generates and
  validates them against ParameterInfo, but nothing merges them into the
  avatar's Expression Parameters, and nothing stops a name collision with
  a parameter the author uses for something else. Prefix generated names
  with the product's name, and treat a collision between a generated name
  and a ParameterInfo entry of a different type as a validation error:
  the plugin is about to rely on that name with its own type. Duplicates
  among the provided entries themselves are not an error (Parameters).
- **Naming is a compatibility surface.** Once authors build content on a
  parameter name, renaming it breaks their content. Decide names up front
  (SKILL.md, Naming) and keep them stable across versions.

## Reading menus

- **Build time: read the MA component graph, not the menu asset.** Where
  menus will land is visible from the ModularAvatarMenuItem hierarchy and
  the ModularAvatarMenuInstallers already on the avatar. The avatar's own
  Expressions Menu asset belongs to the author; passes do not walk it to
  decide placement.
- **Edit time: the menu is a tree of assets.** A VRCExpressionsMenu holds
  up to 8 controls; a SubMenu control points at another VRCExpressionsMenu
  asset. Tools that read menus (inspectors, migrators) traverse controls
  recursively and must guard against missing submenu assets and cycles of
  their own making. A control carries a name, an icon, a type, a bound
  parameter with a value, and sub-parameters (below).

## Writing menus (dependency case 2)

Two routes, both consumed by MA at build time:

| Route | Shape | Fits when |
| --- | --- | --- |
| Menu asset + installer | Generate a VRCExpressionsMenu asset (and one asset per submenu), then add a ModularAvatarMenuInstaller whose menuToAppend is that asset. Set installTargetMenu to nest into a specific existing menu; leave it empty to append to the avatar's root menu | The menu is large, shared between products, or also useful as a standalone asset |
| Menu item hierarchy | One GameObject per control carrying a ModularAvatarMenuItem with the control's data; a SubMenu item takes its entries from its child items (its submenu source is its children) or from a referenced menu asset | The menu mirrors the component's own hierarchy and should stay visible and editable in the scene |

- A generated menu asset needs a stored home: create it inside an asset
  container the pass owns (a generated-assets folder for the product), and
  recreate it each build rather than editing the stored one.
- Control types and what they bind:

| Control | Behaviour | Parameter binding |
| --- | --- | --- |
| Button | Sets the value while pressed | One parameter + the value to set |
| Toggle | Latches the value on each press | One parameter + the value for the on state |
| SubMenu | Opens a submenu; can also set a parameter when opened | Optional parameter + value, plus the submenu itself |
| RadialPuppet | A dial the player drags | One float sub-parameter receiving 0..1 |
| TwoAxisPuppet | A 2D pad | Two float sub-parameters (horizontal, vertical) |
| FourAxisPuppet | Two 2D pads combined | Four float sub-parameters |

- What a MenuItem carries: a complete VRCExpressionsMenu.Control (the
  item's Control field — name, icon, type, parameter, value,
  sub-parameters, labels), a MenuSource — a referenced menu asset, the
  item's child items, or another GameObject whose children form the
  submenu — and a label: an empty label means the GameObject's name
  becomes the menu label, which is why the hierarchy route names its
  GameObjects for the menu. A ModularAvatarMenuGroup instead pulls
  every item under a target object into the menu at the group's
  position.
- **Automatic parameter creation.** A MenuItem whose parameter is
  declared nowhere gets that parameter created by MA. The item's flags
  then configure the created parameter: isSynced (default true) and
  isSaved (default true); isDefault marks which item's value is the
  parameter's default when several items share one parameter (the last
  in hierarchy order wins); and automaticValue allocates values
  sequentially in hierarchy order — the standard way to build a group
  of toggles over one Int parameter. A generator that relies on
  automatic creation must set these flags deliberately; declaring the
  parameter explicitly (P-13) is the controlled route.
- Puppet controls bind their axes through the control's sub-parameters,
  not its main parameter. A radial-driven effect often needs a companion
  parameter marking "the value is being changed" so other states can
  react while the player drags; whether to add one is a product design
  decision, but if it exists it belongs to the same declaration and
  naming scheme as the value parameter.
- Menus are per-avatar and shared: appending is the default posture.
  Replacing or clearing the author's menu from a plugin is destructive
  in the same sense as NEVER #9 and is not a plugin behaviour.
- **Menus in case 1 are outside this skill's scope.** With NDMF alone
  there is no installer and no menu merge, so every menu route in
  this skill runs through dependency case 2. Editing the author's
  menu asset directly would work mechanically; this skill does not
  cover it.

## Components that add parameters and menu items

The menu-generator product shape — an existing menu generator is
built exactly this way — assembles the pieces above into one
component:

- The component is editor-only and holds only the definition: which
  parameters, which controls, bound to what. It generates nothing
  itself.
- A parameter provider (Reading parameters) supplies the component's
  parameters, so validation — its own and other plugins' — sees them
  in Generating before anything is emitted.
- A pass in Generating, ordered before Modular Avatar, emits per
  component: one ModularAvatarParameters on the component's own
  GameObject, declaring the parameters the controls bind; and the
  MenuItem hierarchy as child GameObjects of that GameObject. The
  placement is load-bearing twice over: it puts the component and its
  items inside the Parameters component's rename scope (Writing
  parameters); and MA collects MenuItems from anywhere on the avatar,
  so nesting under the component is what keeps the product's items
  grouped, ordered, and editable as one unit (nesting under a SubMenu
  item forms the submenu).
- The parameters the items bind are the ones the same emission
  declares. Relying on the items' automatic parameter creation
  instead leaves the created parameters' sync and saved settings to
  the item flags — a deliberate choice, not a default to drift into.
- The build pass skips a component whose GameObject already carries
  the generated components (the presence check of Edit-time
  pre-generation, below), which is what makes the two routes compose.
- Case 2 only: with no MA there is nothing to emit into, and case 1
  menus are out of scope (above).

## Edit-time pre-generation

A product may let the author generate at edit time (an inspector action)
instead of waiting for the build. The pattern that keeps both routes
consistent: generation writes the same MA components the build pass would
write (parameters, menu items or installer, merge animator), and the
build pass checks whether those components are already present on the
component's GameObject and skips generation for that component when they
are. Without the presence check, the build generates a second copy.
