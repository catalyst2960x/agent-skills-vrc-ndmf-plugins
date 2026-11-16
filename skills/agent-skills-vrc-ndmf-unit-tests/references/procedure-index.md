# Procedure Index

The procedures for writing and maintaining a plugin's unit
tests. Each entry states its steps and the traps its steps
exist to avoid. The case catalogue holds what to test; this
index holds how the work proceeds.

## T-01: Set up the test assembly and base

Steps:

1. Create the test assembly in the MA form: Editor-only,
   UNITY_INCLUDE_TESTS, TestRunner references, nunit as a
   precompiled reference.
2. Copy the test base from the test-base template; set the
   temp asset path and the plugin's assemblies in the
   references.
3. Write the base verification cases (base-verification
   template) and run them before any product case.

Traps:

- An assembly that also compiles outside the test context
  ships test code in the product.
- A base whose own cases were never run: every later failure
  is ambiguous between the product and the fixture.

## T-02: Build a fixture

Steps:

1. Start from the minimal avatar; add only what the case
   reads.
2. Configure the component completely for a generation
   case; leave exactly the one input unset or invalid for a
   report case.
3. Track everything created; prefabs are instantiated per
   test from a shared asset, never edited in place.

Traps:

- A fixture carrying more than the case reads hides which
  input the result depended on.
- An untracked object survives teardown and changes the
  next test.
- A null element written into a serialized array of an
  embedded class does not reach the plugin as null — the
  build copy materializes a default instance. A null in an
  object-reference array does arrive. Build the degenerate
  input in the form that arrives (Test Case Catalogue,
  PASS-06).

## T-03: Select the cases a change requires

Steps:

1. From the catalogue, take every case marked required for
   the plugin's type, and every case whose area the change
   touches.
2. For a defect fix, take the regression case first (T-07).
3. Record the selection; a required case not written is a
   stated gap, not an oversight discovered later.

## T-04: Write a case against generated output

Steps:

1. Build through the full pipeline, unless the property is
   pass-level (then the direct context, with the agreement
   case covering the difference).
2. Read the output from the built avatar: the FX controller
   from the descriptor's layers, then layers, states, clips
   by name.
3. Assert values — weights, curve values, condition modes
   and thresholds — not presence alone.

Traps:

- Reading the author's scene objects: the build wrote to
  the clone.
- A case that passes when the component was skipped:
  assert the generation happened, not only that no error
  was reported.

## T-05: Write a case against the build report

Steps:

1. Wrap the build in CaptureErrors.
2. Assert severities: the class of problem maps to
   Information or NonFatal, and no entry is an Error.
3. Assert the generation side of the same case in the same
   test: what was generated, or that nothing was.

Traps:

- Asserting message text: messages are diagnostics, the
  severity is the contract.
- Capturing around a direct context when the case is about
  the pipeline's report.

## T-06: Verify a case

Steps:

1. Make it fail. For a regression case, revert the fix; for
   a new case, perturb the designed behaviour in a scratch
   copy — the designed value, the threshold, the generation
   itself. Run the case and confirm it fails at the
   assertion that carries its claim, not in the fixture,
   the build, or teardown. Restore the perturbation.
2. Re-derive the expectations. Every expected value traces
   to the designed behaviour or to an enumerated fixture
   input; a value taken from the build's current output is
   replaced by the designed one.
3. Run the case alone, then in the full suite, then once
   more. The verdict is the same all three times.
4. Check the subject. An integration claim ran the real
   pipeline, and every stand-in in the fixture acts under
   the contract its template states.
5. Read the case once. It states its fixture, its build,
   and its assertions, and nothing else; logic beyond that
   belongs in the base or in the product.

Traps:

- Perturbing the fixture instead of the behaviour: the
  case fails because its input changed, which shows nothing
  about detecting the defect.
- Accepting any failure as detection: a case that fails in
  its setup "detects" every defect, and therefore none.
## T-07: Record a regression case for a defect

Steps:

1. Reproduce the defect as a fixture: the smallest
   configuration that produced it.
2. Write the case so it fails on the defective behaviour
   and passes on the designed behaviour; the assertion names
   the designed value.
3. Keep the case permanently, marked with the catalogue ID
   it instantiates or as a product-local case beside it.

Traps:

- A case written against the fixed code only: it cannot
  show it would have caught the defect. Verify the case
  (T-06); its first step is this check.

## T-08: Run the suite in Unity

Steps:

1. The suite ships disabled — its folder carries the `~`
   suffix — or is not distributed; before the run it stands
   under an imported name, placed there by the project's
   own operation. A folder still carrying the `~` is not
   imported — no assembly, and the Test Runner shows
   nothing.
2. Open the project and clear compile errors in the test
   assembly before opening the runner.
3. In the Test Runner window, run the base verification
   cases first, then the plugin's whole assembly on the
   EditMode tab; the PlayMode layer runs on the PlayMode
   tab.
4. Read the failures in the runner and triage them
   (Diagnosing a Failing Case). For CI, the same run goes
   through batch mode with `-runTests` and an XML results
   file.

## T-09: Triage a failing case

The failure-triage reference holds the procedure: base
first, then the pass/pipeline bisection, then product
against dependency.

## T-10: Close out a test change

Steps:

1. Run the whole suite for the plugin, not only the new
   cases; order dependence shows only in the full run.
2. Check the working tree for residue: no temp assets, no
   fixture edits, remain after the run.
3. Report which catalogue cases the change covers, which
   required cases remain unwritten, and anything the tests
   cannot see that the change relies on.

