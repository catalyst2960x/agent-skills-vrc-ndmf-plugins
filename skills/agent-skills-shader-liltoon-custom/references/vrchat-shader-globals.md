# VRChat Shader Globals and Contact Buffers

The VRChat client publishes data that any shader rendered in the
client can read: global variables carrying camera, mirror and time
state, and — through Contact Receivers — per-renderer contact
buffers. lilToon itself reads none of it. A custom shader declares
what it needs and reads the values in its hooks; this reference is
the declaration surface and the value layouts.

Two rules govern every declaration in this file:

- The `_VRChat` prefix is a protected namespace. Declare only the
  documented names, with the documented types. Do not invent
  `_VRChat*` variables; VRChat may add undocumented ones at any time.
- These are not material properties. Do not add them to the datas
  lilblock, and do not declare them through `LIL_CUSTOM_PROPERTIES`
  (that macro writes into the material's uniform block, which the
  client does not write to). Declare them as plain file-scope
  uniforms in `custom_insert.hlsl`, which is included into every
  pass; the client supplies the values by name.

## Camera and mirror globals

| Variable | Type | Contents |
| --- | --- | --- |
| `_VRChatCameraMode` | float | 0 rendering normally; 1 VR handheld camera; 2 desktop handheld camera; 3 screenshot |
| `_VRChatCameraMask` | uint | The active camera's `cullingMask`; available when `_VRChatCameraMode != 0` |
| `_VRChatMirrorMode` | float | 0 not in a mirror; 1 mirror viewed in VR; 2 mirror viewed in desktop mode |
| `_VRChatFaceMirrorMode` | float | 1 when rendering the face mirror (VR and desktop use different camera types), 0 otherwise |
| `_VRChatMirrorCameraPos` | float3 | World-space position of the mirror camera (eye-independent, centred in VR); (0,0,0) when not rendering in a mirror |
| `_VRChatScreenCameraPos` | float3 | World-space position of the main screen camera; (0,0,0) when inactive |
| `_VRChatPhotoCameraPos` | float3 | World-space position of the handheld photo camera; (0,0,0) when inactive |
| `_VRChatScreenCameraRot` | float4 | World-space rotation (quaternion) of the main screen camera; (0,0,0,0) when inactive |
| `_VRChatPhotoCameraRot` | float4 | World-space rotation (quaternion) of the handheld photo camera; (0,0,0,0) when inactive |

The values are per render: the same material is drawn for the main
view, again for a mirror, again for the handheld camera, with
different values each time. A hook body therefore branches on the
current render — for example, suppressing or altering an effect when
`_VRChatMirrorMode != 0`. This is a runtime branch inside the pass,
not a variant: do not gate it with a feature define.

## Time globals

All time globals carry unsigned integer bit patterns and are
declared as `uint`. They are available in the client and in the SDKs
in play mode.

| Variable | Contents |
| --- | --- |
| `_VRChatTimeUTCUnixSeconds` | Lower 32 bits of the current UTC time, seconds since the Unix epoch (unsigned; no 2038 overflow) |
| `_VRChatTimeNetworkMs` | Synchronised network time in milliseconds (the value of Udon's `Networking.GetServerTimeInMilliseconds`); for synchronisation and offsets only — the absolute value is meaningless and the value can wrap |

`_VRChatTimeEncoded1` packs the time of day into bit fields (ranges
inclusive, zero-indexed): bits 0–4 UTC hour, 5–10 UTC minute, 11–16
second (shared by UTC and local), 17–21 local hour, 22–27 local
minute, 28–31 reserved.

`_VRChatTimeEncoded2` packs: bits 0–9 millisecond (shared), bit 10
sign of the timezone offset (1 if negative), bits 11–26 timezone
offset from UTC to local time in seconds, bits 27–31 reserved.

Local time and the offset follow the "preferred timezone" setting in
the VRChat menu, and "current time of day" is always the *observer's*:
an avatar shader showing a clock shows each viewer their own local
time, not the wearer's.

The SDK ships decode helpers; include the header from
`custom_insert.hlsl` instead of decoding by hand:

```hlsl
#include "Packages/com.vrchat.base/ShaderLibrary/VRCTime.cginc"
// uint VRC_GetUTCUnixTimeInSeconds();
// uint VRC_GetNetworkTimeInMilliseconds();
// void VRC_GetUTCTime(out uint hours, out uint minutes, out uint seconds, out uint milliseconds);
// void VRC_GetLocalTime(out uint hours, out uint minutes, out uint seconds, out uint milliseconds);
// int  VRC_GetTimezoneOffsetSeconds();
```

## Material Contacts: the VRCContacts buffers

A Contact Receiver with Write to Shaders enabled publishes the
contacts it detects straight to shaders, as two named buffers in the
material property blocks of up to four renderers, updated every
frame. The specification is fixed by the SDK header
`Packages/com.vrchat.base/ShaderLibrary/VRCContacts.cginc` — a
platform header, not part of any shader package or of VRCFury; SPS
is one consumer built on it.

### Setup on both sides

On the shader side, include the header and declare the buffer in
`custom_insert.hlsl`, at file scope:

```hlsl
#include "Packages/com.vrchat.base/ShaderLibrary/VRCContacts.cginc"
VRC_CONTACT_DECLARE(Example)
```

`Example` is the buffer name suffix. Read the buffers in the vertex
insertion macros (`LIL_CUSTOM_VERTEX_OS` / `LIL_CUSTOM_VERTEX_WS`)
for displacement, or in fragment hooks through `fd` — the buffers
are plain uniforms, readable in either stage.

On the avatar side, the receiver must be set to Receiver Type
`Proximity` with Write to Shaders enabled. Its fields: `Parameter`
(the Animator parameter name; leave it empty for a shader-only
receiver), the Shader Parameter Override (component field
`separateShaderParameter`; this text is the buffer name suffix, and
with `Parameter` empty the receiver updates shaders only),
`writeToMaterials` (enables the property-block write),
`writeToMaterialsContactTags` (enables the tag bits below), and
`shaderTargetRenderers` (the renderers receiving the blocks, at most
4). A buffer carries data only when both sides meet — the shader
declares the name and a receiver writes to it — and because the
write is per renderer, the same name on different renderers can hold
different receivers' data. Any tool that renames a receiver's
`Parameter` renames the shader properties with it.

The writes are performed by the VRChat client at runtime.
Av3Emulator does not emulate shader buffers, so in the Editor
the buffers are never populated: code that reads them compiles
and runs there, but the data path itself is only live in the
client.

### Buffer layout

Both buffers have 9 elements. Indices 0–7 hold the contacting
senders in proximity order (nearest first); index 8 is the receiver
itself (the header's `VRC_CONTACT_SELF_INDEX`; the limits are
`VRC_CONTACT_MAX_TARGETS` = 8 senders and `VRC_CONTACT_MAX_TAGS` =
16 tag bits).

| Buffer | Contents |
| --- | --- |
| `_VRChatContact_Transforms_<name>` (`float4x4[9]`) | World-space transforms of the senders' collision shapes; element 8 is the receiver's *root* transform, not its shape |
| `_VRChatContact_Data_<name>` (`float4[9]`) | Packed per-contact data, parallel to Transforms |

Data packing: `.x` is the owning player's ID (int32). `.y` is the
matched-tag bitmask (uint32) — bit k corresponds to the k-th entry
of the receiver's collision tag list — *except* on the self element,
where `.y` holds the current target count (int32) instead; reading
tags at the self index returns the count's bit pattern. `.z` and
`.w` are reserved and undefined. Sender scale is not in Data: it
arrives inside Transforms, as the lengths of the matrix basis
columns (columns 0, 1, 2 for X, Y, Z). Consumer wire conventions
build on this — SPS encodes socket type and control words in the
sender's world scale X, and plug radius and length in Y and Z.

### Header macros

`VRC_CONTACT_DECLARE(name)` declares both uniforms. The macro
expansion carries no trailing semicolon: terminate the
declaration with one yourself (`VRC_CONTACT_DECLARE(name);`).
Without it, a following declaration is glued onto the expansion
and compilation fails — observed first in the shadow pass when
two buffers were declared back to back. The access
macros — the complete set the header defines:

| Macro | Returns |
| --- | --- |
| `VRC_CONTACT_TARGETCOUNT(name)` | Current target count, `asint(Data[self].y)` |
| `VRC_CONTACT_TRANSFORM(name, i)` / `VRC_CONTACT_TRANSFORM_SELF(name)` | The sender's localToWorld matrix / the self element's |
| `VRC_CONTACT_WORLDPOS(name, i)` / `VRC_CONTACT_WORLDPOS_SELF(name)` | World position, the matrix's `_m03_m13_m23` |
| `VRC_CONTACT_PLAYERID(name, i)` / `VRC_CONTACT_PLAYERID_SELF(name)` | `asint(Data[i].x)` / the self element's |
| `VRC_CONTACT_TAGS(name, i)` | `asuint(Data[i].y)` |
| `VRC_CONTACT_GET_TAG(name, i, tagIndex)` | `(TAGS >> tagIndex) & 1` |
