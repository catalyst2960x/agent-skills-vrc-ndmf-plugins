---
name: ndmf_plugin_development
description: "VRChat avatar plugin development with NDMF (Non-Destructive Modular Framework) and Modular Avatar (MA): build-time passes that generate animator controllers, parameters and menus, integrate Avatar Optimizer (AAO), and package for VPM. Covers NDMF-only and NDMF + MA projects."
---

# NDMF Plugin Development

Build VRChat avatar (SDK3A) plugins on NDMF (Non-Destructive Modular
Framework) for Unity 2022 LTS: an editor-only
component the author configures in the Inspector, plus a build-time plugin that
validates the configuration and generates or transforms avatar content
(typically an AnimatorController merged into FX).

This skill covers exactly two dependency cases. The cases are about
the framework dependencies, NDMF and Modular Avatar. The VRChat SDK
is not a case: it is the platform both cases build for — present in
the target project, referenced by both cases' asmdefs, and declared
as a package dependency the same way in both cases (VPM Packaging).

1. **NDMF only** — of the framework dependencies, the plugin depends
   on NDMF alone.
2. **NDMF + Modular Avatar (MA)** — the plugin depends on NDMF and
   Modular Avatar.

Dependency versions: NDMF 1.15.*, Modular Avatar 1.19.*.
NDMF APIs move between versions, so Step 1 of the workflow
re-verifies every API against the installed version before use.

Load this skill before designing or writing an NDMF plugin, even for one pass.
Do not use it for world or Udon work, for shader delivery checks, or
for adopting a VRChat mechanism whose behaviour is unverified.

## Table of Contents

| Section | Content | Reference |
| --- | --- | --- |
| [Common Mistakes](#common-mistakes-never-list) | Failure modes to design out | This file |
| [Design Decisions](#design-decisions-before-starting) | Choices to settle before code | This file |
| [Core Workflow](#core-workflow) | The seven steps, in order | This file |
| [Procedure Index](references/procedure-index.md) | Every build task as a greppable procedure: steps and traps, facts live in the other references | `references/procedure-index.md` |
| [Framework](references/framework.md) | Plugin, phases, ordering, BuildContext | `references/framework.md` |
| [Animator Generation](references/animator-generation.md) | Layers, states, transitions, Direct Blend Tree layers, behaviours, merge | `references/animator-generation.md` |
| [Modular Avatar Integration](references/modular-avatar-integration.md) | Emitting MA components, Reactive Components, preview | `references/modular-avatar-integration.md` |
| [Parameters](references/parameters.md) | The build-time parameter set: ParameterInfo, EffectiveName, conversions between representations | `references/parameters.md` |
| [Error Reporting](references/error-reporting.md) | Reporting through ErrorReport: SimpleError, severities, the validation severity policy | `references/error-reporting.md` |
| [Expression Parameters and Expressions Menu](references/menus-and-parameters.md) | Reading/writing Expression Parameters and Expressions Menus, providers, budget | `references/menus-and-parameters.md` |
| [Manipulation Techniques](references/manipulation-techniques.md) | Driving objects, blendshapes, and shader parameters: reactive vs manual routes, property bindings | `references/manipulation-techniques.md` |
| [Anti-Patterns](references/anti-patterns.md) | Unity habits that break under NDMF/MA, explained as patterns: for reviewing code written without framework knowledge | `references/anti-patterns.md` |
| [VRChat Avatar Platform](references/vrchat-avatar-platform.md) | VRChat-side facts the generated output must satisfy: playable layers, built-in parameters, behaviour semantics, allowed components, stations, spatial audio, component-generated parameters | `references/vrchat-avatar-platform.md` |
| [VPM Packaging](references/vpm-packaging.md) | VRChat-side requirements for delivering a plugin as a VPM package: manifest fields, dependency ranges, legacy migration from Assets, user-package testing | `references/vpm-packaging.md` |
| [Avatar Optimizer Compatibility](references/avatar-optimizer-compat.md) | Keeping a plugin's components from triggering AAO's unknown-component warning: remove before AAO, or register with its API | `references/avatar-optimizer-compat.md` |
| [Debugging](references/debugging.md) | Finding where a build goes wrong: inspecting a manual build's output, the single-step build debugger, ordering diagnosis | `references/debugging.md` |
| [Cheatsheet](CHEATSHEET.md) | Skeleton, asmdef and package tables | `CHEATSHEET.md` |

## Common Mistakes (NEVER List)

| # | NEVER do this | Why it hurts | Use instead |
| --- | --- | --- | --- |
| 1 | Report a build problem with Debug.Log | The log is not the build report; the author never sees the failure tied to the component | NDMF ErrorReport with the component's GameObject as context (see Error Reporting) |
| 2 | Read the VRCExpressionParameters asset directly to learn parameter names and types | Other plugins rename, add and sync parameters during the build; the asset is not the build-time truth | ParameterInfo.ForContext(ctx).GetParametersForObject(avatarRoot), keyed by EffectiveName |
| 3 | Edit the avatar's existing FX controller asset in place | Mutates the author's asset and races with Modular Avatar's own cloning and merging | Case 2: generate a new AnimatorController in the pass and merge it. Case 1: edit through virtual controllers (Animator Generation, Merging into the avatar) |
| 4 | Add a ModularAvatarMergeAnimator from a pass that runs after Modular Avatar | MA has already consumed the merge components; the generated controller is silently dropped | Order the pass InPhase(BuildPhase.Generating).BeforePlugin("nadena.dev.modular-avatar") |
| 5 | Destroy a source component before its validation and generation have run, or leave a found instance of it on the avatar | An early destroy skips that component's error report; a leftover instance reaches Avatar Optimizer as an unknown component | Process every found instance first, then destroy every found instance, success or failure; removal acts on the build clone only (P-18) |
| 6 | Use GameObject active state (or any runtime script) as the trigger | An uploaded avatar runs no user scripts; there is no runtime to observe it | Animator parameter conditions (the Driver pattern), or MA reactive components |
| 7 | Trust serialized arrays and elements to be non-null | Data written by older versions or by script can carry null arrays and null elements into the pass | Sanitize once at the top of the pass (null array = empty, drop null elements) and use the sanitized arrays everywhere (P-04) |
| 8 | Take SDK field or enum names from documentation or memory | Doc names are display names; a wrong C# identifier only fails on the user's machine | Verify against code that compiles against the SDK, or the SDK itself; mark anything unverified |
| 9 | Generate destructively into the avatar descriptor's own animator controllers inside a distributed plugin's build pass | The NDMF build runs on a clone so the author's assets survive it; writing into their controllers breaks that guarantee and races MA | Destructive routes belong to editor tooling the author runs deliberately (Animator Generation); distributed plugins generate and merge |

## Design Decisions Before Starting

Settle these before writing code; changing any of them mid-implementation
usually rewrites the pass.

- **Dependency case.** Exactly two cases exist in this skill. Case 1,
  **NDMF only**: the plugin works through NDMF's own services (phases,
  BuildContext, ErrorReport, ParameterInfo) and transforms the build-time
  avatar clone or generates assets. There is no merge component in this
  case: a plugin that adds animator output edits the avatar's own
  controllers through NDMF's virtual controllers
  (AnimatorServicesContext; Animator Generation, Merging into the
  avatar), and never mutates the shared controller assets. Case 2,
  **NDMF + Modular Avatar**: adds the generate-and-merge route through
  ModularAvatarMergeAnimator, MA's parameter surface, Reactive Components,
  and MA's preview support. Pick the case before designing the pass; the
  output route and ordering constraints follow from it. No other
  product is a third case, and none becomes a dependency through this
  skill.
- **Trigger model.** On an uploaded SDK3A avatar no author code runs:
  neither MonoBehaviours nor custom StateMachineBehaviours execute, so
  the AnimatorController — driven by parameters and carrying the SDK's
  own state behaviours — is the only runtime mechanism a product can
  operate through. Generation is build-time, so a component's effect
  must be expressed as animator parameter conditions, MA reactive
  behaviour, or a static transformation. If a design
  seems to need runtime observation, the design is wrong, not the framework.
- **Output route.** Generate a fresh AnimatorController and merge it, or
  transform what exists through NDMF/MA services. Pick one route per product
  and keep the ordering constraint it implies (NEVER #4). A destructive
  route (writing into the descriptor's own controllers) is only for editor
  tooling the author runs on their own project, never for a distributed
  plugin's build pass (NEVER #9).
- **Preview story.** For every visible effect, decide how the author sees
  it in the editor before building: inherit Modular Avatar's preview by
  expressing the effect as a Reactive Component, write an NDMF preview
  filter (IRenderFilter), or state explicitly that the product has no
  preview (legitimate for non-visual output such as audio).
- **Error policy.** For each defect decide Error (skip this component,
  generate the rest) versus NonFatal (generate anyway), from the end
  behaviour's own documented fallback: if the behaviour defines a
  fallback for a missing value, warn; if it has none and the missing
  value is used as given, error.
- **Loose coupling to referenced sets.** What a component references
  by name is input, not a claim to verify. The build does not fail
  because a referenced name is absent from the set the build can
  see: that set is an inventory other passes keep changing, and
  generation that depends on its current contents breaks when the
  inventory moves. Mistakes are prevented at input, not detected at
  build: where the product can detect the candidates it offers them
  as editor suggestions, and the field stays editable as free text
  so a name the product cannot detect is still entered and corrected
  by hand. Applied to parameters: a condition's name is never
  validated for existence and never skips a component; type
  resolution still consults the build-time set, the derived names,
  and the built-in catalogue (Parameters), and a name no source
  types registers as Float.
- **Naming in one place.** Package name, plugin QualifiedName, namespaces
  and assembly names are decided once, up front, so a rename touches one
  definition each. QualifiedName defaults to the plugin class's full name;
  override it with a stable product id.

## Core Workflow

Run the steps in order.

1. **Verify versions and APIs.** Read the plugin and API sources of the
   versions actually installed (or the pinned reference versions above).
   Confirm each type, method and enum member the design uses; verify each
   VRChat mechanism's behaviour against the official documentation before
   the design relies on it. Write down what remains
   unverified instead of smoothing it over.
2. **Lay out the package.** Runtime assembly: the component (MonoBehaviour +
   IEditorOnly) and its serializable types. Editor assembly: plugin,
   inspector, drawers. Set the asmdef fields deliberately and add an
   assembly-info file where internals are shared with tests or sibling
   assemblies (Cheatsheet, Assemblies). package.json carries the VPM
   dependencies of the chosen dependency case. Every file and folder gets
   a .meta with a unique GUID.
3. **Write the plugin skeleton.** [assembly: ExportsPlugin], Plugin<T>,
   Configure() declaring each pass with its phase and ordering constraints
   (Framework reference; skeleton in the Cheatsheet).
4. **Write the pass.** Collect target components under the avatar root
   (including inactive), sanitize their serialized data, resolve their
   parameter types through ParameterInfo, generate, merge, then remove
   every found source
   component instance, success or failure (P-18).
5. **Generate the animator output** per Animator Generation: the shape
   chosen per feature (the layer/state pattern, or the Direct Blend Tree
   shape), and the merge component (dependency case 2).
6. **Report every author-facing problem through ErrorReport** per Error
   Reporting, with the component as context. No Debug.Log for validation.
7. **Audit and hand over.** Static checks first: brace balance, JSON
   validity, the invented-API scan below, cross-references, and .meta
   completeness; P-17 runs this step as a procedure. The invented-API
   scan misses one case: a method that exists in another product — or
   in a reference product that depends on a library your package does
   not — passes a name check while failing to compile. Verify that
   every called method resolves inside the package's own dependency
   closure (its own assemblies plus its declared dependencies), not
   merely that it exists somewhere in the reference set. Hand over
   with three lists: what was tried but could not be done; what was
   asked but not done; and what was done beyond the ask. State Unity
   import as unconfirmed until the user reports back.

## Sources

The framework facts were read from the two dependency products at the
versions listed above.

| Product | What was taken from it | License |
| --- | --- | --- |
| bdunderscore/ndmf | Framework: plugin model, phases, BuildContext, ErrorReport, ParameterInfo, preview system | MIT |
| bdunderscore/modular-avatar | Integration: MergeAnimator, parameter surface, Reactive Components, preview filters, large-plugin pass organization | MIT (Editor/images excluded from redistribution) |
