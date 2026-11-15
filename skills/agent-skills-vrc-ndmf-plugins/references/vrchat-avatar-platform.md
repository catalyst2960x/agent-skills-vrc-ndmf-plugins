# VRChat Avatar Platform Reference

The VRChat-side facts a plugin's generated output must satisfy: playable
layers, built-in animator parameters, state behaviour semantics, allowed
components, and the parameters that VRChat components themselves write.
Where this file states a requirement or a definition, the requiring party
is the VRChat avatar SDK, SDK3A: these are SDK3A requirements and
definitions, not conventions of NDMF, Modular Avatar, or this skill.
Facts are those of the VRChat documentation at creators.vrchat.com
(URLs per section). These pages describe the live platform, not an SDK version:
re-check the page when behaviour seems to have moved, and verify C#
identifiers against the SDK as always (SKILL.md NEVER #8). This file
is the standing extract for the mechanisms it covers; for a mechanism
it does not cover, check the official documentation before adopting
it.

## Table of Contents

- [Built-in Animator Parameters](#built-in-animator-parameters)
- [Parameter Type Conversion](#parameter-type-conversion)
- [State Behaviour Semantics](#state-behaviour-semantics)
- [Allowed Avatar Components](#allowed-avatar-components)
- [Stations](#stations)
- [Spatial Audio](#spatial-audio)
- [Component-Generated Parameters](#component-generated-parameters)
- [Contacts](#contacts)
- [Synchronization](#synchronization)
- [Constraints](#constraints)
- [Head Chop](#head-chop)
- [See Also](#see-also)

## Built-in Animator Parameters

Source: https://creators.vrchat.com/avatars/animator-parameters/

These parameters are SDK3A definitions: VRChat maintains them on every
SDK3 avatar. They are read-only
(they cannot be set from the Expressions Menu, by OSC, or by a
parameter driver), they are used by adding them to a controller, they
do not need an Expression Parameters entry, and they do not consume the
custom parameter budget (Expression Parameters and Expressions Menu). Generated conditions
may read any of them; their types come from this catalogue, not from
ParameterInfo (Animator Generation, Built-in parameters in generated
output).

| Name | Type | Notes |
| --- | --- | --- |
| IsLocal | Bool | True on the wearer's own client only |
| PreviewMode | Int | 1 while the avatar is being previewed |
| Viseme | Int | Oculus viseme index 0–14 (0 = sil); with jaw systems, 0–100 volume |
| Voice | Float | Microphone volume 0.0–1.0 |
| GestureLeft / GestureRight | Int | 0 Neutral, 1 Fist, 2 HandOpen, 3 FingerPoint, 4 Victory, 5 RockNRoll, 6 HandGun, 7 ThumbsUp |
| GestureLeftWeight / GestureRightWeight | Float | Analog trigger 0.0–1.0 |
| AngularY | Float | Angular velocity on Y |
| VelocityX / VelocityY / VelocityZ | Float | Move speed per axis, m/s |
| VelocityMagnitude | Float | Total move speed |
| Upright | Float | 0 prone, 1 standing |
| Grounded | Bool | Touching ground |
| Seated | Bool | In a station |
| AFK | Bool | HMD proximity off, or End key |
| TrackingType | Int | 0 uninitialized, 1 generic rig, 2 transitional, 3 head+hands (or desktop humanoid), 4 +hip, 5 +feet, 6 full body. Values 0–2 in VR are initialization states: do not branch into a dead end on them; the value can change at any time |
| VRMode | Int | 1 in VR, 0 otherwise |
| MuteSelf | Bool | The wearer muted themselves |
| InStation | Bool | In a station (distinct from Seated) |
| Earmuffs | Bool | Earmuff mode on |
| IsOnFriendsList | Bool | Viewer is friends with the wearer; false locally |
| AvatarVersion | Int | 3 for SDK3 avatars |
| IsAnimatorEnabled | Bool | False one frame before the animator is disabled; the disable is a viewing client culling a remote avatar, so the value is per client and not synced |

Eye/face tracking group (Bool, computed per client from received
tracking data, so remote values can differ from the wearer's):
`UserEyeTracking`, `UserFaceTracking`.

Avatar scaling group: `ScaleModified` (Bool), `ScaleFactor` (Float),
`ScaleFactorInverse` (Float), `EyeHeightAsMeters` (Float),
`EyeHeightAsPercent` (Float, 0.2–5.0 m mapped to 0–1).

Custom parameters, for contrast: at most 8192 total per avatar (synced
and unsynced); cross-platform (PC/Quest) sync matches parameters by
their position in the Expression Parameters list and their type, not
by name — use the same asset for both platform uploads.

Parameter name form: the platform's own names — every built-in above
and the aliased defaults VRCEmote and VRCFaceBlendH/V — are single
concatenated words, with no separator characters and each word
capitalized. A separator appears only where a component joins the
name of a value it writes to a prefix ({prefix}_{value};
Component-Generated Parameters). A name a plugin invents — a proxy
parameter's prefix, a default supplied for an unset name — follows
the same form; for a name that becomes an Expression Parameter, a
`/` also reads as hierarchy in an OSC address. A name the author
supplies is an input and is never rewritten to fit the form
(Parameters).

## Parameter Type Conversion

A parameter carries two types: the source type — the built-in's type
(above) or the Expression Parameters declaration's type — and the
animator type, the type the parameter is declared with in an
AnimatorController's parameter list. When the two differ, VRChat
converts the value to the animator type at runtime:

| Source type | Animator type | Conversion |
| --- | --- | --- |
| Int | Float | The value as a float (1 → 1.0) |
| Int | Bool | 0 is false; anything else is true |
| Float | Int | Rounded to the closest int (0.5 → 0, 0.6 → 1, 1.5 → 2) |
| Float | Bool | 0.0 is false; anything else is true |
| Bool | Int | true is 1; false is 0 |
| Bool | Float | true is 1.0; false is 0.0 |

The conversion applies per controller: two controllers that declare
the same parameter with different animator types read different
values from the same source. A mismatch is defined behaviour, not
an error; it is also how a Bool built-in can drive a Blend Tree, by
being declared Float in that controller.

## State Behaviour Semantics

Source: https://creators.vrchat.com/avatars/state-behaviors

The SDK state behaviours, and their platform semantics. How a
generator attaches them to states is Animator Generation (Attaching
SDK state behaviours).

| Behaviour | What it does |
| --- | --- |
| Avatar Parameter Driver | Sets, adds, copies, remaps, or randomizes avatar parameters on state entry |
| Animator Play Audio | Configures and plays an AudioSource |
| Animator Tracking Control | Switches individual body/element tracking between tracking and animation |
| Animator Layer Control | Blends the weight of one Animator Layer inside a playable layer |
| Animator Locomotion Control | Enables or disables locomotion |
| Animator Temporary Pose Space | Moves the avatar into a temporary pose space for the state |
| Playable Layer Control | Blends the weight of an entire playable layer |

- All state behaviours run on the first frame of the transition into
  their state. Execution is not guaranteed for very short states or
  transitions; a total of at least 0.02 s in and around the state is
  the documented safe floor.
- What a behaviour sets stays set: the layer weight from
  Animator Layer Control or Playable Layer Control,
  locomotion enabled or disabled, the tracking
  assignment, and the pose space remain after the state
  is exited, until another state runs the same behaviour
  and changes them. Nothing restores the previous value
  on exit by itself.
- Every state behaviour carries a Debug String field;
  when the behaviour runs, the string is printed to the
  output log.
- Avatar Parameter Driver: operations run once, on state entry, in
  list order. Synced values are clamped (Int 0–255, Float −1 to 1);
  controller-only parameters are not clamped. Built-in parameters
  cannot be driven, and do not work as a Copy source. Add and Random
  can diverge between clients: use a synced destination and run the
  driver locally (Local Only).
- Animator Layer Control: layer 0 cannot be changed (always weight
  1.0). Exiting a state mid-blend sets the layer to the goal weight
  at once.
- Animator Tracking Control: each body part is set to
  Tracking (the part follows IK, or simulation for the
  eyes and eyelids, or visemes for the mouth and jaw),
  Animation (the part follows the animator), or No
  Change. Setting Eyes & Eyelid to Animation disables
  eye movement and eyelid blinking; setting Mouth and
  Jaw to Animation disables visemes, and the viseme
  parameters stop being updated. When every IK tracking
  point is set to Animation, the animation plays as it
  is instead of through Networked IK; the points are
  Head, Left Hand, and Right Hand on desktop and 3-point
  tracking, and those plus Hip, Left Foot, and Right
  Foot under 6-point and full-body tracking.
- Animator Temporary Pose Space: runs only if its delay elapses
  before the state is exited. For posture-driven viewpoint changes,
  not for scaling the avatar.
- Animator Play Audio: a Source Path that names no AudioSource at
  the first transition disables the behaviour; blank means the
  avatar root. Random and Unique Random choose per player; the
  Parameter order, driven by a synced parameter, makes every player
  hear the same clip. The Parameter order's parameter must be
  defined in the avatar's Expression Parameters; if it cannot be
  found, the first clip is chosen instead. The behaviour plays
  through the AudioSource and does not enable it: an AudioSource
  whose component is disabled plays nothing, so the source has
  to be enabled for playback. Adding the behaviour to
  a sub-state machine itself (rather than to states) is not
  recommended.

## Allowed Avatar Components

Source: https://creators.vrchat.com/avatars/whitelisted-avatar-components/whitelisted-avatar-components

- Only listed components work in VRChat. Any other component, and any
  custom script, does not work and may block the upload. A plugin that
  emits or requires components must stay inside this list (plus the
  Modular Avatar and NDMF editor-side components its own build removes
  or consumes).
- VRChat components allowed on avatars: VRCAvatarDescriptor,
  VRCConstraint, VRCContactReceiver, VRCContactSender, VRCHeadChop,
  VRCIKFollower (deprecated — use VRC Constraints), VRCPhysBone,
  VRCPhysBoneCollider, VRCPipelineManager, VRCRaycast,
  VRCSpatialAudioSource (see Spatial Audio), VRCStation (see
  Stations).
- Unity components allowed: AimConstraint, Animation, Animator,
  AudioSource, Camera (see below), Cloth, Collider, FlareLayer,
  Joints, Light, LineRenderer, LookAtConstraint, MeshFilter,
  MeshRenderer, ParentConstraint, ParticleSystem,
  ParticleSystemRenderer, PositionConstraint, Rigidbody,
  RotationConstraint, ScaleConstraint, SkinnedMeshRenderer,
  TrailRenderer, Transform.
- Cameras: on the wearer's own (local) avatar, Camera components are
  fully whitelisted. On a remote avatar, cameras are disabled at
  load in every case — disabled, not removed, so an animation can
  enable them. They are removed outright, and cannot be enabled at
  all, unless the viewer is the wearer's friend or has used
  Show Avatar on that avatar; becoming friends afterwards does not
  reload the avatar, so it does not restore cameras removed at load.
- VRCRaycast and FinalIK components share a combined limit of 80 per
  avatar. DynamicBone is deprecated; use PhysBones.
- Android has additional restrictions beyond this list.

## Stations

Source: https://creators.vrchat.com/worlds/components/vrc_station/

The station component (type `VRCStation`) is documented with the
world components, but it is on the avatar allowed list and works on
avatars: other players sit on the wearer.

- An avatar may carry up to 6 stations; any beyond that are disabled.
- A station must be enabled at upload. A station disabled at upload
  never works, whatever later animations do.
- The fields: Player Mobility (Mobile, Immobilize, or Immobilize For
  Vehicle — the vehicle variant is the one meant for stations that
  move, as an avatar station does), Can Use Station From Station, an
  optional Animation Controller overriding the Sitting playable
  layer, Disable Station Exit, Seated (which drives the seated
  avatar's Seated parameter), and the Station Enter and Exit Player
  Location transforms. For an avatar seat, Disable Station Exit is
  normally left off: with it on, the seated player cannot leave
  voluntarily. The 2-metre rule is an upload-time check: the enter
  and exit locations must be no more than 2 metres apart at upload,
  and animation may move them further apart afterwards without
  breaking the station.
- Interaction uses a collider on the station's own GameObject; with
  none, VRChat adds a trigger BoxCollider of 1 m × 1 m × 1 m, and a
  collider on a child object does not count — the interaction area
  is set by a trigger collider placed on the station object itself
  (Sphere, Capsule, or Box; a Mesh Collider also works).
  No mesh is required for the station itself; the interact
  highlight follows a Mesh Renderer on the collider's object or its
  children (a Skinned Mesh Renderer does not count).
- Toggling a station at runtime is animator work (FX layer):
  disabling the station component ejects the seated player, while
  disabling the station's GameObject or its collider only stops new
  players from sitting. The working pattern is to toggle the
  collider — it starts disabled, and the animation enables it —
  rather than the station object's active state. Viewers can block
  the wearer's animations, so a station that is on by default stays
  usable for those viewers even if the wearer animates it off.
- While a player is seated, their avatar's origin (their feet) sits
  at the Station Enter Player Location, and moving that transform
  moves the player with it; the player's facing follows the station
  object itself.
- Seated and the Animation Controller interact: with Seated checked,
  the seated player's own Sitting layer goes to weight 1 and plays
  its sitting motion. To override the motion with the station's
  Animation Controller instead, Seated is left unchecked (the
  Sitting layer stays at weight 0); the override controller
  effectively runs between the Action and FX layers and can drive
  Humanoid animation.

## Spatial Audio

Source: https://creators.vrchat.com/worlds/components/vrc_spatialaudiosource

The spatial audio component (type `VRCSpatialAudioSource`) adds 3D
spatialization to an AudioSource (adding the component adds the
AudioSource). It works on avatars as in worlds, with the limits
below, enforced at runtime.

- An avatar AudioSource without one gets one created by the SDK
  with default settings, which may not match the design — a
  generator that emits or relies on an AudioSource adds the
  component itself and sets it.
- Avatar limits: Gain at most 10 dB (the property range is 0–24 dB);
  Far at most 40 m (also the default). The player's own Player
  Audio settings can override both.
- The spatialization properties — Gain, Near, Far, Volumetric
  Radius, and the falloff choice — are read at initialization:
  animating them at runtime is not supported. AudioSource
  properties unrelated to spatialization (pitch, for example) can
  still be animated.
- A custom falloff curve is allowed on avatars (Use AudioSource
  Volume Curve); in a world whose Far is shorter, VRChat squashes
  the curve by reducing the AudioSource's Max Distance. The avatar
  audio channel also runs through a compressor that prevents
  maliciously loud sounds.
- Toggling avatar audio: enable and disable the AudioSource
  components rather than their GameObjects.

## Component-Generated Parameters

Parameters written by VRChat components themselves. They exist without
any Expression Parameters entry, and ParameterInfo aggregates them
(Parameters) — a validating plugin meets these names and
must not treat them as missing declarations or as collisions with its
own parameters.

### PhysBones

Source: https://creators.vrchat.com/common-components/physbones

A PhysBone component (type `VRCPhysBone`) whose `Parameter` field
is set to a prefix writes:

| Parameter | Type | Meaning |
| --- | --- | --- |
| {prefix}_IsGrabbed | Bool | The chain is being grabbed |
| {prefix}_IsPosed | Bool | The chain was posed after grabbing |
| {prefix}_Angle | Float | 0.0–1.0, end bone's angle from rest (180° = 1.0) |
| {prefix}_Stretch | Float | 0.0–1.0 of maximum stretch |
| {prefix}_Squish | Float | 0.0–1.0 of maximum squish |

Two generation-relevant rules: a bone's transforms only respect
animation when the component's Is Animated option is enabled, and a
constraint component and a PhysBone must not affect the same
GameObject (execution-order conflicts) — put the constraint on the
parent.

### Contact Receivers (on avatars)

Source: https://creators.vrchat.com/common-components/contacts/

- A contact receiver (type `VRCContactReceiver`) writes its named
  parameter on contact. Receiver types:
  Constant (value while any contact is present, resets when none),
  OnEnter (one frame on contact), Proximity (Float 0.0–1.0, closest
  contact wins; Float parameters only). Constant and OnEnter suit
  Bool parameters; with a Float or Int parameter, detection writes
  1. OnEnter can take a Min Velocity.
- The parameter does not need to be in the synced Expression
  Parameters list.
- A Receiver can also serve shaders through Write to Shaders,
  instead of or besides writing its parameter (Contacts).
- Advanced options generate derived parameters from the same base
  name: {parameter}_SenderType (Int: 0 none, 1 same avatar, 2 other
  avatar, 3 world, 4 item), {parameter}_SenderLocalPositionX/Y/Z,
  {parameter}_SenderLossyScaleX/Y/Z, {parameter}_SenderVelocityX/Y/Z,
  {parameter}_SenderVelocityMagnitude, {parameter}_PointSeparation,
  {parameter}_ProximitySpeed (Proximity type only), and one Bool per
  collision tag, {parameter}_CollisionTag_{tag}.
- Collision tags are case-sensitive, at most 16 per contact. Contact
  shapes are limited to 3 m radius (6 m per side) after scaling.
- Local-only contacts do not affect the performance rank; an avatar
  may carry up to 256 contact components when they are local-only.

### Raycast

Source: https://creators.vrchat.com/avatars/avatar-components/raycast

The SDK's raycast component (type `VRCRaycast`) fires a ray and
moves a result transform to the hit point. With its `Parameter`
field set to a prefix, it writes three animator parameters:

- `{prefix}_Hit` (Bool) — the ray is hitting something
- `{prefix}_Ratio` (Float, 0.0–1.0) — the proportion of the
  configured distance travelled before the hit
- `{prefix}_Distance` (Float) — the world distance travelled
  before the hit

At most 80 per avatar, shared with FinalIK (Allowed Avatar
Components). The maximum ray distance is 1000 units.

The name needs its owner: "raycast" also names Unity's
Physics.Raycast, which is a scripting query and not a component,
and no other raycast component is on the allowed list. On an
avatar, a raycast a plugin can generate or reference is always
this component.

## Contacts

Source: https://creators.vrchat.com/common-components/contacts/
(the buffer layout rests on the SDK header `VRCContacts.cginc`)

Contact Receivers also write animator parameters
(Component-Generated Parameters, Contact Receivers). The other
output is Write to Shaders (Material Contacts):

- With Write to Shaders enabled — the Receiver Type is
  Proximity — the Receiver updates the MaterialPropertyBlock of
  its target renderers every frame.
- One Receiver updates at most 4 renderers; for more, split the
  same configuration across several Receivers.
- The buffer name is the Receiver's Parameter, or the Shader
  Parameter Override when one is set. Setting the Override and
  leaving Parameter empty keeps the Receiver off the parameter
  path and dedicated to shaders.
- Two buffers arrive, each with 9 elements. The transforms
  buffer holds the world transforms of the shapes of the 8
  nearest Senders, with the Receiver's own root transform (the
  root, not the shape) in the last element. The data buffer
  packs each target's owner and matched tags: x is the owner's
  player ID, y is the tag bits, and the last element's y is the
  target count.
- The tag bits arrive only when contact tag writing is also
  enabled on the Receiver; transforms alone do not need it.
- Distinguishing the wearer from others on the shader side
  takes separate Receivers — Allow Self only and Allow Others
  only — with separate buffer names.

How a shader declares and reads the buffers is VRChat
Material Contact (Path 2).

## Synchronization

- What VRChat sends for an avatar is a closed set: the built-in
  parameters, the synced Expression Parameters, tracking IK, the
  player's position and scale, and PhysBone grab/pose state.
  Everything else about an avatar's behaviour is recomputed
  independently on each client from that set; a behaviour that looks
  synchronized is synchronized only through these.
- A Parameter Driver has no network behaviour of its own: it runs on
  every client when the same transition fires there (State Behaviour
  Semantics). Local Only restricts the driver's writes to the
  wearer's client — the state still transitions everywhere. A random
  or one-shot result agrees across clients only when the wearer's
  client writes it to a synced parameter.
- PhysBone and Contact parameters are written locally, per frame,
  from each client's own simulation: treat their values as
  per-client approximations, and do not declare them as synced
  Expression Parameters. Synced values are sampled at a low rate, so
  a one-frame value (an OnEnter pulse) is dropped, and a continuously
  rewritten value chatters. When an event must reach every client,
  translate it on the wearer's client into a synced parameter (a
  Parameter Driver writing a synced destination) and let the other
  clients react to that.
- Synced Expression Parameters travel as a packed stream in list
  order, not by name: the sender serializes its synced values in the
  order of its Expression Parameters, and each receiver unpacks them
  against the order of the avatar it actually sees — a platform
  variant or a fallback may be a different avatar. Cross-variant
  consistency therefore depends on the synced parameters' order and
  total bit count, not their names. Modular Avatar merges
  declarations in hierarchy order, so placement order is part of
  sync compatibility.
- PhysBone grab and pose state is matched between clients by Network
  IDs assigned in hierarchy order; avatar variants whose PhysBones
  appear in a different order misalign that state.

## Constraints

Source: https://creators.vrchat.com/common-components/constraints/

- Prefer VRChat constraints on avatars: any Unity constraint on an
  avatar is automatically converted to a VRChat constraint when the
  avatar loads in-game, so Unity constraints make the editor preview
  and the performance rank inaccurate.
- The six types mirror Unity's: VRCAimConstraint, VRCLookAtConstraint,
  VRCParentConstraint, VRCPositionConstraint, VRCRotationConstraint,
  VRCScaleConstraint.
- A constraint's Target Transform and its source references cannot be
  changed by animation; to switch targets, toggle between several
  constraint components, or animate source weights instead.
- Freeze To World locks the target in world space on the frozen axes
  and is designed to be toggled by animation clips; it is not the same
  as disabling the component.
- Multiple sources blend by normalized weight: each source's share is
  its weight divided by the total of all source weights. At an overall
  weight of 0 the constraint writes its At Rest values, so the object
  follows its real parent — weight 0 is the rest pose, not an absence
  of control. The At Rest and Offset values define that reference
  pose and are normally computed by the editor's Activate from the
  current pose; anything writing a constraint supplies values
  consistent with the pose it should hold.
- VRC constraints add settings Unity's do not have: the Target
  Transform above, Solve In Local Space (evaluate in local rather
  than world space), Freeze To World, and Rebake Offsets When
  Unfrozen. Partial Freeze Rotation Axes settings do not behave as
  per-axis locks — rotation is quaternion-based — so do not rely on
  them; the position and scale freeze axes do not have this problem.
- Constraints have no synchronization of their own: their output is
  computed locally on each client, so a constraint-driven state
  (including a world-fixed position) agrees across clients only when
  it is driven by synchronized state (Synchronization).
- Performance rank counts every constraint on the avatar, including
  disabled ones, plus a depth score equal to the longest dependency
  chain; depth is only calculated accurately when the avatar uses
  VRChat constraints exclusively.

## Head Chop

Source: https://creators.vrchat.com/avatars/avatar-components/vrc-headchop

- The head chop component (type `VRCHeadChop`) controls which bones
  VRChat scales down to keep the head out of the wearer's first-person
  view. It affects only the wearer's own view — not mirrors, not other
  players.
- Limits: at most 16 per avatar (the SDK errors
  above that), each listing at most 32 target bones with a per-bone
  scale factor and an apply condition (always, VR only, non-VR only),
  multiplied by one global scale factor.
- The component's enabled state and its global scale factor can be
  animated, which is how first-person visibility toggles are built.

## See Also

- Animator Generation — the layer/state pattern and the behaviour
  generation notes this file's semantics apply to.
- Parameters; Error Reporting; Expression Parameters and Expressions Menu — build-time parameter
  truth, declaration, and the sync budget.
