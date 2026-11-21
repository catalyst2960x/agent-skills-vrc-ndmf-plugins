# Test Infrastructure

The shared base every test class in a plugin's test assembly
derives from. The base exists so that a case states only its
own preconditions and assertions: creating the avatar, running
the build, capturing the reports, and finding the generated
output are the same in every case. The worked code is the
test-base template; this reference states the
rules the code follows.

## The test assembly

Tests are NUnit editor tests in an Editor-only assembly,
compiled under UNITY_INCLUDE_TESTS, referencing NDMF, the
plugin's runtime and editor assemblies, and Modular Avatar
when the plugin is dependency case 2. Tests that build
VRChat fixtures also reference the SDK assemblies —
VRC.SDK3A, VRC.SDK3A.Editor, VRC.SDKBase, and
VRC.SDKBase.Editor — the form Modular Avatar's own test
assembly uses. The SDK types themselves ship as precompiled
DLLs in the SDK package, so the assembly definition keeps
overrideReferences off; turning it on without enumerating
the SDK DLLs leaves the SDK types unresolved in the tests
while the product code still compiles. In a distributed
package, the test folder's name carries a trailing `~` so
Unity does not import it into a user's project; the adopting
project links or copies the folder in for its own development.

## Tracked cleanup

Every Unity object a test creates — GameObjects, controllers,
materials, the built avatar itself — is registered with the
base as it is created. Teardown destroys every registered
object and deletes the temporary asset folder the direct
contexts write into. A test never cleans up after itself by
hand, and never relies on the next test tolerating its
residue: isolation is what makes a failure attributable to the
case that produced it.

## The minimal avatar

The base builds a minimal avatar in code: an Animator and an
avatar descriptor on the root, the descriptor's playable
layers populated, the FX layer carrying an empty controller
for the plugin to merge into, and an empty Expression
Parameters asset. Platform variants (a VRChat descriptor, or
NDMF's avatar root for a generic platform) are produced by the
same builder with the platform named. A case that needs a
realistic avatar shape loads a prefab fixture instead; common
fixtures live in one folder next to the tests.

An asset a fixture's state behaviour references — an audio
clip, for example — is created as a persisted asset in the
suite's temp asset folder, not as an in-memory object: the
build copies an in-memory object, so the built behaviour
references a different instance, while a persisted asset
reaches the built avatar as the same instance.

## The two build entries

A direct BuildContext over a clone of the avatar runs one
pass in isolation: construct the context, activate the
extension contexts the pass reads, and invoke the pass. This
entry serves cases whose claim is a single rule of a single
pass.

The full pipeline runs through NDMF's manual build entry
(AvatarProcessor.ManualProcessAvatar), which clones the
avatar, runs every phase with every registered plugin, and
returns the built avatar. This entry serves the integration
cases: ordering against Modular Avatar and Avatar Optimizer,
and restructuring by other systems, are visible only here.
The built avatar is registered for teardown like any other
created object.

## Framework defaults in fixtures

A case is designed around the frameworks' default
behaviour, never around disabling it: turning a default off
to make a case pass changes the environment the product
runs in, and the case stops measuring the product. Where a
claim does not hold against a default behaviour, the claim
narrows to the range where it does, or the fixture is
designed so the behaviour sits in the same position in
every build the case compares.

The comparison form this decides is the before/after
difference: other systems insert layers by placement rules
of their own — Modular Avatar's MMD support, for one,
decides its insertion from the first three FX layers — so a
difference between two builds attributes an insertion to
the plugin whenever the rule fires in only one of them. An
author FX with three real layers places the generated
layers outside the first three in both builds, the
insertion fires alike in both, and the difference is the
plugin's contribution. Fixtures also premise nothing on
what the pipeline removes: an empty layer, and a generated
controller's default Base Layer, do not remain in the built
FX.

Survival is part of fixture design for the same reason:
Modular Avatar's garbage-collection pass removes
GameObjects the build leaves carrying only marker
components, and plain inactive objects. An object a case
expects to find on the built avatar carries a real
component, so the pass keeps it; and an assertion that a
generated holder is gone is made as component absence
across the whole built avatar, not by looking the holder
up — an object the pass collected and an object the plugin
never made are the same lookup result, and only the
absence form separates a defect from the collection.

## Capturing reports

Reports made during an action are captured with
ErrorReport.CaptureErrors, the framework's test-intended
capture API: it runs the action inside a report scope and
returns the ErrorContext list. Assertions read
ErrorContext.TheError.Severity. The severity is the contract
a case fixes: Information for a required input that is not
set, NonFatal for a set-but-invalid value and for contention,
and no Error from the plugin in any case. Message text is a
localization surface and is not asserted.

The captured context exposes the error itself, the
reporting plugin, the pass name, and the extension context
— and nothing else. Context objects attached when the
problem was reported appear in the build report window but
are not retrievable from the captured context, so a case
identifies a report by its plugin and pass name, never by
a context object. The message (IError.ToMessage()) serves
diagnostics — assertion failure text — not assertions.

A case that asserts on output reads the reports first in
one specific way: no captured report is an Error. That
pre-check separates a pass that threw from a generation
that reported and skipped by design, and the existence
assertions after a build carry the report summary as their
message — a skipped generation's reason is in the reports,
and an existence failure without them sends the triage back
to the beginning.

## FX inspection

Assertions read the built avatar, not the scene. The FX
controller is reached through the built descriptor's playable
layers; within it, layers, states, and clips are found by
name. Presence and absence have dedicated helpers in the
base, for layers and for states, so a case never consumes
the nullable layer finder itself, and a missing state fails
with the states present rather than as a null reference
downstream. Where a mismatch's diagnostic is the collection
itself — the reports behind a count, the layers behind a
naming claim, the transitions behind a condition — the
assertion carries that collection's summary as its message. An assertion compares values, not presence: a layer's
defaultWeight, a curve's binding (path, type, property) and
its key values, a transition's condition modes and
thresholds, a controller parameter's name and type. A
generated layer serialized at weight 0, or a clip whose curve
carries the wrong value, is a defect these comparisons exist
to catch. An expected binding path is resolved from the
built hierarchy, from the object's real names: restructuring
renames what it moves, so a path written from the scene side
asserts against a name the built avatar does not carry.

A material a clip writes is identified by its values, not
by reference. The build copies a material that is not a
persisted asset along with the clips that reference it, so
the material on the built avatar is a distinct instance
carrying the same values, and a reference-equality
assertion fails on a correct build. The fixture
differentiates materials by a value property — a colour —
and the assertion reads that property. A value a clip
drives on a material is read from the renderer's property
block: material curves are applied through the block, not
written into the material object.

Identity across the build is asserted by instance ID, not by
reference. An object the build does not copy — a surviving
state behaviour, for example — is the same native object on
the built side but can arrive as a different managed wrapper,
so NUnit's AreSame fails on a correct build. The assertion
compares GetInstanceID values, with which Unity's own
equality agrees:
Assert.AreEqual(expected.GetInstanceID(),
actual.GetInstanceID()).

## Running the tests

Requirements: Unity 2022 LTS with the Test Framework (the
test assemblies reference UnityEngine.TestRunner and
UnityEditor.TestRunner, and nunit as a precompiled
reference); the plugin's dependency set installed in the
project — NDMF, and for dependency case 2 also Modular
Avatar and the VRChat SDK that the fixtures build against
(Avatar Optimizer only for the case whose fixture is an
optimized build); and the test folder imported, under a
name Unity imports.

A folder whose name ends in `~` is not imported at all: its
scripts do not compile, its assembly does not exist, and
the Test Runner shows nothing for it. The suite is
distributed in that disabled form — the folder ships as
`UnitTests~` — or is not distributed at all. Placing it
under an imported name is each project's own operation;
the run below presupposes it has been done.

In the editor, the tests run from the Test Runner window
(Window, General, Test Runner): the EditMode tab for the
editor tests, the PlayMode tab for the PlayMode layer. The
runner lists tests by assembly, class, and method; a run
can cover the whole assembly or a selection. The base
verification cases run first (T-01), then the full suite
(T-10). An assembly that does not compile appears in the
runner not at all, so compile errors are cleared before the
runner's state means anything.

The same run is available from the command line in batch
mode (`-runTests` with `-testPlatform EditMode` or
`PlayMode` and `-testResults` naming an XML file), which is
the form CI uses; the editor window and the command line
run the same assemblies and report the same results.

## What the tests cannot see

The build output is static. Whether a contact drives a toggle
in the client, whether a clip plays, and which state a running
Animator is in are runtime facts with no representation in the
built assets, and no case asserts them. A case's expected
result is always decidable from the built avatar and the
captured reports; anything beyond that is manual verification,
recorded as out of scope rather than approximated by a proxy
assertion.
