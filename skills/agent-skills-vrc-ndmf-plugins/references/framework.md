# NDMF Framework Reference

Facts are those of NDMF 1.15.* (package nadena.dev.ndmf). Re-verify
against the installed version before relying on a signature: its API
surface moves between versions.

## Plugin discovery and identity

- A plugin is published with an assembly attribute:
  `[assembly: ExportsPlugin(typeof(MyPlugin))]`.
- The plugin class derives from `Plugin<T>` where T is the class itself, and
  implements `protected override void Configure()`.
- `QualifiedName` defaults to `typeof(T).FullName` (the override decision
  is SKILL.md, Naming) and `DisplayName` defaults to the QualifiedName.
  Other plugins order against the QualifiedName string.
- `Configure()` is the only place `InPhase(...)` may be called; NDMF throws
  if it is called elsewhere.

## Phases and ordering

Built-in phases run in this order:

| Phase | Intended use (per the NDMF source comments) |
| --- | --- |
| FirstChance | Runs before platform initialization, for plugins that must run before everything else; EditorOnly objects are not yet removed in this phase |
| PlatformInit | Platform setup before everything |
| Resolving | Very early processing of components and avatar state, before large-scale changes; Modular Avatar (MA) resolves serialized references and clones controllers here. NDMF's built-in pass also removes EditorOnly objects in this phase |
| Generating | Generate components that later plugins consume. A pass that creates components or merge requests for Modular Avatar belongs here |
| Transforming | General-purpose avatar transformations; most of Modular Avatar's logic runs here |
| Optimizing | Pure optimizations that must run late |
| PlatformFinish | Platform-specific cleanup and validation after optimization (for example parameter-limit validation) |

- Declare a pass as `InPhase(BuildPhase.Generating)` and constrain it with
  `.BeforePlugin("nadena.dev.modular-avatar")` or `.AfterPlugin(...)`, by
  qualified-name string or by plugin type, then register the work in one
  of two forms: `.Run("Display name", Pass)` where Pass is an
  `Action<BuildContext>`-shaped method, or `.Run(new MyPass())` where
  MyPass is a `Pass<MyPass>` subclass whose work is its
  `protected override void Execute(BuildContext)`. A class pass carries
  its own display name and declares the extension contexts it needs
  with `[DependsOnContext(typeof(...))]` on the class; NDMF activates
  those contexts before Execute runs.
- Ordering constraints are the contract between plugins. If a pass produces
  something another plugin consumes (a component, a merge animator, a
  parameter), the producer must be ordered before the consumer; an unordered
  pass that depends on another plugin's output is a latent silent failure.
- Passes from many plugins share each phase. Do not assume a pass sees the
  avatar as the author left it; earlier passes may already have cloned,
  renamed or removed objects. Resolve objects through the BuildContext
  (below) rather than caching scene references across passes.

## BuildContext

The pass receives a `BuildContext` with, among others:

| Member | Use |
| --- | --- |
| AvatarRootObject / AvatarRootTransform | The avatar being built; the root for component collection and relative paths |
| ObjectRegistry | Tracks object identity across cloning and replacement |
| GetState<T>() | Per-build state shared between a plugin's own passes |

- When a pass replaces an object with a clone or a generated object,
  `ObjectRegistry.RegisterReplacedObject(old, new)` records the
  replacement so a later pass or error reporting that still holds
  the original can resolve it; register when such a holder may
  exist (P-18). Use
  `ObjectRegistry.GetReference(obj)` when a stable reference is needed for
  errors or for state.
- Extension methods in `nadena.dev.ndmf` (and the VRChat extensions in
  `nadena.dev.ndmf.vrchat`) add avatar-specific accessors on BuildContext;
  check the installed source for what exists before writing a helper.
- Generated assets that end up referenced by the avatar are persisted
  automatically when the build finishes; an explicit
  `AssetSaver.SaveAsset(obj)` call is needed only when the object must
  already be persistent before the build ends, or when it is
  intentionally not reachable from the avatar. Do not call it
  reflexively: it does not repair an object nothing references.
- Cross-pass data of the plugin's own belongs in GetState<T>() with a
  plugin-specific state type, not in lambda captures or statics
  (NDMF best practices).

## Package and assembly shape

The assembly and package layout — the Runtime/Editor split, the asmdef
fields and reference lists, assembly-info, and the two cases'
package.json dependencies — is specified in CHEATSHEET (Assemblies;
package.json) and P-01; it is not restated here. Two framework facts
belong with it:

- The component itself is `MonoBehaviour, IEditorOnly`: configuration
  only, no runtime behaviour. Two different removals apply to two
  different things, and neither happens before Generating:
  - NDMF's own removal runs early in Resolving and destroys
    GameObjects tagged "EditorOnly"; it does not look at components.
    A component on an untagged object is untouched by it, which is
    why a Generating pass can collect the component.
  - Components implementing IEditorOnly are removed by the VRChat
    SDK, after the NDMF optimize hook: the SDK's
    RemoveAvatarEditorOnly follows the -1025 callback that runs
    Optimizing through PlatformFinish. A plugin whose component must not be
    visible to an Optimizing-phase processor removes it itself
    before then (Avatar Optimizer Compatibility, Route 1).
  NDMF's own `INDMFEditorOnly` is the portable form of the marker: it
  derives from IEditorOnly only when the VRChat SDK is present and
  is a no-op marker otherwise, for component source that must also
  compile in projects without the SDK.
- package.json lists NDMF directly even when Modular Avatar would bring
  it transitively, because the plugin compiles against NDMF itself.

## Preview system

Separate from the build pipeline, NDMF runs a preview system in the
editor (namespace nadena.dev.ndmf.preview). The filter contract, Modular
Avatar's per-component preview filters, and the decision order for a
product's preview story are specified in Modular Avatar Integration
(Preview); the procedure is P-21. This is how Modular Avatar previews
its Reactive Components.

## VRChat build context

For VRChat avatars, NDMF's BuildContext carries the avatar descriptor:
read it through the VRChatAvatarDescriptor() extension method. The older
AvatarDescriptor property on BuildContext is marked obsolete in favour of
that extension; new code uses the extension.

On a VRChat build, NDMF does not run as one block: it registers two
VRCSDK preprocess callbacks. The first (callbackOrder -11000) runs the
phases from FirstChance through Transforming; the second (-1025, just before
the SDK's RemoveAvatarEditorOnly) runs Optimizing through PlatformFinish.
Anything hooking the VRCSDK chain between those orders — VRCFury, at
about -10000, is the common case — therefore runs between NDMF's
Transforming and Optimizing phases: earlier passes cannot see its
output, Optimizing passes can. Entering Play mode goes through the
same VRCSDK preprocessor chain, so the same split applies there. On a
VRChat build the context is also serialized once mid-build, right
after Transforming, in addition to the end-of-build serialization.

## Version floors

The APIs in this file are those of NDMF 1.15.*; they arrived
incrementally, and the floors that matter are:
ErrorReport and ObjectRegistry 1.3.0; the preview framework 1.5.0;
IAssetSaver and [DependsOnContext] 1.6.0; AnimatorServicesContext and
INDMFEditorOnly 1.7.0; ManualProcessAvatar 1.10.0; the single-step
build debugger 1.14.1. A plugin's declared NDMF range in
package.json must not promise a lower bound below the floor of an
API the plugin actually uses; when a project pins an older NDMF than
the skill's verified version, check the installed source and the
NDMF changelog before relying on a named API.
