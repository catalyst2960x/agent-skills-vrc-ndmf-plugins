# Test Case Catalogue

The cases a plugin's test suite is assembled from. Each case
states its preconditions, its input, and an expected result
that is decidable statically, from the built avatar and the
captured reports. Priorities: **Required** — every plugin
carries the case; **Required when applicable** — carried when
the plugin has the feature the case addresses;
**Recommended** — added as capacity allows.

A case is named by its category and number (BASE-01)
throughout this skill — in the catalogue, in the templates,
and in citations. In this skill's files, the first use of a
category in a file carries its meaning with it (ND-04
(non-destructiveness)), because several categories — ORD
and ND in particular — cannot be guessed from the
abbreviation, and PASS reads as a verdict where it means
component processing.

Cases are picked by the plugin's type. A generating plugin
(one that generates AnimatorControllers or menus) takes BASE,
PASS, GEN, ERR, ORD, ND. A parameter-supplying plugin adds
PARAM. A reactive plugin adds PREV. A mesh-processing plugin
adds MESH and HIER. An editor-support plugin adds EDIT. PLAT
and LOGIC apply per feature.

## BASE — the test base itself

- BASE-01 (Required) A minimal avatar with no plugin
  component builds cleanly, with no reports.
- BASE-02 (Required) Isolation: a second test starts with
  the first test's objects gone from scene and assets.
- BASE-03 (Recommended) The avatar builder produces both
  platform variants (VRChat descriptor; generic avatar root).
- BASE-04 (Required) One representative case run through
  one pass directly and through the full pipeline produces
  the same generated output; a divergence is itself the
  finding.

## PASS — component processing

- PASS-01 (Required) A fully configured component
  generates its output on the build clone, and the source
  component is removed. No reports.
- PASS-02 (Required) A required input unset: nothing is
  generated, one Information report remains, no exception.
- PASS-03 (Required) A set-but-invalid value: nothing is
  generated, a NonFatal report remains, the build does not
  stop.
- PASS-04 (Required when applicable) A target outside the
  avatar: nothing is generated, a NonFatal report remains.
- PASS-05 (Required) Several components, one unset: the
  valid ones generate, the unset one only reports, and every
  source component is removed. One failure stops nothing else.
- PASS-06 (Required) Serialized data carrying null
  elements and out-of-range enum values is sanitized and
  processed without an exception. The degenerate form the
  case uses is the one that reaches the plugin: a null
  element in an array of an embedded serializable class is
  materialized as a default instance by the build copy and
  never arrives as null, while a null in an array of object
  references arrives as null.
- PASS-07 (Required when applicable) An out-of-range
  serialized value reaches the output clamped to its range;
  the clamp is in the generation, not only in the inspector.
- PASS-08 (Recommended) A mid-generation failure leaves no
  partial output and touches nothing else.

## GEN — generated output structure

- GEN-01 (Required for generating plugins) Two components
  producing the same layer name yield two layers, distinct by
  the deterministic numbering rule, in processing order. The
  case applies where several components generate into one
  shared controller; where each component generates a
  controller of its own, no shared namespace exists, no
  numbering rule arises, and the case does not apply.
- GEN-02 (Required for generating plugins) Every added
  layer's weight is 1 — the case that detects a weight
  assigned to a struct copy and never written back.
- GEN-03 (Required for generating plugins) A generated
  clip's curves match the target in binding (path, type,
  property) and in key values.
- GEN-04 (Required for generating plugins) Transitions
  carry the designed conditions: parameter, mode, and
  threshold per condition; a conjunction rides one transition,
  returns split per condition, as designed.
- GEN-05 (Required for generating plugins) Every parameter
  a transition or a driver references is declared on the
  generated controller, with the same name and type.
- GEN-06 (Required) The initial state is determined by the
  defaults: across target initial state × condition default,
  the default state and initial curves already show the
  outcome, without running anything.
- GEN-07 (Required for generating plugins) With Write
  Defaults mixed in the merge target, every generated state
  carries the designed Write Defaults value.
- GEN-08 (Required when applicable) A generated state
  behaviour is present on the built avatar's state, with its
  fields as configured — the case that detects a behaviour
  created by direct instantiation, which does not survive
  into the built avatar.
- GEN-09 (Required when applicable) Empty generation
  material (no clips, no targets) produces the designed
  result; where an empty transition clip is needed, the
  substitute (a dummy curve) is present.

## PARAM — parameters

- PARAM-01 (Required for supplying plugins) A supplied
  parameter is present while the component is effective and
  absent when it is disabled (for example by generated MA
  components already present). Both sides are cases.
- PARAM-02 (Required for supplying plugins) The supplied
  parameter's type, default, sync, and persistence match the
  component's settings.
- PARAM-03 (Required when applicable) A parameter the
  logic depends on is defined in Expression Parameters after
  the build when it was undefined, with the designed type,
  sync, persistence, and default; when it was defined, the
  existing definition is preserved.
- PARAM-04 (Required for supplying plugins) Supplied names
  survive a build in which parameter renaming runs: the
  plugin's own remapping is a no-op for its parameters.
- PARAM-05 (Required when applicable) Condition parameter
  types resolve by the priority: an existing parameter
  (including a component's derived names) by its type, a
  built-in by the catalogue's type, an unknown name as Float.
  An unknown name is not an error and does not skip the
  component.
- PARAM-06 (Required when applicable) Suggested names are
  generated per target, unique, and in a deterministic scan
  order; collisions are uniquified by the designed rule.
- PARAM-07 (Required when applicable) Suggestion
  collisions: same kind and same type is not a collision;
  anything else is. Derived names from one prefix are judged
  together.
- PARAM-08 (Required when applicable) A suggested
  assignment lands on the build copy only; the author's scene
  object keeps its empty field, and a field already filled is
  not overwritten.
- PARAM-09 (Required when applicable) Two assignments to
  one target: the first applies, the second reports NonFatal.
  A target outside the avatar reports NonFatal and is skipped.
- PARAM-10 (Recommended) When Expression Parameters, the
  FX controller, and a provider disagree on a default, the
  same default wins under both processing orders.

## ERR — error reporting

- ERR-01 (Required) The three problem classes report at
  their fixed severities: unset is Information, set-but-
  invalid and contention are NonFatal, and no case produces
  an Error. The plugin never stops the build.
- ERR-02 (Required) Each report is identifiable by its
  source: the captured context exposes the reporting plugin
  and pass, and the case asserts them. Context objects
  attached when reporting appear in the build report window
  but are not retrievable from the captured context, so a
  case does not assert on them.
- ERR-03 (Required when applicable) Two components driving
  one target report NonFatal, and the winner is the designed
  one; the same target twice inside one component reports the
  same way.
- ERR-04 (Recommended) Report counts match problem counts:
  several components with the same problem produce one report
  each, neither merged nor duplicated.

## ORD — ordering and interoperation

- ORD-01 (Required for generating plugins) After the full
  build, the generated layer is in the FX controller, with
  its generated name and content.
- ORD-02 (Required when applicable) Through an Avatar
  Optimizer build, the generated layer survives and its
  references point into the optimized hierarchy.
- ORD-03 (Required) When restructuring (an armature merge)
  moves the target, the generated curves point at the moved
  target: resolution happens at the point of use, not from a
  registration-time snapshot. The expected path is resolved
  from the built hierarchy — the merge renames the moved
  object — not written from the scene-side path.
- ORD-04 (Required) The same output is generated for a
  target directly under the avatar root and for one several
  levels down; position dependence is a defect.
- ORD-05 (Recommended) Reordering the components changes
  nothing in the generated set, apart from the parts the
  design makes order-defined (name numbering).
- ORD-06 (Required) The plugin's declared pass ordering
  (before Modular Avatar, before Avatar Optimizer) is the
  order the build runs, and the generation that depends on it
  holds.
- ORD-07 (Required when applicable) A reparent by another
  system between generation and tracking changes chain
  membership: moved under a toggled ancestor, the tracked
  component's built layer carries the gained ancestor's
  proxy in its forward conditions; moved out from under
  one, the lost ancestor's proxy is absent. The conditions
  follow the built hierarchy, not the author-side one.

## ND — non-destructiveness

- ND-01 (Required) The author's scene components keep
  every field value across a build; writes land on the build
  clone only.
- ND-02 (Required) The author's controller and menu assets
  are unchanged by a build; generation and merging work on
  copies.
- ND-03 (Required) Components outside the plugin's scope,
  and what they drive, are unchanged.
- ND-04 (Recommended) Every generated asset and object is
  identifiable as the plugin's own output, not mixed into the
  author's assets.

## PREV — preview

- PREV-01 (Required for reactive plugins) The preview
  shows both the effective and the ineffective state, matching
  the build result for each.
- PREV-02 (Required for reactive plugins) Preview and
  build output agree under the same conditions.
- PREV-03 (Required when applicable) A preview value copy
  is equal to its source on creation and detected as different
  after either side changes; deep values are not misjudged
  through shared references.
- PREV-04 (Required) Toggling the preview leaves the
  scene's original values unchanged.

## PLAT — platform and runtime

- PLAT-01 (Required when applicable) A feature whose
  mechanism differs per platform (animator-driven on VRChat,
  statically applied on generic) shows the designed form on
  both platforms; the difference itself is the case.
- PLAT-02 (Required when applicable) A platform-filtered
  component is disabled off its platform and effective on it;
  mixed include/exclude produces the designed report.
- PLAT-03 (Recommended) PlayMode is a final check on
  representative cases whose structure editor tests already
  fixed; structure is not re-verified there.

## MESH — mesh processing

- MESH-01 (Required for mesh plugins) Against a test mesh
  whose vertex positions and blendshape effects are known,
  every primitive's selection or removal matches the expected
  result — a full comparison, not sampled points.
- MESH-02 (Required for mesh plugins) Degenerate inputs
  (an empty mesh, everything selected, a zero-scale bone)
  produce the defined result, with no exception and no NaN.
- MESH-03 (Required when applicable) Surviving vertices
  keep every data channel (all UV channels, colours).
- MESH-04 (Required for mesh plugins) The author's mesh
  asset is unchanged; processing works on a copy.

## HIER — hierarchy restructuring

- HIER-01 (Required when applicable) Reference cycles
  (direct, indirect, across features) are detected and
  reported; the same shapes without a cycle succeed.
- HIER-02 (Required when applicable) A chain that matches
  only partially is not merged; the target stays as it was.
- HIER-03 (Required when applicable) Objects the retention
  rules protect (end bones, animated objects) survive a
  cleanup; only unprotected objects are removed. Both sides
  are cases.

## MENU — menus

- MENU-01 (Required when applicable) A generated menu
  installs by the designed route: appended through an
  installer when one is present, decomposed onto menu items
  with a visiting installer when none is.
- MENU-02 (Required when applicable) Cyclic and chained
  menu and installer references terminate, with the designed
  result.
- MENU-03 (Required when applicable) Deduplication applies
  per installation; installing the same menu several times
  keeps the installation count.
- MENU-04 (Required when applicable) A generated submenu
  installs as one submenu control in the built menu, whose
  sub-menu carries the designed child controls — names,
  types, parameters, and values.
- MENU-05 (Required when applicable) A control installed
  into a group, or under a parent control, lands under that
  parent in the built menu.

## EDIT — editor operations

- EDIT-01 (Required for editor-support plugins) An
  inspector or window operation is undoable; applied to a
  prefab instance, it remains as an override.
- EDIT-02 (Required when applicable) A setup operation run
  twice changes nothing the second time.
- EDIT-03 (Required when applicable) Inference from names
  does not pick a wrong match on confusable input; with
  several candidates, the designed rule (first match) decides.

## LOGIC — pure logic

- LOGIC-01 (Required when applicable) Degenerate inputs
  (coincident values, zero, interval endpoints) and special
  values (infinity, signed zero) return the defined result,
  with no exception and no non-finite contamination.
- LOGIC-02 (Recommended) A random operation sequence
  preserves the structure's invariants throughout.

## Cross-cutting viewpoints

| Viewpoint | What it fixes |
| --- | --- |
| Tolerance of absence | Null, unset, and empty inputs produce the defined result (no generation, an Information report), never an exception |
| Contention is visible | Same-name, duplicate, and conflicting settings are reported, with a designed winner |
| Priority without chaining | Rule application order is deterministic, and a rule's result is never re-fed to the same rule |
| Boundaries | Range ends (clamping), both sides of a threshold, zero/one/many |
| Order independence | A configuration that should succeed succeeds under any component or processing order |
| Cycles terminate | Reference cycles are detected, or processing terminates by a designed cut |
| Source assets are invariant | Rewriting happens on copies; inputs and author settings do not change |
| References follow moves | After restructuring, paths and masks still point at their targets |
| Optimization, both sides | An optimization has a case where it applies and a case where it must not |
| Initial state is determined | Defaults and conditions fix the initial state at build time |
| Round-trip limits | A feature that restores settings from output is checked over what it can and cannot restore, separately |
| Regressions are fixed in place | A past defect keeps a dedicated case tied to its source |
| Cross-platform consistency | Which attribute differences are inconsistencies is fixed per attribute |
| Defaults have one source | Among Expression Parameters, the controller, providers, and automatic assignment, the winning default does not depend on order |
| Mathematical boundaries | Degenerate and special values outrank ordinary inputs as cases |
| Retention rules | What a cleanup keeps is cased on both sides: kept, and removed |
| Inference does not misfire | A heuristic is cased on confusable input, not only on correct matches |
| Idempotence | An edit operation run twice changes nothing the second time |

## Regression practice

When a defect is reported, the reproducing case is added
first and watched failing; the fix is complete when the case
passes. The case carries its source — an issue number or a
date — in its name or a comment. A defect found on a real
machine is restated as an editor case as far as it can be;
the part that cannot (client behaviour) is marked out of
scope in the case, and its static part (the generated
structure) is still cased. When a dependency version rises,
the whole suite runs, and each failure is triaged as a
product-side change or a dependency-side change before
anything is edited.
