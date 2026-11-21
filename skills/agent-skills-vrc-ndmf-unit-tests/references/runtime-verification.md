# Runtime Verification

A companion to the test suite, not part of it. The suite's
scope is static (SKILL.md, Static scope): a requirement that
cannot be restated statically is split, its static part
becomes a case, and its runtime part stays manual. This
reference gives the runtime part a method, using
Av3Emulator, the Avatar 3.0 emulator. Runtime
verification is not a case form, has no procedure in the
Procedure Index, and does not enter the suite's completion
condition.

What follows is the method only: what is driven and where
values are read. The means of reaching into the running
editor — the tooling and transport an environment uses — is
that environment's own and is not stated here.

## Av3Emulator

Av3Emulator is an Avatar 3.0 emulator that runs in the Unity
Editor. Attached to an avatar in Play Mode, it runs the
avatar's playable layers as the client would, driven by
emulated inputs in place of a VRChat client. What the method
uses of it:

- It supplies the inputs a client would: the built-in
  parameters (gestures among them) through its own input
  fields, and expression parameters through its parameter
  store, so a fixture is driven the way a wearer's client
  drives the worn avatar.
- It runs state transitions on those parameters, so a
  generated layer's current state and its round trips are
  observable while it runs.
- It exposes the running animators per layer, so the values
  written into a controller and the layer weights are read
  back from the playables as the run proceeds.
- It creates mirror and shadow runtimes alongside the main
  one, under a shared name prefix; the method identifies its
  target among them by exact name.

Its limits are stated in What Av3Emulator does not
reproduce, below.

## Verifying runtime behaviour with Av3Emulator

The harness: build the fixture through the manual build
entry (AvatarProcessor.ManualProcessAvatar), attach
Av3Emulator to the built clone, and enter Play Mode. Av3Emulator
creates its runtimes under a shared name prefix —
main, mirror, and shadow; the target runtime is identified
by its exact name, not by a prefix match.

Driving: inputs are written through the runtime's parameter
store — the value field of a Bool entry, the expression
value field of a Float entry. Writing to a playable directly
does not hold: Av3Emulator's own supply overwrites it. An
input the runtime supplies itself every frame — the built-in
gestures are the standing example — is driven through the
runtime's own input fields (the gesture, index, and weight
fields), for the same reason; and a runtime-supplied input
follows the platform's own rules as it arrives, as the
gesture weights arrive as an analog value only while the
gesture is Fist and a Contact's Proximity arrives graded by
distance. The expression system clamps inputs to [-1, 1], so
an expectation is computed from the inputs as read back, not
from the values the driver intended.

Reading: a value the plugin wrote appears neither in the
runtime's exported values nor on the root Animator. Written
values are read from the per-layer AnimatorControllerPlayable
that carries the parameter, with the playable's parameter
getters; the target playable is identified by the parameters
it holds. A controller with no layers has an empty parameter
table in its playable — a declaration alone is not readable.
Layer weights are read from the playable as well. A value a
driver writes to an expression parameter is read back in the
parameter store, and the two reads agree where both exist.
Scene-side results — an object's active state, a blend
shape's weight — are read from the objects of the runtime
clone.

The method serves behaviour the static assertions cannot
observe: drivers that fire on state entry, a gate held in a
layer weight, reactive consumption by another system — any
claim whose evidence is a value changing over time rather
than a structure in the built avatar.

## Verifying state

The current state of a layer is read from its playable, by
layer index. The index is the canonical identifier: a state
name's hash collides across layers that carry states of the
same name, so a name is an auxiliary check only. Driving the
condition parameters and reading the generated layer's
current state confirms a transition round trip directly —
and a defect in this area presents exactly as a mismatch
between the hierarchy's state and the layer's state, which
the two reads make visible. For gesture layers the same read
is made per layer of the Gesture playable, with the driving
done through the runtime's gesture fields; the mirror and
shadow runtimes have no Gesture playable, so the observation
covers the main runtime only.

The form varies with what the product exposes. Where a
layer's state is the claim, the state is read directly, as
above. Where the claim is a value — a logic gate's truth, a
smoothing's convergence — the state is not read at all: the
output parameters carry the verdict, read over time. And
where a product's states are internal, verification runs by
effects: the toggle's round trip is read from the objects'
active states, the blend shapes' weights, and the values the
drivers wrote, with the gate's weight read from the playable
directly. On load the avatar enters its initial states, so a
driver set to fire on state entry has already fired once
when observation starts; an initial read is a post-entry
read, not a baseline.

## Verifying components in combination

Combinations are run in three families: two products on one
avatar, one driving an object under which the other's
component sits, with the downstream component's chains
passing through the driven object; a product with a Modular
Avatar component that restructures between generation and
consumption (the Bone Proxy form, in Restructuring above);
and either fixture with Avatar Optimizer's TraceAndOptimize
added last at the avatar root. Multiple instances of one
product's component on one avatar are not run as a runtime
case: multi-placement is covered by the suite (layer-name
numbering, contention reports), and the runtime method here
covers one instance of each party.

The method: drive the upstream input, and read the same
fact on three faces — the intermediate driven objects'
hierarchy state, the downstream system's generated proxy
values on the playable, and the downstream generated layer's
current state. The round trip is closed when disabling the
input returns all three faces to their baseline values. The
three-face read is itself the defect detector: a defect in
this area presents as a mismatch between the faces —
hierarchy inactive, proxy at 1, layer active — and the fix
is confirmed when the three agree through the round trip.
For a restructuring combination the faces are the built
hierarchy path, the set of proxies present in the built
controller (which grows by the ancestors a move gains), and
the layer-state round trip. For an optimizer combination,
the same fixture is rebuilt with the optimizer added, and
the reads are repeated: the build reports, the built
controller's structure against the unoptimized build, and
the runtime round trip.

One further path runs from a generated component to the
framework's own consumption: a component the product
generates on a holder object is consumed by Modular
Avatar's reactive system, and the verification reads the
product's generated animation arriving through the
Animation Index and the driven property following the
condition at runtime.

The range of these combinations is stated with them. The
optimizer fixtures are mesh-less minimal avatars: the
optimizer's heavy processing — mesh and bone consolidation,
animator layer merging — is not exercised, and behaviour
under it is unconfirmed. The last link of the audio
combination, the playback itself, is outside the emulator's
range and rests on a real-client confirmation.

## What Av3Emulator does not reproduce

Response to animation is per property, not per component.
Measured in Av3Emulator: a Contact Receiver's radius, an
AudioSource's priority, and a Light's type did not respond
to animation; a PhysBone's pull, a Light's intensity and
enabled state, a Scrollbar's number of steps, a Dropdown's
value, and an AudioSource's volume did.


Av3Emulator supplies parameters and runs state transitions.
It does not reproduce motion-time scrubbing: a state whose
motion time is a parameter advances freely instead of
following the parameter. Scrub generation is verified
instead against a plain Animator in Play Mode, where a
timeParameterActive state evaluates as authored.
AudioSources do not sound. The
shader-side sender type is not populated. Contacts and
PhysBones are approximations, a per-frame step size is not
separable from the emulator's frame rate, and nothing in
Av3Emulator stands in for synchronization between clients.
The final
confirmation of runtime behaviour is the real client; an
Av3Emulator result settles only the range it reproduces.

## Restructuring between generation and consumption

The runtime form of the ORD-07 fixture (Test Case
Catalogue): a ModularAvatarBoneProxy on the target object
makes the build reparent it between generation and the pass
that consumes it. The fixture is a pair — joining, where the
target moves under a toggled gate, and leaving, where it
moves out from under one. The static form asserts the built
layer's condition sets (Templates, ordering-interoperation);
the runtime form drives the built avatar in Av3Emulator and
reads whether the drive reaches the object at its moved
position.

The boundary is stated with the method: remapping covers
reparents up to the pass that consumes the paths. Reparents
in the Optimizing phase are outside it, and a runtime result
under such a reparent is not evidence about the mechanism.

## Sources

| Product | What was taken from it | License |
| --- | --- | --- |
| lyuma/Av3Emulator (package lyuma.av3emulator) | The runtime the method drives and reads (Av3Emulator, above). The method's claims rest on measurements in plugin verification work, cross-checked across several controllers and fixtures | MIT |
