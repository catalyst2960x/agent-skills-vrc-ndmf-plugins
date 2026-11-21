# Fragment hooks

The insertion points in lilToon's fragment stage: for each pipeline
stage of the forward pass, insert code before it (`BEFORE_`) or replace
it (`OVERRIDE_`), operating on the fragment data `fd`.

## Mechanism

- Both forms are statement macros with no arguments, defined in
  `custom.hlsl` (reached by every pass through the SubShader insert).
- `lil_common_frag.hlsl` gives every `BEFORE_<POINT>` an empty default
  and every `OVERRIDE_<POINT>` a default containing the stock stage
  implementation, each guarded by `#if !defined(...)`. A custom shader
  pre-defines either macro to take part.
- The forward pass files invoke the pair in order inside `frag()`:
  `BEFORE_X` immediately before `OVERRIDE_X`. The lite, gem, fur and
  refraction-blur forward passes invoke the same pairs. The non-forward
  passes (shadow caster, depth only, depth normals, meta, motion
  vectors, universal 2D) invoke only `UNPACK_V2F`.
- Consequence for hook choice: a stage you `OVERRIDE_` is gone unless
  your code reproduces it — the default body is the implementation, for
  example `OVERRIDE_MAIN` is `LIL_GET_MAIN_TEX; LIL_APPLY_MAIN_TONECORRECTION; fd.col *= _Color;`.
  Use `BEFORE_` for additions and adjustments; use `OVERRIDE_` only to
  replace a stage's algorithm.

## Shape of a hook body: a dispatch call

In shipped custom shaders the macro body is usually not the code
itself but a one-line call to a function defined in
`custom_insert.hlsl`:

```
#define OVERRIDE_ALPHAMASK lilGetAlphaMsdfMask(fd);
#define OVERRIDE_SHADOW    lilGetShadingMsdfMask(fd LIL_SAMP_IN(sampler_MainTex));
```

This keeps `custom.hlsl` a readable list of which stages the shader
touches, keeps the implementations in ordinary functions (named,
debuggable, reusable across hooks), and lets the function use the
insert file's position in the pass. Arguments beyond `fd` — typically a
sampler — are passed with `LIL_SAMP_IN(...)`, lilToon's macro for
handing a sampler to a function, because the syntax for that differs
between the built-in pipeline and the SRPs (lilToon Surface). The
vertex macros take the same shape: a `LIL_CUSTOM_VERTEX_OS` body that
calls a transform helper from the insert file is the standard form for
non-trivial vertex work.

## The copy trap

An `OVERRIDE_` whose body is a near-verbatim copy of lilToon's stage,
with the custom lines inserted, forks that stage. The copy does not
receive lilToon's fixes or changes: on every lilToon update within the
supported range, each copied stage must be re-diffed against its new
original and re-merged by hand. A shipped mask extension carries an
insert file of about 40 KB, most of it copied emission stages — that
is the measured cost of the pattern. Prefer, in order: a `BEFORE_`
hook when the change is an adjustment; a dispatch call that wraps or
reimplements only the differing part; a full copy only when the stage's
algorithm itself is being replaced, recorded as a maintenance
obligation in the distribution's release checks (Distribution).

## Gating on feature defines

A hook that touches a lilToon feature's stage or uniforms must be
gated on that feature's `LIL_FEATURE_*` define, with the override
undefined in the `#else` branch — otherwise the shader fails to
compile in projects whose shader settings disable the feature. The
mechanism and the pattern are in lilToon Surface (The condition:
feature defines).

## The 35 hook points, in invocation order

| Point | Stage it precedes / replaces |
| --- | --- |
| `UNPACK_V2F` | Unpacking the interpolators into `fd` (the only hook in non-forward passes) |
| `CALC_DDX_DDY` | Computing the main UV derivatives |
| `ANIMATE_MAIN_UV` | Main UV animation |
| `ANIMATE_OUTLINE_UV` | Outline UV animation |
| `PARALLAX` | Parallax mapping |
| `MAIN` | Main colour sampling and tone correction |
| `OUTLINE_COLOR` | Outline colour |
| `FUR` | Fur processing (fur shaders) |
| `ALPHAMASK` | Alpha mask |
| `DISSOLVE` | Dissolve |
| `UDIMDISCARD` | UDIM discard — **OVERRIDE_ only**, no BEFORE_ form exists |
| `NORMAL_1ST` | First normal map |
| `NORMAL_2ND` | Second normal map |
| `ANISOTROPY` | Anisotropy |
| `AUDIOLINK` | AudioLink |
| `MAIN2ND` | Second main colour (decal/overlay layer) |
| `MAIN3RD` | Third main colour |
| `SHADOW` | Shadow / lighting computation |
| `BACKLIGHT` | Backlight |
| `REFRACTION` | Refraction |
| `REFLECTION` | Reflection |
| `MATCAP` | MatCap |
| `MATCAP_2ND` | Second MatCap |
| `RIMLIGHT` | Rim light |
| `RIMSHADE` | Rim shade |
| `GLITTER` | Glitter |
| `EMISSION_1ST` | First emission |
| `EMISSION_2ND` | Second emission |
| `DISSOLVE_ADD` | Dissolve edge emission |
| `BLEND_EMISSION` | Emission blending into the colour |
| `DEPTH_FADE` | Depth fade (soft particles / water edge) |
| `DISTANCE_FADE` | Distance fade |
| `DITHER` | Dithering |
| `FOG` | Fog |
| `OUTPUT` | Final output write |

The official documentation lists 30 of these; `CALC_DDX_DDY`,
`DEPTH_FADE`, `DITHER`, `RIMSHADE` and `UDIMDISCARD` are present in the
2.4.* source but undocumented there. Order matters when two hooks
interact: a `BEFORE_` hook sees exactly the `fd` state the stages above
it in this table have produced.

Minimal example (from the official documentation — rerouting the main
UV at unpack time):

```
#define BEFORE_UNPACK_V2F \
    fd.uv0 = input.uv0;
```

## The fragment data: lilFragData (`fd`)

Hooks operate on the in-scope local `fd` of type `lilFragData`
(`Shader/Includes/lil_common.hlsl`, zero-initialised by
`lilInitFragData()`). The fields a hook author works with, by group:

- **Colour** — `float4 col` (working lit colour), `float3 albedo`,
  `float3 emissionColor`.
- **Light** — `float3 lightColor`, `float3 indLightColor`,
  `float3 addLightColor`, `float attenuation`,
  `float3 invLighting` (= `saturate((1.0 - lightColor) * sqrt(lightColor))`).
- **UV** — `float2 uv0`–`uv3`, `uvMain`, `uvMat`, `uvRim`, `uvPanorama`,
  `uvScn` (screen UV), `ddxMain`, `ddyMain`; `bool isRightHand`,
  `dissolveActive`, `dissolveInvert`.
- **Position** — `float3 positionOS`, `positionWS`; `float4 positionCS`,
  `positionSS`; `float depth`, `depthObject`. `positionSS` is a 0..1
  screen position, not pixels: the fragment-stage transform
  (`lilTransformCStoSSFrag`) divides by the screen size. Pixel
  coordinates are `positionCS.xy`, which is what lilToon's own
  screen-space sampling (depth fade) uses.
- **Vectors** — `float3x3 TBN`, `float3 T`, `B`, `N` (normal), `V`
  (view), `L` (light), `origN`, `origL` (without normal map / SH),
  `headV` (stereo midpoint view), `reflectionN`, `matcapN`, `matcap2ndN`,
  `float facing`; camera basis `cameraFront/Up/Right`, `cameraMatrix`.
- **Dot products** — `float vl, hl, ln, nv, nvabs`.
- **Shading state** — `float4 triMask`, `float3 parallaxViewDirection`,
  `float2 parallaxOffset`, `float anisotropy`, `smoothness`, `roughness`,
  `perceptualRoughness`, `shadowmix` (0 in shadow), `audioLinkValue`.
- **HDRP only** — `uint renderingLayers`, `featureFlags`,
  `uint2 tileIndex`.

Fields are produced progressively down the hook table: `fd.N` is the
geometric normal until `NORMAL_1ST` runs, `fd.col` holds no main colour
until `MAIN`, and `fd.lightColor` is final only after `SHADOW`. Read the
invocation order before reading a field.
