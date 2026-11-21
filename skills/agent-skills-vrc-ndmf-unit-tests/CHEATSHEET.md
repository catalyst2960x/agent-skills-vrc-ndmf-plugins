# NDMF Plugin Unit Tests Cheatsheet

Quick reference. This file is the canonical home of two things:
the test base surface, and the case category map. Everything
else here is a pointer to the reference that owns the fact —
facts are maintained in one place only. The reasoning is in
[SKILL.md](SKILL.md); the full code is in the templates.

## Test base surface

| Member | What it gives a case |
| --- | --- |
| `Track(obj)` | Registers a created object for teardown; returns it |
| `TearDown()` | Destroys every tracked object; deletes the temporary asset folder |
| `CreateMinimalAvatar(name)` | An avatar root with an Animator, a descriptor with populated playable layers (FX carries an empty controller), and empty Expression Parameters |
| `CreateChild(parent, name)` | A tracked child GameObject under a parent |
| `CreateContext(root)` | A direct BuildContext over a clone, for running one pass |
| `Build(root, out errors)` | The full pipeline through the manual build entry; returns the built avatar and captures the reports |
| `FindFxController(built)` | The built avatar's FX AnimatorController |
| `FindLayer(controller, name)` | A layer by name, or null |
| `HasLayer(controller, name)` | Whether a layer of that name exists |
| `AssertLayerAbsent(controller, name)` | Fails if the layer exists; the message lists the layers present |
| `RequireLayer(controller, name, context)` | A layer by name; on failure the message lists the layers present, plus the context (after a build, the report summary) |
| `PathOf(obj)` | The avatar-relative path from the object's real names in the built hierarchy |
| `ReportSummary(errors)` | One line per report (severity, pass, message), for assertion messages |
| `FindState(layer, name)` | A state by name, or null |
| `RequireState(layer, name)` | A state by name; on failure the message lists the states present |
| `TransitionSummary(layer)` | Every transition with its conditions, for assertion messages |
| `FindClip(state)` | The state's motion as an AnimationClip |

## Case checklist

Copy this list for each case written; the selection comes
from the case catalogue (T-03).

- [ ] The fixture is minimal and fully tracked (T-02)
- [ ] The build entry matches the property: direct context
      for a pass-level property, the pipeline otherwise (T-04)
- [ ] Assertions compare values, not presence alone (T-04)
- [ ] A report assertion is paired with the generation
      assertion for the same build (T-05)
- [ ] The case fails on the defective behaviour it guards
      (T-07)
- [ ] The case is verified: it fails under perturbation at
      its own assertion, its expectations trace to the
      design, and its verdict is stable alone and in the
      suite (T-06)

After the checklist, the case is complete. Where the
requirement behind it was split (SKILL.md, Static scope),
the runtime side is verified by the companion method
(Runtime Verification, references); it is not a checklist
item and does not reopen the case.
- [ ] The full suite passes with the case included (T-10)

The severity assertion idiom: after a build, the expected
report is found by severity, and Error is absent — the
absent-Error check comes first, with the summary as its
message, separating a pass that threw from a designed skip
before any output assertion runs.

```csharp
Assert.IsFalse(errors.Any(e => e.TheError.Severity == ErrorSeverity.Error), ReportSummary(errors));
Assert.IsTrue(errors.Any(e => e.TheError.Severity == ErrorSeverity.Information), ReportSummary(errors));
```

## Case category map

| Category | Asserts on | Plugin types |
| --- | --- | --- |
| BASE | The test base itself: clean build, isolation, both build layers agreeing | All |
| PASS | Component processing: generation, severity per problem class, sanitization, removal | All |
| GEN | Generated structure: names, weights, curves, transitions, declarations, initial state | Generating |
| PARAM | Supply, attributes, definition guarantee, type resolution, suggestions and assignments | Supplying |
| ERR | Severity assignment, report context, contention, report counts | All |
| ORD | Ordering against MA and AAO, following restructuring, position generality | All |
| ND | The author's components, assets, and out-of-scope objects unchanged | All |
| PREV | Preview states, preview/build agreement, value copies, scene invariance | Reactive |
| PLAT | Per-platform mechanism differences, platform filters, PlayMode as final check | Per feature |
| MESH | Full-primitive comparison on known meshes, degenerate inputs, channel retention | Mesh |
| HIER | Cycle detection, partial matches, retention rules | Restructuring |
| MENU | Installation routes, cycle termination, deduplication scope, submenu and group structure | Menu-generating |
| EDIT | Undo, idempotence, inference on confusable input | Editor-support |
| LOGIC | Degenerate and special values, invariants under random operations | Pure-logic holders |
