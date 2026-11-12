# NDMF Plugin Development

Build VRChat avatar (SDK3A) plugins on NDMF (Non-Destructive Modular
Framework) for Unity 2022 LTS: an editor-only
component the author configures in the Inspector, plus a build-time plugin that
validates the configuration and generates or transforms avatar content
(typically an AnimatorController merged into FX).

This skill covers exactly two dependency cases, and nothing else is a
dependency of the plugins it describes:

1. **NDMF only** — the plugin depends on NDMF alone.
2. **NDMF + Modular Avatar (MA)** — the plugin depends on NDMF and
   Modular Avatar.

Dependency versions: NDMF 1.15.0-beta.5, Modular Avatar
1.19.0-rc.1. NDMF APIs move between versions, so Step 1 of the workflow
re-verifies every API against the installed version before use.

No product other than the two dependencies is a source of code or API
surfaces for a plugin built with this skill, and none becomes a
dependency through it.
Load this skill before designing or writing an NDMF plugin, even for one pass.
Do not use it for world or Udon work (see agent-skills-vrc-udon), for shader
delivery checks (see Unity Code Delivery), or for adopting a VRChat mechanism
whose behaviour is unverified (see VRChat Mechanism Check).

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
| [Anti-Patterns](references/anti-patterns.md) | Unity habits that break under NDMF/MA, and the rule that replaces each: for reviewing code written without framework knowledge | `references/anti-patterns.md` |
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
| 3 | Edit the avatar's existing FX controller asset in place | Mutates the author's asset and races with Modular Avatar's own cloning and merging | Generate a new AnimatorController in the pass and merge it (Animator Generation) |
| 4 | Add a ModularAvatarMergeAnimator from a pass that runs after Modular Avatar | MA has already consumed the merge components; the generated controller is silently dropped | Order the pass InPhase(BuildPhase.Generating).BeforePlugin("nadena.dev.modular-avatar") |
| 5 | Destroy source components before generation succeeds, or destroy ones that failed validation | The author loses the configuration with no trace of what was wrong | DestroyImmediate only the components that generated successfully, after the merge component is added |
| 6 | Use GameObject active state (or any runtime script) as the trigger | An uploaded avatar runs no user scripts; there is no runtime to observe it | Animator parameter conditions (the Driver pattern), or MA reactive components |
| 7 | Trust serialized arrays and elements to be non-null | Data written by older versions or by script can carry null arrays and null elements into the pass | Sanitize once at the top of the pass (null array = empty, drop null elements) and use the sanitized arrays everywhere |
| 8 | Take SDK field or enum names from documentation or memory | Doc names are display names; a wrong C# identifier only fails on the user's machine | Verify against code that compiles against the SDK, or the SDK itself; mark anything unverified |
| 9 | Generate destructively into the avatar descriptor's own animator controllers inside a distributed plugin's build pass | The NDMF build runs on a clone so the author's assets survive it; writing into their controllers breaks that guarantee and races MA | Destructive routes belong to editor tooling the author runs deliberately (Animator Generation); distributed plugins generate and merge |

## Reference Loading Guide

| Task | MANDATORY READ | Optional | Do NOT Load |
| --- | --- | --- | --- |
| Know the task, need the procedure (any task) | Grep `references/procedure-index.md` for the task word or identifier, read that entry, then its Depth reference | — | — |
| New plugin: skeleton, phases, ordering, package layout | `references/framework.md`, `CHEATSHEET.md` | — | `references/animator-generation.md` |
| Pass that generates animator layers, states or behaviours | `references/animator-generation.md` | `references/framework.md` | — |
| Attaching VRChat state behaviours, or choosing a destructive output route | `references/animator-generation.md` (Attaching SDK state behaviours; Output routes) | `references/vrchat-avatar-platform.md` (State Behaviour Semantics) | — |
| Emitting Modular Avatar components, working with Reactive Components, or preview | `references/modular-avatar-integration.md` | `references/framework.md` (Preview system) | — |
| Parameter validation, parameter conversion | `references/parameters.md` | `references/framework.md` | — |
| Error reporting, validation severity policy | `references/error-reporting.md` | `references/parameters.md` | — |
| Declaring parameters, building Expressions Menus, parameter providers | `references/menus-and-parameters.md` | `references/modular-avatar-integration.md` | — |
| Building a component that adds parameters and menu items (a menu-generator component) | `references/menus-and-parameters.md` (Components that add parameters and menu items) | `references/modular-avatar-integration.md` (Emitting Modular Avatar components) | — |
| Toggling/moving objects, driving blendshapes, or animating shader (material) properties | `references/manipulation-techniques.md` | `references/modular-avatar-integration.md` (Reactive Components) | — |
| Assembly layout: asmdef fields, assembly-info, test assemblies | `CHEATSHEET.md` (Assemblies) | — | — |
| Choosing between the two dependency cases (NDMF only vs NDMF + Modular Avatar) | Design Decisions in this file | `references/framework.md` | — |
| Reviewing or fixing an existing plugin | The reference for the area touched, plus the NEVER list | `CHEATSHEET.md` | — |
| Reviewing code written by a Unity developer new to NDMF/MA | `references/anti-patterns.md` | The NEVER list | — |
| VRChat platform facts behind a design or generation choice (playable layers, built-in parameters, allowed components, PhysBone/Contact/Raycast parameters) | `references/vrchat-avatar-platform.md` | The reference for the area touched | — |
| Delivering or reviewing a plugin as a VPM package (package.json fields, VCC install routes, Assets migration) | `references/vpm-packaging.md` | `references/framework.md` | — |
| Avatar Optimizer warns about the plugin's component, or a component must survive until late in the build | `references/avatar-optimizer-compat.md` | `references/framework.md` | — |
| Build output is wrong and the failing point in the pipeline is unknown, or behaviour differs between a VRChat build / Play mode and a plain build | `references/debugging.md` | `references/framework.md` | — |

## Design Decisions Before Starting

Settle these before writing code; changing any of them mid-implementation
usually rewrites the pass.

- **Dependency case.** Exactly two cases exist in this skill. Case 1,
  **NDMF only**: the plugin works through NDMF's own services (phases,
  BuildContext, ErrorReport, ParameterInfo) and transforms the build-time
  avatar clone or generates assets. There is no merge component in this
  case, so a plugin that adds animator layers manages the target
  controller itself (clone it; never mutate the shared asset). Case 2,
  **NDMF + Modular Avatar**: adds the generate-and-merge route through
  ModularAvatarMergeAnimator, MA's parameter surface, Reactive Components,
  and MA's preview support. Pick the case before designing the pass; the
  output route and ordering constraints follow from it. No other
  product is a third case, and none becomes a dependency through this
  skill.
- **Trigger model.** Generation is build-time and the avatar has no runtime
  code, so a component's effect must be expressed as animator parameter
  conditions, MA reactive behaviour, or a static transformation. If a design
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
- **Naming in one place.** Package name, plugin QualifiedName, namespaces
  and assembly names are decided once, up front, so a rename touches one
  definition each. QualifiedName defaults to the plugin class's full name;
  override it with a stable product id.

## Core Workflow

Run the steps in order.

1. **Verify versions and APIs.** Read the plugin and API sources of the
   versions actually installed (or the pinned reference versions above).
   Confirm each type, method and enum member the design uses; verify VRChat
   mechanism behaviour per VRChat Mechanism Check. Write down what remains
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
   (including inactive), sanitize their serialized data, validate through
   ParameterInfo, generate, merge, then remove only the successfully
   generated source components.
5. **Generate the animator output** per Animator Generation: one layer per
   component or setting, idle/active states, condition-gated transitions,
   the SDK behaviour on the active state, and the merge component
   (dependency case 2).
6. **Report every author-facing problem through ErrorReport** per Error
   Reporting, with the component as context. No Debug.Log for validation.
7. **Audit and hand over** per Unity Code Delivery: static checks first
   (brace balance, JSON validity, invented-API scan, cross-references,
   .meta completeness), the three handover lists, and Unity import stated
   as unconfirmed until the user reports back. The invented-API scan
   misses one case: a method that exists in another product — or in a
   reference product that depends on a library your package does not —
   passes a name check while failing to compile. Verify that every called
   method resolves inside the package's own dependency closure (its own
   assemblies plus its declared dependencies), not merely that it exists
   somewhere in the reference set.

## Terminology

Fixed forms for this skill; keep new or edited text aligned.

| Form | Rule |
| --- | --- |
| NDMF | Non-Destructive Modular Framework. Expanded at first use in this file; the references use the short form. |
| Modular Avatar (MA) | Expanded with the short form at first use in each file, MA afterwards. The joined form (ModularAvatarParameters and similar) appears only in code identifiers and type names, never in prose. |
| Avatar Optimizer (AAO) | Same expansion rule as MA. |
| ordering constraint | The only "constraint" of NDMF sequencing: a BeforePlugin / AfterPlugin declaration (Framework). |
| Unity constraint / VRChat constraint | The constraint components. A bare "constraint" for a component appears only inside the platform reference's constraint sections, where the subject is fixed. A limitation or precondition is a "condition" or a "rule", never a "constraint". |
| Owner-qualified implementation names | Names shared across ecosystems (Unity built-ins, the VRChat SDK, third-party assets such as FinalIK or the deprecated DynamicBone) are written with their owner or as the type name — VRCRaycast, VRCPhysBone, Unity constraint, VRChat constraint — because the same word names different implementations in each. A bare shared name never names a component outside a section whose subject fixes it. |
| Expression Parameters / Expressions Menu | VRChat's feature names, always in this form. |
| behaviour | The prose spelling of this skill. Names taken from a source (type names, page titles, URLs) keep the source spelling. |
| SDK3A | The VRChat SDK for avatars; this skill's scope. "SDK3" alone appears only where a source uses it (the platform's avatar version value, legacy projects). |

## Sources

The framework facts were read from the two dependency products at the
versions listed above.

| Product | What was taken from it | License |
| --- | --- | --- |
| bdunderscore/ndmf | Framework: plugin model, phases, BuildContext, ErrorReport, ParameterInfo, preview system | MIT |
| bdunderscore/modular-avatar | Integration: MergeAnimator, parameter surface, Reactive Components, preview filters, large-plugin pass organization | MIT (Editor/images excluded from redistribution) |
