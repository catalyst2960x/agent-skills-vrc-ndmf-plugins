---
name: agent-skills-vrc-ndmf-unit-tests
description: "Unit testing VRChat avatar plugins built with NDMF (Non-Destructive Modular Framework) and Modular Avatar (MA): NUnit editor tests that assert on generated animator controllers, parameters, and build reports, with a test base, representative cases, and a case catalogue."
---

# NDMF Plugin Unit Tests

Verify an NDMF plugin's requirements as NUnit editor tests, for
Unity 2022 LTS: build an avatar with the plugin's components on
it, run the real build, and assert on what the build produced —
the generated AnimatorControllers, clips, parameters, and
ErrorReport entries, all read statically from the built avatar.
Behaviour that exists only in the VRChat client (contact
response, playback, state transitions at runtime) is not
observable from build output and is out of scope for these
tests; it stays with manual verification.

The plugin under test follows the same two dependency cases as
the plugin development skill — NDMF only, or NDMF + Modular
Avatar — and the test assembly references what the plugin
references. Dependency versions: NDMF 1.15.*, Modular Avatar
1.19.*.

This skill is the companion to the NDMF plugin development
skill: that skill owns how the output is generated, this one
owns how the generation is verified. Load it before writing or
extending a plugin's tests.

## Table of Contents

| Section | Content | Reference |
| --- | --- | --- |
| [Common Mistakes](#common-mistakes-never-list) | Failure modes to design out | This file |
| [Design Decisions](#design-decisions-before-starting) | Choices to settle before writing tests | This file |
| [Core Workflow](#core-workflow) | The six steps, in order | This file |
| [Test Infrastructure](references/test-infrastructure.md) | The test base: tracked cleanup, the minimal avatar, the two build entries, error capture, FX inspection, and running the tests (requirements, the disabled distribution form, the Test Runner) | `references/test-infrastructure.md` |
| [Test Case Catalogue](references/test-case-catalogue.md) | The case categories by plugin type, the case list with priorities, the cross-cutting viewpoints, regression practice | `references/test-case-catalogue.md` |
| [Procedure Index](references/procedure-index.md) | The procedures T-01 to T-10: setting up, selecting and writing cases, verifying a case, regression, running the suite in Unity, triage, and close-out | `references/procedure-index.md` |
| [Anti-Patterns](references/anti-patterns.md) | Test-code patterns that pass while the product is wrong, or fail while it is right | `references/anti-patterns.md` |
| [Diagnosing a Failing Case](references/failure-triage.md) | The triage order for a failing case: base, isolation, the pass/pipeline bisection, the report, product against dependency | `references/failure-triage.md` |
| [Cheatsheet](CHEATSHEET.md) | The test base surface and the case category map | `CHEATSHEET.md` |
| [Templates](templates/README.md) | Worked sample code for mechanisms the references state as rules | `templates/` |

## Common Mistakes (NEVER List)

| # | NEVER do this | Why it hurts | Use instead |
| --- | --- | --- | --- |
| 1 | Assert on the author's scene objects after a build | The build runs on a clone; the scene objects only ever show the pre-build state, so the assertion passes whatever the build did | Read the built avatar the build entry returns; compare scene objects only in the non-destructiveness cases, where unchanged is the claim (Test Case Catalogue, ND — non-destructiveness) |
| 2 | Create test objects without registering them for teardown | Residue from one test changes the next test's scene and assets, and failures become order-dependent | Track every created object and asset; teardown destroys all of them and deletes the temporary asset folder (Test Infrastructure) |
| 3 | Assert presence only — a layer exists, a clip exists | A layer serialized at weight 0, or a curve with the wrong value, passes a presence check | Compare values: layer weights, curve bindings and key values, transition modes and thresholds, parameter types and defaults (Test Case Catalogue, GEN — generated output) |
| 4 | Replace the pipeline with mocks for an integration claim | Ordering against other passes and other systems' restructuring is part of the claim; a mock removes exactly that | Run the real pipeline through the manual build entry; use a direct BuildContext only for cases whose claim is one pass in isolation (Test Infrastructure) |
| 5 | Write a case for behaviour only the client shows | Contact response and runtime transitions are not in the build output; the case either asserts nothing or asserts a proxy | Assert the static structure that produces the behaviour; leave the behaviour itself to manual verification, stated as out of scope in the case |
| 6 | Fix a defect first and add its case afterwards | Without a failing case, the fix is unverified and the defect returns silently | Add the reproducing case first, watch it fail, then fix; the case stays as a regression case with its source recorded (Test Case Catalogue, Regression practice) |
| 7 | Assert on report message text | Message text is a localization surface; it changes without the defect changing | Assert on the captured ErrorContext severities: unset input is Information, a set-but-invalid value and a contention are NonFatal, and the plugin assigns no Error |

## Design Decisions Before Starting

- **Static scope.** A test asserts only on what the build
  output shows statically: controllers, clips, parameters,
  reports, and the built hierarchy. What runtime state the
  client reaches is not a test result. A requirement that
  cannot be restated statically is split: its static part
  becomes a case, its runtime part stays manual.
- **Two layers of test.** A pass-unit case runs one pass in a
  direct BuildContext and isolates a rule. An integration case
  runs the full pipeline and sees ordering and other systems'
  effects. The representative cases are integration cases; a
  pass-unit case is added where a rule needs isolation, and
  one representative case is run both ways to catch divergence
  between the layers.
- **Severity is the report contract.** The plugin's error
  policy fixes the severity per problem class (Test Case
  Catalogue, ERR — error reporting): cases assert the severity of each expected
  report and that no Error appears. Counts alone do not fix
  the contract, and message text is not asserted.
- **Fixtures in two forms.** A simple hierarchy is built in
  code; a realistic avatar shape is a prefab fixture. Common
  fixtures (the minimal avatar in its platform variants) live
  in one folder, and a case picks the form with nothing extra
  for its intent.
- **Regression cases are permanent.** A defect's case is
  written before the fix, carries its source (an issue number
  or a date) in its name or a comment, and is not removed when
  the surrounding code changes. A case whose reason is unknown
  is a case that gets deleted, so the reason is recorded.
- **The test code is itself kept sound, by five properties.**
  A case is shown able to fail (detection); its expected
  values correspond to the design — derived from the designed
  behaviour and the fixture's inputs, never copied from the
  build's current output; its verdict is reproducible, the
  same alone and in the suite; its subject is valid — the
  real pipeline for an integration claim, a stand-in only
  under a stated contract; and the case itself stays simple,
  stating only its fixture and assertions on the shared base.
  Tests of tests do not terminate, so the last property
  carries the others: a case simple enough to verify by
  reading, checked once by the verification procedure (T-06).

## Core Workflow

Run the steps in order.

1. **Verify versions and APIs.** The test-facing APIs are
   versioned like the rest: confirm the BuildContext
   constructors, the manual build entry, and
   ErrorReport.CaptureErrors against the installed NDMF
   before use.
2. **Lay out the test assembly.** An Editor-only assembly
   under UNITY_INCLUDE_TESTS, referencing NDMF, the plugin's
   own assemblies, and Modular Avatar when the plugin is
   dependency case 2. In a distributed package the test folder
   name carries a trailing `~`, so users do not import the
   tests (Test Infrastructure).
3. **Write the test base** from the infrastructure template:
   tracked cleanup, the minimal avatar, the two build entries
   with error capture, and the FX inspection helpers.
4. **Pick the cases by plugin type** from the catalogue and
   implement the required set first: the base cases, the
   pass cases, the generated-output cases for a generating
   plugin, the parameter cases for a supplying plugin, the
   report and non-destructiveness cases for every plugin.
   Each new case is verified before it counts (T-06).
5. **For each defect, case first.** Reproduce it as a failing
   case, fix, and keep the case as a regression case.
6. **On a dependency version change, run everything.**
   Triage each failure as a product-side change or a
   dependency-side change before editing either.

The work is complete when every required case for the
plugin's type is implemented and the full suite passes with
them included (T-10); a failing case is triaged (Diagnosing
a Failing Case) and resolved, or recorded as a stated gap,
and the run repeats until that condition holds.

## Sources

| Product | What was taken from it | License |
| --- | --- | --- |
| bdunderscore/ndmf | The test-facing surface: BuildContext, AvatarProcessor's manual build entry, ErrorReport.CaptureErrors | MIT |
| bdunderscore/modular-avatar | The build environment the tests exercise: the merge and reactive systems whose output the cases assert on | MIT (Editor/images excluded from redistribution) |
