# VRChat Material Contact Reference

The four paths that connect VRC Contacts, VRC Material
Contact, the Animator, and shaders on a VRChat avatar. Of the
parties, the Animator is the part the plugin generates: its
controllers, parameters, and clips are the plugin's output.
The others are counterparts the output connects to — the
plugin places and configures VRC Contact components, and it
binds to shaders by name; it does not generate shaders.

Two names need stating once, because both are VRChat's own.
**VRC Contacts** is the contact component system: the
components `VRCContactSender` and `VRCContactReceiver`. **VRC
Material Contact** is a Receiver's shader-facing output: with
Write to Shaders enabled, the Receiver writes contact buffers
that shaders read. The platform facts about both are in VRChat
Avatar Platform (Component-Generated Parameters; Contacts).
The general technique behind Path 3 — driving material
properties from clips — is not VRChat-specific and is in
Manipulation Techniques (Shader parameters).

A shader holds no state: everything it does is a function of
its inputs — material properties, and the VRC Material Contact
buffers the platform writes per frame.

Of the four paths, only one is an operation the plugin
performs on a shader: Animator → Shader (Path 3). The paths
from Contacts are platform mechanisms, not operations: the
plugin's output consumes Path 1 as conditions, and sets up
Path 2 only from the Receiver side, by configuring the
Receiver and its buffer name — reading the buffers is shader
authoring, stated here because the name is the plugin's
contract. Path 4 is the ordinary object operation
(Manipulation Techniques, Objects) pointed at contact
objects; what is specific to it is the consequence for
detection, stated under Path 4.

## The three parties, and where state lives

- A shader is computation at rendering time and holds no state.
  Its inputs are material properties and the buffers the
  platform provides; its results are read by nobody.
- VRC Contacts distribute proximity detection over two paths:
  a Receiver writes Animator parameters (Path 1), and its VRC
  Material Contact output feeds shader buffers (Path 2).
- The Animator holds parameter values and layer state, and
  drives material properties (Path 3) and the placement of
  objects, VRC Contacts included (Path 4).

| State | Owner |
| --- | --- |
| Proximity (who is in range, and in what order) | The contact simulation |
| Parameter values and the current layer state | The Animator |
| Current blend shape weights | The skinned mesh |
| Shader computation results | Nobody (a pure function of the current inputs) |

## The four paths

| Path | From → To | Carried by |
| --- | --- | --- |
| 1 | VRC Contacts → Animator | A Receiver writing its named parameter |
| 2 | VRC Material Contact → Shader | Write to Shaders: per-frame MaterialPropertyBlock buffers |
| 3 | Animator → Shader | Clip curves bound to material properties |
| 4 | Animator → VRC Contacts | Activation and placement of the objects carrying the components |

## Path 1: VRC Contacts → Animator

A Receiver writes its detection result into its named
parameter; a controller reads the value by declaring the same
name and type, and no Expression Parameters entry is needed.
The Receiver types, the derived Advanced Parameters, and the
per-client character of the values are platform facts in
VRChat Avatar Platform (Contact Receivers; Synchronization).

Sample code is in the templates (material contacts,
animator parameters).

## Path 2: reading VRC Material Contact buffers

The shader includes `VRCContacts.cginc` and declares the buffer
under the Receiver's buffer name, with the header's declaration
macro (`VRC_CONTACT_DECLARE`). The name becomes part of
generated identifiers through the macros, so it must be usable
as an identifier. What the two buffers contain is in VRChat
Avatar Platform (Contacts).

The buffers are read-only for the shader. A buffer is a
snapshot updated every frame: a reader sees only the latest
values, with no change detection and no history. With zero
targets the buffer is empty and a contact-driven effect is
inert; the nearest Sender is index 0.

Sample code is in the templates (material contacts,
shader buffers).

## Path 3: Animator → Shader

Generated clips drive material properties through renderer
bindings. This is the one general operation among the four
paths, and the only one whose technique is not specific to
VRChat: the binding form, the shader-side requirements, and
per-renderer instanced values are in Manipulation Techniques
(Shader parameters).

Sample code is in the templates (animator material
property driving).

## Path 4: the Animator controls VRC Contacts

VRC Contact Senders and Receivers are components on objects,
so the Animator's ordinary object control (Manipulation
Techniques, Objects) changes the geometry of detection itself:

- Disabling the object carrying a contact removes its Senders
  and Receivers from detection. Toggling activation is the
  simplest form of opening and closing a contact.
- Moving the object's Transform moves the detection range with
  it: a shape is evaluated at the object's root Transform plus
  its offset. To follow a bone, place the contact's object as
  a child of the bone's object; the hierarchy decides, no code
  is needed.
- The shape parameters themselves (radius, offsets on the
  component) are not targets of this path; activation and
  placement are.
- Combined with Path 1 this forms a feedback loop: detection
  changes a parameter, and the parameter changes the geometry
  of detection. The loop's state is held by the Animator's
  parameters.

Sample code is in the templates (animating contact
placement).

## Three separate name bindings

Three name bindings connect the parties. They are separate
mechanisms: matching one says nothing about the others.

| Binding | Left | Right | When they do not match |
| --- | --- | --- | --- |
| Receiver parameter | A VRC Contact Receiver's Parameter | An Animator parameter | The parameter is not set |
| Shader buffer | A VRC Contact Receiver's Parameter or Override | The shader's declaration name | The buffer does not arrive, and the shader reads empty data |
| Clip property | An animation clip's property path | A material property name | That property is not driven |

Separately: a controller that reads parameters generated by
other components (PhysBone, VRC Contact Receiver) declares
them itself, with the same name and type (VRChat Avatar
Platform, Component-Generated Parameters).

## Directions with no path

- There is no path from a shader to the Animator, and none
  from a shader to VRC Contacts. The platform provides no
  write API, and an avatar has no means of reading GPU results
  back to the CPU.
- An apparent detour does not close either. Attaching a VRC
  Contact Sender to a shape deformed in the vertex shader and
  detecting that does not work: a contact's shape is determined
  by the object's Transform and offsets, and vertex deformation
  is reflected in neither. PhysBone colliders are likewise
  based on Transforms.
- Render textures and Cameras do not form a path either. A
  Camera component is usable on an avatar under conditions
  (VRChat Avatar Platform, Allowed Avatar Components), and a
  Custom Render Texture lets a shader read its own previous
  frame — a feedback that completes inside the shader — but a
  render texture's content can only be read by shader
  sampling, and no CPU-side reader converts pixels into
  Animator parameters.
- What is possible is parallel driving: driving both the
  Animator and the shader from the same input (a VRC Contacts
  detection, a parameter) coordinates their behaviour. That is
  not a path carrying a shader result to the Animator.
- Branching on a shader's output therefore either completes
  inside the shader, or is supplied separately as an input on
  the VRC Contacts or Animator side. (In a World, Udon with
  `VRCAsyncGPUReadback` opens the shader-to-Animator path;
  Worlds are outside this skill's scope.)

## Update timing

Each path is evaluated per frame. In chains across paths —
from VRC Contacts to the Animator, and on through properties
to the shader, for example — delays of one frame per path can
accumulate.
