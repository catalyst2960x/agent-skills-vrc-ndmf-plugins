# Cheatsheet — lilToon custom shaders

One page: every macro, hook and binding a custom shader uses. Details
and conditions live in the references.

## Files

| File | You put |
| --- | --- |
| `Shaders/lilCustomShaderProperties.lilblock` | Property declarations |
| `Shaders/custom.hlsl` | All `LIL_CUSTOM_*` macros, `BEFORE_*` / `OVERRIDE_*` hooks |
| `Shaders/custom_insert.hlsl` | Unity-dependent functions/includes (via `lilCustomShaderInsert.lilblock`) |
| `Shaders/lilCustomShaderDatas.lilblock` | `ShaderName`, `EditorName`, optional `Replace` / `Insert*` tags |
| `Shaders/*.lilcontainer` | One generated shader each — delete the variants you do not ship |
| `Editor/CustomInspector.cs` | Derived inspector: class name, `shaderName` const, variant table |
| `Editor/<Name>.asmdef` | `name` + file name renamed with the shader |

## Declarations (custom.hlsl)

```
#define LIL_CUSTOM_PROPERTIES   <uniforms>      // inside the material CBUFFER
#define LIL_CUSTOM_TEXTURES     TEXTURE2D(...); // after stock textures
```

Samplers: default is the shared aliases `sampler_linear_repeat` /
`sampler_linear_clamp`; never declare in custom.hlsl — a dedicated
`SamplerState` belongs in custom_insert.hlsl only, with a reason.
lilToon's own state is readable from custom code: stock uniforms,
per-texture samplers (`sampler_MainTex`), `LIL_SAMPLE_2D_ST`,
`LIL_SAMP_IN(...)`, helpers (`lilTransformOStoWS`, `lilMedian`,
`lilBlendColor`), `LIL_TIME`, `LIL_RENDER`. A feature's uniforms and
code exist only under its `LIL_FEATURE_*` define (liltoon-surface.md).

## Vertex inputs — define to add

Always present: `positionOS`, `uv0`–`uv3`.
Effective: `LIL_REQUIRE_APP_TEXCOORD4`–`7`, `_COLOR`, `_NORMAL`,
`_TANGENT`, `_VERTEXID`, `_PREVPOS`, `_PREVEL` (TEXCOORD4/5 — conflicts
with uv4/uv5).
No-ops in 2.4.*: `LIL_REQUIRE_APP_POSITION`, `_TEXCOORD0`–`3`.

## Interpolators

```
#define LIL_CUSTOM_V2F_MEMBER(id0,...,id7)  <members with TEXCOORD##idN>
#define LIL_CUSTOM_V2G_MEMBER(id0,...,id7)  fur geometry stage
#define LIL_V2F_FORCE_<MEMBER>              TEXCOORD0/1, POSITION_OS/WS/SS, NORMAL, TANGENT, BITANGENT
#define LIL_CUSTOM_VERT_COPY                <assign custom members in the vertex copy step>
```

## Vertex modification

```
#define LIL_CUSTOM_VERTEX_OS   // inout input, uvMain, positionOS
#define LIL_CUSTOM_VERTEX_WS   // + vertexInput, vertexNormalInput; runs twice for one-pass outline
#define LIL_CUSTOM_SAFEVERT    // flag: vertex body callable from a geometry stage
```

## Fragment hooks — BEFORE_ inserts, OVERRIDE_ replaces

In order: UNPACK_V2F, CALC_DDX_DDY, ANIMATE_MAIN_UV, ANIMATE_OUTLINE_UV,
PARALLAX, MAIN, OUTLINE_COLOR, FUR, ALPHAMASK, DISSOLVE, UDIMDISCARD
(OVERRIDE only), NORMAL_1ST, NORMAL_2ND, ANISOTROPY, AUDIOLINK, MAIN2ND,
MAIN3RD, SHADOW, BACKLIGHT, REFRACTION, REFLECTION, MATCAP, MATCAP_2ND,
RIMLIGHT, RIMSHADE, GLITTER, EMISSION_1ST, EMISSION_2ND, DISSOLVE_ADD,
BLEND_EMISSION, DEPTH_FADE, DISTANCE_FADE, DITHER, FOG, OUTPUT.
Non-forward passes: only UNPACK_V2F. Hooks read/write `fd`
(`lilFragData`): col, albedo, emissionColor, lightColor, uvMain, N, V, L,
smoothness, shadowmix, ...
Hook bodies are normally one-line calls to functions in
custom_insert.hlsl. A hook touching a feature's stage/uniforms is gated:
`#if defined(LIL_FEATURE_X)` … `#else #undef OVERRIDE_X`; lite-absent
stages with `#if !defined(LIL_LITE)`. An OVERRIDE that copies the stock
stage forks it — re-diff on every lilToon update.

## Pass defines — branch in either channel

`LIL_PASS_FORWARD`, `LIL_PASS_FORWARDADD`, `LIL_PASS_SHADOWCASTER`,
`LIL_PASS_DEPTHONLY`, `LIL_PASS_META`, and the DepthNormals /
MotionVectors pair whose spelling differs between the documentation
(plural) and the 2.4.* source (singular) — grep the exported shader.

## Inspector binding

`EditorName` (datas) = derived class full name → `CustomEditor`.
Virtuals: `LoadCustomProperties(props, material)`,
`DrawCustomProperties(material)`. Set `isCustomShader = true`; keep
`ReplaceToCustomShaders()` in sync with the container set.
Styles: boxOuter, boxInnerHalf, boxInner, customBox, customToggleFont.
Helpers: Foldout, DrawLine, DrawWebButton, LoadCustomLanguage, GetLoc.

## Pre-flight checklist

- [ ] Template copied whole, rename set complete, datas ShaderName and inspector shaderName identical (P-01)
- [ ] Custom code only in the custom files; nothing under `Assets/lilToon` edited (P-02)
- [ ] Variant set trimmed deliberately, and the inspector search table left consistent (P-03)
- [ ] Every custom property declared in `custom.hlsl`, datas block and inspector (P-04)
- [ ] Vertex inputs requested, interpolators within the pass budget (P-05)
- [ ] Hooks at the right stage, gated, with the stock fallback kept (P-06)
- [ ] Inspector: `isCustomShader` set, replace list in sync, display names resolved (P-08); if the foldout needs copy / paste / reset, it provides its own section menu — the stock menu covers only lilToon-registered properties (Custom Inspector, Section menu)
- [ ] Exported shader grepped; compile checked in Unity (P-10)
- [ ] Distribution: dependency floor set, lilToon license file included (P-11)
- [ ] Migration: full variant set kept until materials are moved and verified; trim after (P-12)
