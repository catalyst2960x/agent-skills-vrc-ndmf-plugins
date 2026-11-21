# lilToon macro catalogue (everything a custom shader can define or use)

One index of the macro surface, grouped by channel. "Define" macros are
written by the custom shader in `custom.hlsl`; "use" macros and
constants are provided by lilToon and read or called by custom code.
Detail and rules live in the linked references; this file is the map.
All entries are lilToon 2.4.* (Requirements R-10: re-verify spellings
against the installed version).

## Insertion macros — define in custom.hlsl

| Macro | Form | Effect | Detail |
| --- | --- | --- | --- |
| `LIL_CUSTOM_PROPERTIES` | statements | Uniform declarations, placed inside the material uniform block (CBUFFER) | Properties |
| `LIL_CUSTOM_TEXTURES` | statements | `TEXTURE2D(...)` declarations; lilToon adds `_ST`, `_TexelSize`, `_HDR` companions per texture | Properties |

Vertex inputs are not an insertion macro: they are requested with the
`LIL_REQUIRE_*` defines below, written directly in `custom.hlsl`.
Interpolators are declared with `LIL_CUSTOM_V2F_MEMBER` (Vertex body
macros).

## Vertex input requirements — define in custom.hlsl

Each define adds the member to `appdata` and `appdataCopy`. Which
members each define adds, and which of them are unconditional in
2.4.*, is in Vertex Hooks.

`LIL_REQUIRE_APP_POSITION`, `LIL_REQUIRE_APP_TEXCOORD0` … `TEXCOORD7`,
`LIL_REQUIRE_APP_COLOR`, `LIL_REQUIRE_APP_NORMAL`,
`LIL_REQUIRE_APP_TANGENT`, `LIL_REQUIRE_APP_VERTEXID`,
`LIL_REQUIRE_APP_PREVPOS`, `LIL_REQUIRE_APP_PREVEL` (the last two are
the motion-vector inputs: previous position and precomputed velocity,
on TEXCOORD4/5).

Force counterparts for interpolators: `LIL_V2F_FORCE_POSITION_OS`,
`LIL_V2F_FORCE_POSITION_WS`, `LIL_V2F_FORCE_POSITION_SS`,
`LIL_V2F_FORCE_TEXCOORD0`, `LIL_V2F_FORCE_TEXCOORD1`,
`LIL_V2F_FORCE_NORMAL`, `LIL_V2F_FORCE_TANGENT`,
`LIL_V2F_FORCE_BITANGENT` — force a stock member into `v2f` even when
lilToon would otherwise omit it.

## Vertex body macros — define in custom.hlsl

| Macro | Runs | Detail |
| --- | --- | --- |
| `LIL_CUSTOM_V2F_MEMBER(id0..id7)` | Invoked once per pass by the pass file, declaring `v2f` members against the pass's TEXCOORD indices | Vertex Hooks |
| `LIL_CUSTOM_V2G_MEMBER(id0..id7)` | The fur/geometry-pass counterpart, invoked with that pass's index set | Vertex Hooks |
| `LIL_CUSTOM_VERT_COPY` | Once, in the vertex shader's copy step that fills `v2f` from the inputs | Vertex Hooks |
| `LIL_CUSTOM_VERTEX_OS` | Body of `lilCustomVertexOS`, object space, before the stock transform | Vertex Hooks |
| `LIL_CUSTOM_VERTEX_WS` | Body of `lilCustomVertexWS`, world space, after it | Vertex Hooks |
| `LIL_CUSTOM_SAFEVERT` | Flag (no body): replaces `vert()`'s early-return culling with a conditional block, for geometry-stage use | Vertex Hooks |

## Fragment hooks — define in custom.hlsl

For each point: `BEFORE_<POINT>` inserts statements before the stock
stage; `OVERRIDE_<POINT>` replaces the stage (its default define is
the implementation). Bodies are normally one-line calls to functions
in `custom_insert.hlsl`. Full semantics: Fragment Hooks.

The 35 points, in invocation order (Fragment Hooks gives each one's
stage): `UNPACK_V2F`, `CALC_DDX_DDY`, `ANIMATE_MAIN_UV`,
`ANIMATE_OUTLINE_UV`, `PARALLAX`, `MAIN`, `OUTLINE_COLOR`, `FUR`,
`ALPHAMASK`, `DISSOLVE`, `UDIMDISCARD` (OVERRIDE only), `NORMAL_1ST`,
`NORMAL_2ND`, `ANISOTROPY`, `AUDIOLINK`, `MAIN2ND`, `MAIN3RD`,
`SHADOW`, `BACKLIGHT`, `REFRACTION`, `REFLECTION`, `MATCAP`,
`MATCAP_2ND`, `RIMLIGHT`, `RIMSHADE`, `GLITTER`, `EMISSION_1ST`,
`EMISSION_2ND`, `DISSOLVE_ADD`, `BLEND_EMISSION`, `DEPTH_FADE`,
`DISTANCE_FADE`, `DITHER`, `FOG`, `OUTPUT`.

## Pass defines — provided by lilToon, test in custom code

Written into the generated shader during generation; not present in
any editable file. Each pass also sets an include guard
(`LIL_PASS_<NAME>_INCLUDED`) used by the `#undef` pattern.

`LIL_PASS_FORWARD`, `LIL_PASS_FORWARDADD`,
`LIL_PASS_SHADOWCASTER`, `LIL_PASS_DEPTHONLY`,
`LIL_PASS_DEPTHNORMALS`, `LIL_PASS_MOTIONVECTORS`, `LIL_PASS_META`.
Passes and Insert holds the per-pass table and the 2.4.* spelling
note for the DepthNormals and MotionVectors guards — read it before
writing pass-conditional code.

## Generation pragmas — declared in pass files, expanded by the importer

A custom shader does not normally write these (the stock pass files
carry them), but they produce the pass defines above and the variant
groups below, so they are the checklist when a generated shader's
define set looks wrong (Passes and Insert):

`#pragma lil_multi_compile_forward`, `_forwardadd`, `_shadowcaster`,
`_depthonly`, `_depthnormals`, `_motionvectors`, `_sceneselection`,
`_meta` — expand to the pass's multi-compile set and its `LIL_PASS_*`
define.

`#pragma lil_skip_variants_shadows`, `_lightmaps`, `_decals`,
`_addlight`, `_addlightshadows`, `_probevolumes`, `_ao`,
`_lightlists`, `_reflections` — remove that variant group from the
generated shader; this family *is* written by custom code, in pass or
insert files, when a group is meaningless for the shader.

## Vertex-stage helpers — provided, use in insert-file code

`LIL_MATRIX_M`, `LIL_MATRIX_I_M`, `LIL_MATRIX_V`, `LIL_MATRIX_P`,
`LIL_MATRIX_VP` (the pipeline matrices under one name per space);
`LIL_VERTEX_POSITION_INPUTS(positionOS, out v)` and
`LIL_RE_VERTEX_POSITION_INPUTS(v)` (recompute after a world-space
displacement); `LIL_VERTEX_NORMAL_TANGENT_INPUTS(normalOS, tangentOS,
out v)`. Signatures and the full `lilTransform*` family: lilToon
Surface.

## Sampling — provided by lilToon, use in custom code

The two shared sampler states, `sampler_linear_repeat` and
`sampler_linear_clamp` (with their alias families — Properties holds
the resolution table), the per-texture aliases
(`sampler_<TextureName>`, for example `sampler_MainTex`), and the
sampling macros `LIL_SAMPLE_2D`, `LIL_SAMPLE_2D_ST`,
`LIL_SAMPLE_2D_LOD`, `LIL_SAMPLE_1D`, `LIL_SAMPLE_1D_LOD` and
`LIL_SAMP_IN(...)`. Forms and use: lilToon Surface (Per-texture
samplers and sampling macros).

## Constants and feature defines — provided, test in custom code

| Name | Meaning |
| --- | --- |
| `LIL_FEATURE_<FEATURE>` | The named feature is enabled in the project's lilToon settings; its uniforms and code exist only under this define (gate on it — R-07) |
| `LIL_LITE` | The variant belongs to the lite family; several stages do not exist |
| `LIL_RENDER` | Compile-time constant for the rendering-mode family (transparent = 2); stock stages branch on it |
| `LIL_TIME` | lilToon's time value for shader animation |
