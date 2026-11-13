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
  (case 2 adds Modular Avatar). At any other version, re-verify
  before applying a procedure
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
  optional integrations (P-18), and test assemblies referencing the product. All
  three are impossible without named assemblies.
- **An explicit reference graph.** The asmdef references list is the
  assembly-level form of the dependency closure (P-19): in the predefined
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
   .BeforePlugin("com.anatawa12.avatar-optimizer") (P-20).
3. A pass that transforms the build clone (mesh, hierarchy, or
   component fields) goes in Transforming, sequenced by what it
   consumes and what it feeds: after the passes that produce what
   it transforms, before the passes that consume the result. A pass
   that emits another directive component runs before that
   component's applying pass.
4. Register each pass in one of the two forms (Framework): an
   inline handler with .Run("<name>", handler) — the name is what
   build logs and error reports show — or a Pass<T> subclass by
   instance, .Run(new MyPass()), which carries its own display
   name and declares its extension contexts with
   [DependsOnContext].
5. Keep Configure() free of build work: it only declares.

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

## P-05 Resolve parameter types from the build-time set
Keys: ParameterInfo, EffectiveName, GetParametersForObject, SubParameters, type resolution, built-in, duplicate, first
Case: both · Depth: Parameters; Error Reporting (Validation policy)
Requires: P-04 (sanitized components); if this plugin itself generates the parameters being resolved, P-07 in place first — otherwise they are invisible to ParameterInfo.

1. Build the index: ParameterInfo.ForContext(ctx)
   .GetParametersForObject(ctx.AvatarRootObject), grouped by
   EffectiveName, first entry wins on duplicates. Expand each entry
   through SubParameters(): a PhysBones prefix entry stands for its
   five derived names, each carrying its own type.
2. Collect the names the component needs (conditions, pre-conditions,
   feature parameters), deduplicated, empty names skipped.
3. Resolve each name's type from, in order: the index (step 1), then
   the platform's built-in catalogue (VRChat Avatar Platform,
   Built-in Animator Parameters). A name no source types is not an
   error and is never reported: it registers as Float (Parameters).
   Names are accepted as entered; the set is a type source, not a
   gate.
4. Structural defects = Error (Error Reporting, Validation policy).
   Never read the VRCExpressionParameters asset for this (NEVER #2).

## P-06 Report a problem to the author
Keys: error reporting, ErrorReport, SimpleError, Localizer, WithContextObject, NonFatal, ErrorSeverity
Case: both · Depth: Error Reporting
Requires: the error policy decided (SKILL.md, Error policy) and the severity policy (Error Reporting, Validation policy); a decision whether the product ships a localization asset or uses the passthrough localizer.

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
   type converting as Float. Register the reported type: a different
   type compiles, but VRChat then converts the value at runtime and
   P-09's threshold arithmetic stops describing the behaviour (VRChat
   Avatar Platform, Parameter Type Conversion). One standing
   exception: the Direct Blend Tree shape registers Float on the
   controller for the parameters it blends on, because blend trees
   blend on floats and the conversion is exact for Bool and Int
   sources (Animator Generation, Direct Blend Tree layers).
2. Register each name once per controller (track the added set).
3. Neither framework provides this conversion; do not call a conversion
   helper from a library the package does not depend on (the
   dependency-closure check, SKILL.md step 7).

## P-09 Generate a condition-gated layer
Keys: layer generation, idle, active, intermediate state, transition, condition reversal, reverse, hasExitTime, defaultWeight, writeDefaultValues, empty clip
Case: both · Depth: Animator Generation
Requires: the trigger model decided (SKILL.md, Trigger model) — idle/active, optional intermediate stage, return style; P-05 passed for the component; P-08 for the parameters its conditions name.

1. Build the layer per Animator Generation (The layer/state pattern):
   one controller per pass, the component's layer or layers, the
   idle/active
   states and the shared empty clip as specified there.
2. Transitions per Animator Generation (Transitions and condition
   reversal): the forward set as a single AND transition; the return
   as one transition per reversed condition, or the free return.
   Reversal and thresholds follow the table there, computed in the
   parameter's source type.
3. Empty condition set: the forward transition fires unconditionally
   and no return transitions exist. Confirm that is the intent.
4. Dependency case 1: there is no generated controller to hand over.
   Build the same layer content on the virtual controller instead
   (P-24): the states, transitions, and clips are unchanged; only the
   surface differs, and real Motions are virtualized through the
   CloneContext.

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
   matchAvatarWriteDefaults per the generated states' assumption —
   true for the layer/state pattern, false for a Direct Blend Tree
   layer, whose state must keep Write Defaults on.
2. The generating pass must already be ordered before Modular Avatar
   (P-03), or the merge is silently dropped.
3. Only after the merge component exists, DestroyImmediate every
   found source component instance, including ones that failed
   generation (P-20).

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
Case: 2 only (case 1 menus are out of scope) · Depth: Expression Parameters and Expressions Menu (Writing menus)
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

## P-15 Replace or edit an avatar asset in a pass
Keys: asset editing, mesh, texture, material, clone asset, ObjectRegistry, RegisterReplacedObject, asset saver, rewrite references
Case: both · Depth: Manipulation Techniques (Direct editing and the framework surface; Converting shared assets in a pass)
Requires: the target asset's owner resolved on the build-time clone. The edit itself is standard Unity API work and is not specified here; this procedure is the route the asset takes through the build.

1. Receive the asset from the build: read it from its component on
   the build-time clone, resolving through ObjectRegistry when an
   earlier pass may have replaced it.
2. Clone the asset before editing it; the author's asset is never
   edited. An asset the build itself generated is temporary and may
   be edited in place instead of cloned; the framework's
   IsTemporaryAsset check distinguishes the two. One de-duplicating
   mapping, keyed by the original asset, serves every clone the
   pass makes.
3. Edit the clone (mesh data, texture pixels, material values).
4. Rewrite every reference to the original through the mapping —
   renderer and component fields, and animation curves, live and
   animated alike — so no reference to the original remains.
5. Let the build's asset saver persist the clones (Framework).
   Register each replacement with
   ObjectRegistry.RegisterReplacedObject(original, clone) when
   another pass may already hold the original: a pass that resolved
   the asset earlier keeps the original unless the replacement is
   registered. When every consumer reads the asset from its
   component after this pass, the rewritten references suffice.

## P-16 Build a component that applies a build-time transformation
Keys: directive component, build-time transformation, Transforming, scope, nesting, silent, no animator output
Case: both · Depth: Manipulation Techniques (Direct editing and the framework surface); Framework (Phases and ordering)
Requires: the output route decided as the directive shape (SKILL.md, Output route): no animator output; the component carries settings only.

1. Author the component as a settings-only, editor-only tag
   (MonoBehaviour + IEditorOnly, one instance per object): it holds
   the author's settings and no build logic.
2. State the component's scope and nesting rule in the design:
   which objects an instance affects (its own object, its
   descendants, or what its ancestors carry), and when instances
   nest, which one wins (nearest ancestor, a cascade, or nesting
   rejected). A design without a nesting rule is incomplete.
3. Register the applying pass in Transforming (P-03), sequenced by
   what it consumes and what it feeds; a pass that emits another
   directive component runs before that component's applying pass.
4. In the pass, collect the instances (P-04), resolve what they
   reference on the build-time clone, and apply the transformation
   to the clone only. Asset edits follow P-15.
5. An instance whose target is absent or inapplicable on this
   avatar is skipped silently: inapplicability is not a defect and
   draws no report (Error Reporting, Validation policy). Defects in
   the author's own settings are reported as usual.
6. Remove every found instance before the Optimizing phase: the
   applying pass destroys them as it consumes them, or a removal
   pass the plugin registers does. No other plugin removes them.
7. Preview follows the preview story (P-17): a transformation that
   changes what a preview renders gets a filter that re-applies
   the transformation to the proxies; a parameter-only change
   normally has none.

## P-17 Give a visible effect a preview story
Keys: preview, IRenderFilter, PreviewingWith, RenderGroup, nadena.dev.ndmf.preview, Reactive, visible effect
Case: both · Depth: Modular Avatar Integration (Preview); Framework (Preview system)
Requires: the preview story decided before building (SKILL.md, Preview story); for the filter route, the proxy-renderer model in Framework (Preview system).

1. Decide before building (SKILL.md, Preview story). The axis is
   what the effect changes: a transformation that changes what a
   preview renders (geometry, transform structure) gets a preview;
   a change to rendering, physics, or functional parameters alone
   normally does not. Routes: inherit MA's preview by expressing
   the effect as a Reactive Component; attach an IRenderFilter to
   the plugin's own pass with PreviewingWith, re-applying the
   pass's transformation to the preview proxies; or state
   explicitly that there is no preview (non-visual output such as
   audio).
2. IRenderFilter route: implement the filter contract (Modular Avatar
   Integration, Preview) — target groups, the node that applies the
   effect to proxy renderers, the enable gate — including the CanEnableRenderers flag
   when the filter may enable disabled renderers.
3. Preview runs on the live scene outside builds: touch only the proxy
   renderers and materials given, dispose what the node creates, never
   mutate the author's objects.

## P-18 Use Modular Avatar APIs optionally from a case-1 plugin
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

## P-19 Audit and hand over
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
5. Real-machine verification is a separate batch; rows not
   yet run stay unconfirmed in the handover.

## P-20 Remove generated components before Avatar Optimizer runs
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

## P-21 Diagnose a build problem
Keys: debugging, ManualProcessAvatar, single-step, Step Back, ordering, VRCFury
Case: both · Depth: references/debugging.md
Requires: the failing project in the user's Unity Editor; a static audit (P-19) already done for code-side causes.

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

## P-22 Build a component that adds parameters and menu items
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

## P-23 Draw a component's inspector and drawers
Keys: inspector, CustomEditor, PropertyDrawer, SerializedProperty, ReorderableList, editor UI, suggestions
Case: both · Depth: references/inspectors-and-drawers.md
Requires: the component's serialized fields fixed; the validation policy the display must mirror (Error Reporting).

1. Put one CustomEditor per component type in the product's Editor
   assembly. A field type shared by several components gets one
   PropertyDrawer, shared by all of them (Inspectors and Drawers,
   Placement and shape).
2. Draw and write only through SerializedObject and
   SerializedProperty — Update, draw, ApplyModifiedProperties —
   and change arrays only through the serialized array property,
   drawn as reorderable lists.
3. Keep one layout across the product's components: the same
   section order, the same list editing, the same row for the
   same data.
4. Offer derivable values as explicit actions that write through
   the serialized property; never fill or overwrite silently, and
   keep name fields free text with suggestions beside them.
5. Mirror the build's validation as help boxes at the same
   severities; the inspector fixes nothing silently and blocks
   nothing.
6. Edit-time generation actions call the same code the build pass
   runs, with the same presence check.

## P-24 Add animator output to the avatar's own controllers (case 1)
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
   deactivates. The layer's content — states, transitions, condition
   logic — follows the same procedures as on a generated controller
   (P-09, P-10).

## P-25 Package for VPM distribution
Keys: VPM, VCC, package.json, vpmDependencies, legacyFolders, user packages, manifest, zipSHA256
Case: both · Depth: references/vpm-packaging.md
Requires: the package complete through P-19; the Assets path of any earlier Assets-based distribution, if one exists.

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

