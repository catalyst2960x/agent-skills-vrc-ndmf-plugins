# Procedure Index (reverse lookup)

Procedures for building an NDMF plugin, one entry per task, written to be
found by grep: search this file (or the whole skill) for the task word
or for the identifier you are about to write, and
read only that entry. Entries carry the steps and the traps; the facts
and reasoning stay single-sourced in the reference named under Depth, so
when a fact changes, it changes in one place and this index only changes
when a procedure itself changes.

Entry format: `## P-nn <task>` heading, a Keys line (grep vocabulary), the
dependency case, a Requires line (what must already hold before step 1),
numbered steps, and a Depth pointer.

## Requirements applying to every procedure

Keys: requirements, prerequisites, environment, versions

- Environment: Unity 2022.3, with the VRChat SDK (avatars) in the target
  project. Identifiers are those of the versions pinned in SKILL.md
  (NDMF 1.15.0-beta.5; Modular Avatar (MA) 1.19.0-rc.1 in case 2).
  At any other version, re-verify before applying a procedure
  (SKILL.md, Core Workflow step 1).
- The design decisions recorded in SKILL.md — dependency case, trigger
  model, output route, preview story, error policy, naming — are inputs
  to these procedures, not outputs of them. A Requires line that names
  one assumes it is already decided and written down.
- Each entry's Requires line lists only what that procedure adds beyond
  this block and beyond its own Case field.

## P-01 Scaffold a new plugin package
Keys: scaffold, package layout, asmdef, why asmdef matters, AssemblyInfo, package.json, .meta, GUID
Case: both · Depth: SKILL.md Core Workflow step 2; CHEATSHEET (Assemblies; package.json)
Requires: the case decided (P-02); product naming decided — plugin qualified name, namespaces, assembly names (SKILL.md, Naming); a license chosen for the package, because adapting third-party code creates NOTICE duties (SKILL.md, Sources).

1. Decide the dependency case first (P-02's input): it fixes package.json.
2. Create the Runtime assembly (component + serializable types) and the
   Editor assembly (plugin, inspector, drawers); one asmdef each, Editor
   asmdef with includePlatforms = Editor and autoReferenced = false, both
   with a deliberate name and rootNamespace.
3. Add an assembly-info file to any assembly whose internals a test or
   sibling assembly must see (InternalsVisibleTo).
4. Write package.json with exactly the chosen case's vpmDependencies.
5. Give every file and folder a .meta with a unique GUID.

Why the asmdefs are load-bearing, not ceremony. Without them, Unity folds
the package's scripts into the predefined assemblies, and four things the
rest of this skill depends on stop existing:

- **The Editor/player boundary.** includePlatforms = Editor on the Editor
  asmdef is the guarantee that build-pass and inspector code — which
  references UnityEditor and NDMF's editor APIs — never enters the player
  build the user uploads. Folder-name conventions are not a guarantee a
  distributed package can rest on.
- **Assembly identity.** Three mechanisms address assemblies by name:
  InternalsVisibleTo grants (step 3), define-constraint gating of
  optional integrations (P-16), and test assemblies referencing the product. All
  three are impossible without named assemblies.
- **An explicit reference graph.** The asmdef references list is the
  assembly-level form of the dependency closure (P-17): in the predefined
  assemblies every script sees every other script, so a reference the
  package does not declare compiles at home and fails in a user's project.
- **Compile scope.** Importers of the package recompile only the changed
  assembly and its dependents, not the whole predefined assembly; on the
  large avatar projects this plugin targets, that is the iteration cost.

The asmdef name is also a compatibility surface: other assemblies
reference it and InternalsVisibleTo grants name it, so renaming one after
release breaks dependents. Choose names once, as deliberately as
parameter names (P-12). Leaving autoReferenced at its default (true) on
the Editor asmdef makes every assembly in the user's project reference
the plugin's editor code — set it false.

## P-02 Choose the dependency case
Keys: dependency case, NDMF only, Modular Avatar, which case
Case: — · Depth: SKILL.md Design Decisions (Dependency case)
Requires: the product's feature list settled — which outputs it must produce (animator output, parameters, menus, reactive effects) — because the case follows from the outputs, not the reverse.

1. If the product needs merged animator output, MA parameters, menus, or
   reactive components, it is case 2 (NDMF + Modular Avatar).
2. Otherwise case 1 (NDMF only): plan to transform the clone or generate
   assets with NDMF services alone; animator output means managing the
   controller yourself.
3. Record the case where the design is written down; passes, asmdef
   references, and package.json all follow from it.

## P-03 Register a pass with phase and ordering
Keys: pass registration, Configure, InPhase, BuildPhase, Generating, BeforePlugin, Run, Sequence, ordering
Case: both · Depth: Framework
Requires: P-01 (an Editor assembly to hold the plugin class); P-02 (the ordering target in step 2 exists only in case 2).

1. In the plugin's Configure(), declare each pass with
   InPhase(BuildPhase.<phase>).
2. Generation for Modular Avatar goes in Generating, ordered
   .BeforePlugin("nadena.dev.modular-avatar"); a pass that removes
   components also declares
   .BeforePlugin("com.anatawa12.avatar-optimizer") (P-19).
3. Name each pass with .Run("<name>", handler); the name is what build
   logs and error reports show.
4. Keep Configure() free of build work: it only declares.

## P-04 Collect target components and sanitize their data
Keys: collect, GetComponentsInChildren, sanitize, null, array, inactive, AvatarRootObject
Case: both · Depth: SKILL.md Core Workflow step 4; NEVER #7
Requires: P-03 (collection runs inside a pass, against its BuildContext); the runtime component type and its serialized fields already defined.

1. Collect with ctx.AvatarRootObject.GetComponentsInChildren<T>(true) —
   the true includes inactive objects.
2. Sanitize each component's serialized arrays once, at the top: null
   array = empty, null elements dropped.
3. Use only the sanitized arrays for validation, name collection, and
   generation.

## P-05 Validate parameters against the build-time set
Keys: validate, ParameterInfo, EffectiveName, GetParametersForObject, duplicate, first, missing parameter
Case: both · Depth: Parameters; Error Reporting (Validation policy)
Requires: P-04 (sanitized components); if this plugin itself generates the parameters being validated, P-07 in place first — otherwise they are invisible to ParameterInfo.

1. Build the index: ParameterInfo.ForContext(ctx)
   .GetParametersForObject(ctx.AvatarRootObject), grouped by
   EffectiveName, first entry wins on duplicates.
2. Collect the names the component needs (conditions, pre-conditions,
   feature parameters), deduplicated, empty names skipped.
3. Missing condition parameter = Error, skip the component. Missing
   feature parameter with a documented fallback in the end behaviour =
   NonFatal, generate anyway. Structural defects = Error.
4. Never read the VRCExpressionParameters asset for this (NEVER #2).

## P-06 Report a problem to the author
Keys: error reporting, ErrorReport, SimpleError, Localizer, WithContextObject, NonFatal, ErrorSeverity
Case: both · Depth: Error Reporting
Requires: the error policy decided (SKILL.md, Error policy), including the severity split of P-05 step 3; a decision whether the product ships a localization asset or uses the passthrough localizer.

1. Define an error type deriving from SimpleError; override Localizer,
   TitleKey, Severity (and the virtual members you need).
2. Without a localization asset, use a passthrough Localizer whose single
   lookup maps every key to "{0}", passing the message as its own key.
3. Report inside using (ErrorReport.WithContextObject(component.gameObject))
   with ErrorReport.ReportError(error). Severity matches the P-05 policy.
4. Make the message name the actual defect (the missing names, the unmet
   condition). Never Debug.Log (NEVER #1).

## P-07 Supply a component's parameters before generation
Keys: provider, IParameterProvider, ParameterProviderFor, ProvidedParameter, supply before generation
Case: both (an NDMF mechanism) · Depth: Expression Parameters and Expressions Menu (Reading parameters)
Requires: the generator's parameter list (names and animator types) fixed as a single shared definition that the provider and the generator both read.

1. If the component's parameters only exist after generation, other
   validation cannot see them. Add a provider class marked
   [ParameterProviderFor(typeof(TheComponent))] implementing
   IParameterProvider.
2. GetSuppliedParameters returns one ProvidedParameter per parameter:
   name, Animator namespace, the component, the plugin instance, the
   animator parameter type.
3. Keep the provider's list and the generator's list the same list in
   code — one definition, two readers.

## P-08 Register a parameter on a generated controller
Keys: register parameter, AnimatorControllerParameter, AddParameter, conversion, Float, no type
Case: both · Depth: Parameters (Parameter conversions)
Requires: P-05's index built; a controller under construction to register into (P-09's, or an existing one the product manages).

1. From the P-05 index entry, construct an AnimatorControllerParameter
   directly: name = EffectiveName, type = ParameterType, with a null
   type converting as Float.
2. Register each name once per controller (track the added set).
3. Neither framework provides this conversion; do not call a conversion
   helper from a library the package does not depend on (the
   dependency-closure check, SKILL.md step 7).

## P-09 Generate a condition-gated layer
Keys: layer generation, idle, active, pre_active, transition, condition reversal, reverse, hasExitTime, defaultWeight, writeDefaultValues, empty clip
Case: both · Depth: Animator Generation
Requires: the trigger model decided (SKILL.md, Trigger model) — idle/active, optional pre_active, return style; P-05 passed for the component; P-08 for the parameters its conditions name.

1. One controller per pass; one layer per component, named for its
   source, defaultWeight = 1.
2. States idle (default), active, optionally pre_active; every state
   writeDefaultValues = false, sharing one empty clip (one curve, two
   zero keys at 0 and 1/60, on a dummy path).
3. Forward transitions: all conditions of the set (AND), hasExitTime =
   false, hasFixedDuration = true, duration = 0, exitTime = 0. With
   pre_active: idle → pre_active takes the pre-conditions, pre_active →
   active takes the main conditions; no pre_active → idle return.
4. Return: one transition per condition, each carrying that condition
   reversed (OR of negations), or one free transition (hasExitTime =
   true, exitTime = 0). Reversal: If↔IfNot, Equals↔NotEqual, Greater→Less
   and Less→Greater with the Int threshold shifted by 1; Float unchanged.
5. Empty condition set: the forward transition fires unconditionally and
   no return transitions exist. Confirm that is the intent.

## P-10 Attach an SDK state behaviour and map the fields
Keys: attach behaviour, StateMachineBehaviour, VRCAnimatorPlayAudio, field mapping, clamp, relative path, SourcePath
Case: both · Depth: Animator Generation (Mapping; Attaching SDK state behaviours)
Requires: the VRChat SDK present in the project (the behaviour type comes from it); the field mapping and limits verified against SDK-compiling code (NEVER #8); P-09's states, since the behaviour lands on the active state.

1. Create the behaviour, set it on the active state only; it carries both
   the enter and exit settings.
2. Convert at generation time: clamp numbers to the behaviour's
   documented limits and swap min/max after clamping; drop null payload
   entries (clips); turn object references into the path the behaviour
   expects — relative to the avatar root, root itself = "" — after
   validating the target is the root or a descendant.
3. Verify every SDK field and enum identifier against SDK-compiling code
   before writing it; mark what stays unverified (NEVER #8).

## P-11 Merge the generated controller into the avatar
Keys: merge, ModularAvatarMergeAnimator, MergeAnimator, FX, matchAvatarWriteDefaults, removal, DestroyImmediate
Case: 2 only · Depth: Animator Generation (Merging); NEVER #4, #5
Requires: case 2; P-03's ordering before Modular Avatar; P-09's controller; the target playable layer decided (SKILL.md, Output route).

1. Add one ModularAvatarMergeAnimator at the avatar root: animator = the
   generated controller, layerType = FX (or the target layer),
   matchAvatarWriteDefaults per the generated states' assumption.
2. The generating pass must already be ordered before Modular Avatar
   (P-03), or the merge is silently dropped.
3. Only after the merge component exists, DestroyImmediate every
   found source component instance, including ones that failed
   generation (P-19).

## P-12 Declare parameters for the avatar
Keys: declare parameters, ModularAvatarParameters, ParameterConfig, budget, 256, bit, localOnly, saved, sync, name collision
Case: 2 (case 1 has no declaration service) · Depth: Expression Parameters and Expressions Menu (Writing parameters)
Requires: case 2; parameter names decided once and kept stable; the avatar's existing parameter usage read (P-05), so the budget is checked against the real avatar, not in isolation.

1. One ParameterConfig per parameter: nameOrPrefix, the syncType (the
   synced value type, or NotSynced), saved flag, localOnly when remote
   players do not need it, an explicit default value when the design
   needs one. Use Animator Only entries for work parameters and Prefix
   entries for generated families; set bitWidth below 8 when a
   small-range synced value should cost less.
2. Check the sync budget against the avatar's real usage (Expression
   Parameters and Expressions Menu, Budget): the budget is shared across
   the whole avatar,
   so a declaration is a claim on a scarce resource.
3. Apply the collision rules (Expression Parameters and Expressions Menu, Writing parameters).
   Prefix product parameters and treat collisions found through
   ParameterInfo as validation errors.
4. Names are a compatibility surface: decide them once, keep them stable.

## P-13 Build an Expressions Menu
Keys: build menu, EXMenu, ModularAvatarMenuInstaller, menuToAppend, installTargetMenu, ModularAvatarMenuItem, SubMenu, Puppet, sub-parameters, 8 controls
Case: 2 only (case 1 has no menu route) · Depth: Expression Parameters and Expressions Menu (Writing menus)
Requires: case 2; the parameters the controls bind already declared (P-12) or supplied (P-07); for the installer route, an asset container the pass owns, to store the generated menus.

1. Choose the route: a generated VRCExpressionsMenu asset + a
   ModularAvatarMenuInstaller, or a ModularAvatarMenuItem hierarchy.
2. Installer route: store the generated menu assets in an asset container
   the pass owns; set menuToAppend, and installTargetMenu only when
   nesting into a specific existing menu (empty = root menu).
3. Hierarchy route: one GameObject per control with a
   ModularAvatarMenuItem; set its MenuSource (child items, a referenced
   asset, or another object's children). An item whose parameter is
   declared nowhere relies on MA's automatic creation: set isSynced,
   isSaved, isDefault, and automaticValue deliberately, or declare the
   parameter in P-12 instead.
4. Respect the menu limits and the puppet sub-parameter bindings
   (Expression Parameters and Expressions Menu, Writing menus). Append;
   never replace the author's
   menu.

## P-14 Drive an object, a blendshape, or a shader parameter
Keys: drive, toggle, m_IsActive, m_Enabled, blendShape, weight, 0..100, material., shader parameter, ObjectToggle, ShapeChanger, MaterialSetter, MaterialSwap, BlendshapeSync
Case: both (reactive route is case 2) · Depth: Manipulation Techniques
Requires: the driving parameter declared or supplied (P-12 / P-07); target objects and renderers resolvable on the build-time clone — through the ObjectRegistry when earlier passes replaced them.

1. Pick the route per property: a Reactive Component when a built-in
   expresses the end state (ObjectToggle, ShapeChanger, MaterialSetter,
   MaterialSwap, ReactiveMove), manual clip generation for continuous
   values, component enable flags, and case 1.
2. Manual bindings: object active = float curve, GameObject, "m_IsActive";
   component enabled = the component type, "m_Enabled"; blendshape =
   SkinnedMeshRenderer, "blendShape.<ShapeName>", weights 0..100; shader
   property = the renderer's type, "material.<PropertyName>", colors and
   vectors per component.
3. Validate blendshape names against the target mesh's real list; report
   misses (P-06). Drive continuous values with a blend tree over the
   float parameter.
4. Never drive one property through both routes.

## P-15 Give a visible effect a preview story
Keys: preview, IRenderFilter, RenderGroup, nadena.dev.ndmf.preview, Reactive, visible effect
Case: both · Depth: Modular Avatar Integration (Preview); Framework (Preview system)
Requires: the preview story decided before building (SKILL.md, Preview story); for the filter route, the proxy-renderer model in Framework (Preview system).

1. Decide before building (SKILL.md, Preview story): inherit MA's
   preview by expressing the effect as a Reactive Component, write an
   IRenderFilter, or state explicitly that there is no preview
   (non-visual output such as audio).
2. IRenderFilter route: implement the filter contract (Modular Avatar
   Integration, Preview) — target groups, the node that restyles proxy
   renderers, the enable gate — including the CanEnableRenderers flag
   when the filter may enable disabled renderers.
3. Preview runs on the live scene outside builds: touch only the proxy
   renderers and materials given, dispose what the node creates, never
   mutate the author's objects.

## P-16 Use Modular Avatar APIs optionally from a case-1 plugin
Keys: versionDefines, defineConstraints, asmdef separation, optional integration, optional
Case: 1 with optional case-2 features · Depth: CHEATSHEET (Assemblies)
Requires: P-01's asmdefs in place; the Modular Avatar version range the integration supports decided, matching the pinned versions unless re-verified.

1. Put every MA-touching type in its own Editor assembly. The main
   Editor asmdef does not reference Modular Avatar at all: an asmdef
   reference is not conditional, and a reference to an assembly that
   is not installed fails compilation of the whole referencing
   assembly — #if cannot gate a reference.
2. In the separated asmdef's versionDefines, define a symbol for the
   Modular Avatar version range the integration supports, and list
   that symbol in the same asmdef's defineConstraints. Unity
   compiles an assembly only when every define constraint holds;
   without Modular Avatar the symbol is undefined and the assembly
   is skipped entirely.
3. Inside the separated assembly the MA references need no #if
   gating. #if remains for version differences within the supported
   range, not for presence.
4. The separated code still follows case-2 rules (ordering, merge,
   removal) whenever it runs.

## P-17 Audit and hand over
Keys: audit, dependency closure, brace, JSON, GUID, static audit, handover, unconfirmed
Case: both · Depth: SKILL.md Core Workflow step 7
Requires: the package complete through the procedures it uses; the NEVER list (SKILL.md). Unity-side import is the user's environment to confirm; it is reported as unconfirmed and is never a gate the user must clear before the work proceeds.

1. Static checks: brace/paren balance, JSON validity for package.json
   and asmdefs, .meta completeness, GUID uniqueness.
   Name-resolution check: a bare type name that matches one of the
   file's namespace segments (Editor inside a *.Editor namespace)
   resolves to the namespace, not the type; qualify it
   (UnityEditor.Editor). Signature check: editor-GUI call shapes the
   code repeats must use overloads verified against the real
   UnityEditor API; PropertyField labels are GUIContent, not string.
2. Dependency-closure check: every called method and type resolves inside
   the package's own assemblies plus its declared vpmDependencies — not
   merely somewhere in a reference product.
3. Cross-check the code against the NEVER list and the pass checklist.
4. Hand over with the three lists defined in SKILL.md (Core Workflow
   step 7) and Unity import stated as unconfirmed until the user
   reports back.
5. Real-machine verification is a separate batch (P-23); rows not
   yet run stay unconfirmed in the handover.

## P-18 Package for VPM distribution
Keys: VPM, VCC, package.json, vpmDependencies, legacyFolders, user packages, manifest, zipSHA256
Case: both · Depth: references/vpm-packaging.md
Requires: the package complete through P-17; the Assets path of any earlier Assets-based distribution, if one exists.

1. Check package.json against the required and recommended fields
   (VPM Packaging, Manifest) and fill what is missing.
2. Declare dependencies in vpmDependencies as ranges, following the
   range and SDK conventions in VPM Packaging (Dependencies and
   version ranges).
3. If an Assets-based predecessor exists, declare its folder in
   legacyFolders so installing the package removes the Assets copy.
4. Test as a user package before publishing: VCC Settings, Add under
   User Packages, then install from the project's Manage Project
   screen.
5. zipSHA256 goes in the repository listing, never in the package's
   own package.json.

## P-19 Remove generated components before Avatar Optimizer runs
Keys: Avatar Optimizer, AAO, unknown component, com.anatawa12.avatar-optimizer, ComponentInformation, Asset Description, DestroyImmediate
Case: both · Depth: references/avatar-optimizer-compat.md
Requires: the component's lifetime decided (build-time only, or present after AAO runs).

1. Build-time-only component (the default): the pass removes every
   instance it found, including instances that failed validation,
   before the Optimizing phase; no early return may skip the
   removal. Declare BeforePlugin("com.anatawa12.avatar-optimizer")
   on the registration as AAO's documentation recommends.
2. Component that must remain after AAO: register it with AAO's API
   instead — ComponentInformation<T> with CollectMutations and
   CollectDependency, the API assembly referenced from the asmdef
   and gated by the version define range given in Avatar Optimizer
   Compatibility (Route 2).
3. Pure data holder that never processes anything: the Asset
   Description route is available; never use it for a component
   whose tool processes the avatar.

## P-20 Diagnose a build problem
Keys: debugging, ManualProcessAvatar, single-step, Step Back, ordering, VRCFury
Case: both · Depth: references/debugging.md
Requires: the failing project in the user's Unity Editor; a static audit (P-17) already done for code-side causes.

1. Wrong final output: run a manual build
   (AvatarProcessor.ManualProcessAvatar) and inspect the persisted
   controllers, clips, and avatar clone; trace unexpected objects
   through the ObjectRegistry.
2. Unknown failing point: step through the build with the
   single-step debugger (Debugging); remember Step Back replays
   from the start rather than undoing.
3. Behaviour differs between a VRChat build or Play mode and a
   plain build: check the split VRCSDK pipeline (Framework, VRChat
   build context) before suspecting the pass itself.

## P-21 Build a component that adds parameters and menu items
Keys: menu generator component, adds parameters, menu items, ModularAvatarParameters, ModularAvatarMenuItem, provider, emit, presence check
Case: 2 only · Depth: Expression Parameters and Expressions Menu (Components that add parameters and menu items); Modular Avatar Integration (Emitting Modular Avatar components)
Requires: P-07 (the component's parameters supplied by a provider); parameter names and menu structure decided (P-12 and P-13 carry the declaration and menu rules).

1. Make the component editor-only; it holds the definition only and
   generates nothing itself.
2. Supply its parameters through a provider (P-07), so validation sees
   them in Generating before anything is emitted.
3. In Generating, ordered before Modular Avatar: for each component,
   add one ModularAvatarParameters to the component's own GameObject
   declaring the parameters the controls bind (P-12), and build the
   MenuItem hierarchy as child GameObjects of that GameObject (P-13,
   hierarchy route) — the placement fixes both the rename scope and
   the grouping of the items.
4. Skip a component whose GameObject already carries the generated
   components, so the edit-time pre-generation route composes with
   this one.

## P-22 Add animator output to the avatar's own controllers (case 1)
Keys: AnimatorServicesContext, virtual controller, VirtualAnimatorController, DependsOnContext, AnimLayerType, LayerPriority, ObjectPathRemapper
Case: 1 · Depth: Animator Generation (Merging into the avatar, case 1)
Requires: NDMF 1.7.0 or later (the version floor for AnimatorServicesContext); the target playable layer decided.

1. Put [DependsOnContext(typeof(AnimatorServicesContext))] on the
   pass; in Execute, read the context with
   ctx.Extension<AnimatorServicesContext>().
2. Take the target VirtualAnimatorController from
   asc.ControllerContext.Controllers, keyed by
   VRCAvatarDescriptor.AnimLayerType for a playable layer.
3. Register parameters with SetParameter. Add layers with
   AddLayer(LayerPriority, name), choosing the priority against the
   layers other passes add.
4. Build states on the layer's StateMachine (AddState); virtualize
   every real Motion with CloneContext.Clone before assigning it;
   set the state machine's DefaultState.
5. Register newly created objects that animations will reference
   with asc.ObjectPathRemapper.
6. Assign nothing to the descriptor and clone no controller assets:
   NDMF commits the virtual controllers when the context
   deactivates.

## P-23 Verify the build on a real machine (batched)
Keys: real machine, verification, compile, clean project, AAO residue, preview, unconfirmed
Case: both · Depth: Debugging; P-17 step 5
Requires: the package through P-17; a Unity project per row. Run the rows as one batch at the end; a row not yet run is reported as unconfirmed, never as passed.

1. Compile: import the package into a clean Unity 2022 project; it
   compiles with no errors. Repeat in a project without Modular
   Avatar (the case-1 environment) and, for case 2, with it; for an
   optional integration (P-16), both.
2. Build per case: run a manual build (P-20) and inspect the
   persisted controllers — generated layers present, parameters
   registered, and no source components left on the clone.
3. Parameters with other plugins: build an avatar that also carries
   another plugin's parameters; check for name collisions and that
   the shared 256-bit budget holds.
4. Animator behaviour: the Write Defaults assumptions hold on the
   merged result; condition-gated transitions fire in both
   directions; a generated Direct Blend Tree layer evaluates its
   sub-trees.
5. Avatar Optimizer: build in a project with AAO installed; no
   unknown-component warning names the plugin's components, and none
   of them remains after the build.
6. VRChat build and Play mode: a VRChat build's output matches the
   plain build's, and Play mode shows the same behaviour.
7. Preview: the Scene View preview of the plugin's output matches
   what the build produces; a mismatch is a preview defect, not a
   build defect (Modular Avatar Integration, Preview).
