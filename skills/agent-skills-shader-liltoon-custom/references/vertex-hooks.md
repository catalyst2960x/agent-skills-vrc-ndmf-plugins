# Vertex hooks

The insertion points in lilToon's vertex stage: requesting vertex inputs,
adding interpolators, and modifying the vertex in object or world space.
All macros are defined in `custom.hlsl`.

## Requesting vertex inputs: LIL_REQUIRE_APP_*

`appdata` always contains `positionOS` (POSITION) and `uv0`–`uv3`
(TEXCOORD0–3). Further inputs are opt-in: define the macro in
`custom.hlsl` and the member is added to `appdata` (and to
`appdataCopy`, the snapshot of the inputs lilToon keeps for the passes
that need the original, unmodified values).

| Define | Member added | Semantic |
| --- | --- | --- |
| `LIL_REQUIRE_APP_TEXCOORD4` | `float2 uv4` | TEXCOORD4 |
| `LIL_REQUIRE_APP_TEXCOORD5` | `float2 uv5` | TEXCOORD5 |
| `LIL_REQUIRE_APP_TEXCOORD6` | `float2 uv6` | TEXCOORD6 |
| `LIL_REQUIRE_APP_TEXCOORD7` | `float2 uv7` | TEXCOORD7 |
| `LIL_REQUIRE_APP_COLOR` | `float4 color` | COLOR |
| `LIL_REQUIRE_APP_NORMAL` | `float3 normalOS` | NORMAL |
| `LIL_REQUIRE_APP_TANGENT` | `float4 tangentOS` | TANGENT |
| `LIL_REQUIRE_APP_VERTEXID` | `uint vertexID` | SV_VertexID |
| `LIL_REQUIRE_APP_PREVPOS` | `float3 previousPositionOS` | TEXCOORD4 (motion vectors) |
| `LIL_REQUIRE_APP_PREVEL` | `float3 precomputedVelocity` | TEXCOORD5 (motion vectors) |

`LIL_REQUIRE_APP_POSITION` and `LIL_REQUIRE_APP_TEXCOORD0`–`TEXCOORD3`
appear in the template's commented menu but are no-ops in 2.4.*: those
inputs are unconditional and the defines sit inside a comment block in
`lil_common_appdata.hlsl`. Defining them is harmless and documents
intent; relying on them to *add* anything is wrong.

Note the channel conflict: PREVPOS/PREVVEL occupy TEXCOORD4/5, the same
channels as `uv4`/`uv5`. Do not combine them.

## Adding interpolators: LIL_CUSTOM_V2F_MEMBER and LIL_V2F_FORCE_*

Custom interpolators are declared inside the pass's `v2f` struct
through an 8-parameter macro. lilToon calls the macro once per pass,
handing it that pass's free TEXCOORD indices as `id0`–`id7` — the free
slots differ per pass, so members are written against the parameters,
never against fixed slot numbers:

```
#define LIL_CUSTOM_V2F_MEMBER(id0,id1,id2,id3,id4,id5,id6,id7) \
    float wavePhase : TEXCOORD##id0; \
    float3 waveData : TEXCOORD##id1;
```

Use only the indices you need; unused parameters stay unused. The fur
pass has the geometry-stage analogue `LIL_CUSTOM_V2G_MEMBER` with the
same shape.

Stock members that a pass includes only on demand can be forced present
with `LIL_V2F_FORCE_<MEMBER>`: `TEXCOORD0`, `TEXCOORD1`, `POSITION_OS`,
`POSITION_WS`, `POSITION_SS`, `NORMAL`, `TANGENT`, `BITANGENT` (the
template's menu lists the set; a member already in the struct needs no
force).

## Filling custom interpolators: LIL_CUSTOM_VERT_COPY

Near its end, the vertex shader fills the `v2f` members from the
vertex inputs — its copy step. That step invokes
`LIL_CUSTOM_VERT_COPY` once. Define it to
assign your custom members:

```
#define LIL_CUSTOM_VERT_COPY \
    output.wavePhase = input.uv4.x;
```

Members not assigned here are undefined in the fragment stage. (The
template defines the macro empty; the fur vertex path has its own copy
point with the same name.)

## Modifying the vertex: LIL_CUSTOM_VERTEX_OS / LIL_CUSTOM_VERTEX_WS

Two statement macros become the bodies of `lilCustomVertexOS` and
`lilCustomVertexWS`, called from `vert()`.

### LIL_CUSTOM_VERTEX_OS — object space

Runs before world-space conversion. Available variables:

| Variable | Type | Notes |
| --- | --- | --- |
| `input` | `inout appdata` | The vertex inputs (only the members requested or unconditional) |
| `uvMain` | `inout float2` | The computed main UV |
| `positionOS` | `inout float4` | Object-space position; modify `.xyz` to displace |

### LIL_CUSTOM_VERTEX_WS — world space

Runs after the position and normal inputs are built; if the macro is
defined, `vert()` re-derives the position inputs after it, and calls it
a second time for the one-pass-outline position. Available variables:

| Variable | Type | Notes |
| --- | --- | --- |
| `input` | `inout appdata` | As above |
| `uvMain` | `inout float2` | As above |
| `vertexInput` | `inout lilVertexPositionInputs` | `positionWS`, `positionVS`, `positionCS`, `positionSS` |
| `vertexNormalInput` | `inout lilVertexNormalInputs` | `tangentWS`, `bitangentWS`, `normalWS` |

The worked example of an object-space displacement — the official
documentation's vertex wave, which samples a mask with
`LIL_SAMPLE_2D_LOD` and displaces `positionOS.xyz` by a time-driven
sine — is in Worked Example, end to end.

Displacement that must stay consistent across passes (shadow caster,
depth) belongs in object space where possible: the same `vert()` body
serves the forward and shadow passes, while a world-space hook that
depends on pass-specific data can desynchronise them.

For non-trivial work, the macro body is a call, not the computation:
a shipped intrusion shader's whole object-space hook is
`positionOS.xyz = calcIntrudeOffset(positionOS.xyz, uvMain);`, with
`calcIntrudeOffset` defined in `custom_insert.hlsl` (it converts with
`lilTransformOStoWS`, tests the world position against the camera near
plane, and converts back with `lilTransformWStoOS`). The insert file
is included before the vertex body runs, so its functions are callable
from both vertex macros — see Fragment Hooks (Shape of a hook body)
for the same dispatch shape in the fragment stage.

## Running in a geometry stage: LIL_CUSTOM_SAFEVERT

A flag define with no body. Normally, a vertex that fails lilToon's
culling condition leaves `vert()` through an early return. A geometry
shader cannot use that shape: it needs to call the vertex body once
per vertex and make its own decision. With `LIL_CUSTOM_SAFEVERT`
defined, the early return becomes a conditional block wrapping the
body, so the body can run inside a geometry shader (the documented
geometry-shader example uses this). Define it only for shaders whose
passes actually invoke the vertex code from a geometry stage.
