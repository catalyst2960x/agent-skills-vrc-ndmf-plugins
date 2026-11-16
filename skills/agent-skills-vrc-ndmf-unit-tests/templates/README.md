# Templates

Worked sample code for mechanisms the references state as
rules. Each file is self-contained: the situation it applies
to, then the code, tagged with where the code runs. The code
was written for these samples from the mechanism descriptions,
trimmed to the mechanism; it is not compiled or verified in
Unity in this form.

The case files are organized one per feature area, in the
manner of Modular Avatar's own test suite. Cases whose
fixture is a product-class implementation run against a
document-local stand-in: a class that exists only in the
file, whose contract the file's introduction states — the
adopting product substitutes its own component and its real
contract. Each code block names the stand-ins it uses in a
comment at its head, marked as stand-ins to implement.
Which files a plugin takes follows its type:

| Plugin type | Files |
| --- | --- |
| Every plugin | test-base, base-verification, build-layer-agreement, component-processing, error-reporting, non-destructiveness |
| Generating | + generated-output, ordering-interoperation |
| Menu-generating | + menu-generation |
| Parameter-supplying | + parameter-supply |
| Reactive | + preview |
| Mesh-processing | + mesh-processing, hierarchy-restructuring |
| Editor-support | + editor-operations |
| Per feature | platform-variants, pure-logic, playmode |

| File | Content |
| --- | --- |
| [base-verification.md](base-verification.md) | The base's own cases: the minimal avatar builds clean, and teardown leaves the scene and assets as they were. |
| [build-layer-agreement.md](build-layer-agreement.md) | One representative case run at both layers of test, the pass alone and the full pipeline, producing the same generated output. |
| [component-processing.md](component-processing.md) | How a pass treats the components it finds: generation on a valid configuration, Information for an unset input, NonFatal for an outside target, all components processed and removed, null entries skipped, out-of-range values clamped at generation, and a mid-generation failure leaving no partial output. |
| [editor-operations.md](editor-operations.md) | Undoable operations, prefab instance overrides, idempotent setup, and name inference decided by rule on confusable input. |
| [error-reporting.md](error-reporting.md) | Contention between two components on one target: the first generates, the second reports NonFatal and generates nothing. |
| [generated-output.md](generated-output.md) | The structure of the generated layer: weight 1, name numbering, clip curves with their values, transition conditions and their declarations, the default state the defaults determine, Write Defaults uniformity, attached behaviours surviving the build. |
| [hierarchy-restructuring.md](hierarchy-restructuring.md) | Reference cycles detected and reported, partial chains left unmerged, and protected objects surviving a cleanup. |
| [menu-generation.md](menu-generation.md) | Installation by the designed route, cyclic installation references terminating, deduplication scoped per installation, and the generated submenu and group structure — all asserted on the built menu. |
| [mesh-processing.md](mesh-processing.md) | The processed mesh compared in full against a code-built fixture, degenerate meshes, surviving data channels, and the author's mesh unchanged. |
| [non-destructiveness.md](non-destructiveness.md) | The author's scene component, the author's controller asset, and out-of-scope components unchanged across a build, and generated output identifiable as the plugin's own. |
| [ordering-interoperation.md](ordering-interoperation.md) | Generated curves following an armature merge to the target's post-merge path, the same generation for a root-level and a nested target, the generated set unchanged under component reordering, and chain membership following a reparent between generation and tracking (stated, not coded). |
| [parameter-supply.md](parameter-supply.md) | Supply following the effectiveness gate, supplied-parameter attributes, the Expression Parameters definition guarantee, condition parameter type resolution by priority, supplied names surviving renaming, suggestion uniqueness and collisions, recorded assignments landing on the build copy only, and a three-way default disagreement resolving the same under both processing orders. |
| [platform-variants.md](platform-variants.md) | A mechanism taking its designed form on each platform, and a platform-filtered component disabled off its platform. |
| [playmode.md](playmode.md) | A PlayMode final check: the generated controller running, a driven parameter entering the designed state, the state's behaviour firing on entry. |
| [preview.md](preview.md) | Preview and build agreement for the effective and ineffective states, value copies detecting changes on either side, and the scene left unchanged. |
| [pure-logic.md](pure-logic.md) | Degenerate and special values returning the defined result, and a seeded random sequence preserving the structure's invariant. |
| [test-base.md](test-base.md) | The editor-test base: tracked creation and teardown, a minimal avatar, a direct BuildContext and the full pipeline as build entries, error capture, and FX inspection helpers, with the test assembly definition. |
