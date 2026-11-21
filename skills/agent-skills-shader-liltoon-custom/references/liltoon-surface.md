# lilToon's surface available to custom code

Custom code is included inside lilToon's own shader, so lilToon's state
is in scope: its uniforms, its per-texture samplers, its sampling
macros, and its helper functions. Shipped custom shaders build on this
surface heavily — a mask extension, for example, reads the stock alpha
mask's mode and scale uniforms and reuses the stock sampling macros
rather than re-declaring any of it. This reference states what can be
relied on, and the condition attached to most of it: the feature
defines.

## Stock uniforms

Every property lilToon declares is a uniform in the same material block
your `LIL_CUSTOM_PROPERTIES` extend, so custom code can read stock
values directly (for example `_AlphaMaskMode`, `_AlphaMaskScale`,
`_EmissionBlend`, the main texture's `_MainTex_ST`). Reading is safe;
writing a stock uniform from custom code changes lilToon's own stage
behaviour and should be a deliberate design act, not a side effect.

The companions exist for stock textures (`_MainTex_ST`,
`_MainTex_TexelSize`, ...). For custom textures they do not — declare
them yourself (Properties).

## Per-texture samplers and sampling macros

Besides the two shared sampler states (Properties), lilToon generates a
sampler alias per stock texture (`sampler_MainTex`, `sampler_BumpMap`,
...), and custom code sampling a stock texture uses that texture's
alias. The sampling macros wrap the forms custom code needs:

| Macro family | Use |
| --- | --- |
| `LIL_SAMPLE_2D(tex, sampler, uv)` | Plain sample |
| `LIL_SAMPLE_2D_ST(tex, sampler, uv)` | Sample with the texture's `_ST` applied to `uv` |
| `LIL_SAMPLE_2D_LOD(tex, sampler, uv, lod)` | Explicit-LOD sample (the vertex-stage form) |
| `LIL_SAMPLE_1D` / `LIL_SAMPLE_1D_LOD` | 1D forms (gradation textures) |
| `LIL_SAMPLE_2D_CS(tex, positionCS)` / `LIL_GET_DEPTH_TEX_CS(positionCS)` | Screen-space sample at a pixel position (the depth-texture form) |

The `_CS` forms load a texel rather than filtering: on the
texel-load path the position is flipped by `lilCameraDepthTexel`
and truncated to a texel index, so a fractional position reads a
neighbouring texel, up to one pixel off. Code that reconstructs a
world or view position from a depth fetched this way must
reconstruct at the centre of the texel actually read; pairing a
fractional fetch position with a ray through the intended
position tilts the sample off sloped surfaces, and one-sided
estimators read the offset as occlusion.

Functions that receive a sampler as a parameter are written with
lilToon's sampler-passing macros, because how a sampler is passed to a
function differs between the built-in render pipeline (where samplers
are global states) and the SRPs (where they are explicit parameters):
`LIL_SAMP_IN(sampler_MainTex)` at the call site appends the sampler
argument in whichever form the pipeline being compiled expects, and
the receiving function's declaration uses the matching macro. Use it
when a hook dispatches to a function in the insert file that must
sample with a caller-chosen sampler (Fragment Hooks shows the dispatch
shape).

## Helper functions and constants

The includes custom code is compiled with provide the following
(official documentation, utilities page, cross-checked against the
2.4.* source). Use these instead of raw matrix multiplications and
hand-rolled equivalents, so custom code converts spaces and blends
exactly the way lilToon's own stages do.

Matrix macros (the pipeline's matrices under one name per space):
`LIL_MATRIX_M` (model/world), `LIL_MATRIX_I_M` (inverse-transpose
world), `LIL_MATRIX_V`, `LIL_MATRIX_P`, `LIL_MATRIX_VP`.

Space conversions:

| Function | Conversion |
| --- | --- |
| `lilTransformOStoWS(float4 / float3)` | object → world position |
| `lilTransformWStoOS(float3)` | world → object position |
| `lilTransformWStoVS(float3)` | world → view |
| `lilTransformWStoCS(float3)` | world → clip |
| `lilTransformVStoCS(float3)` | view → clip |
| `lilTransformCStoSS(float4)` | clip → screen |
| `lilTransformCStoSSFrag(float4)` | clip → screen, fragment-stage form |
| `lilToAbsolutePositionWS(float3)` | camera-relative → absolute world (HDRP) |
| `lilTransformDirOStoWS(dir, doNormalize)` / `lilTransformDirWStoOS(dir, doNormalize)` | direction vectors |
| `lilTransformNormalOStoWS(normal, doNormalize)` | normals |
| `lilViewDirection(positionWS)` | unnormalised view direction |
| `lilHeadDirection(positionWS)` | view direction from the stereo midpoint (parallax-free in VR) |
| `lilCStoGrabUV(positionCS)` | clip → GrabPass UV |

Whole-vertex transforms, for insert-file vertex code:

| Macro | Effect |
| --- | --- |
| `LIL_VERTEX_POSITION_INPUTS(positionOS, out vertexInput)` | Fills `vertexInput` with `positionWS`, `positionVS`, `positionCS`, `positionSS` |
| `LIL_RE_VERTEX_POSITION_INPUTS(vertexInput)` | Recomputes the set after `positionWS` was modified — required after any world-space displacement in the insert file, or the later spaces keep the pre-displacement values |
| `LIL_VERTEX_NORMAL_TANGENT_INPUTS(normalOS, tangentOS, out vertexNormalInput)` | Fills `tangentWS`, `bitangentWS`, `normalWS` |

Colour and misc:

| Function | Behaviour |
| --- | --- |
| `lilCalcUV(uv, tex_ST, tex_ScrollRotate)` | The stock UV transform (scale/offset plus scroll/rotation) |
| `lilBlendColor(dstCol, srcCol, srcA, blendMode)` | Blend by mode: 0 normal, 1 add, 2 screen, 3 multiply |
| `lilUnpackNormalScale(tex, scale)` | Normal-map unpack with strength |
| `lilTooning(x)` | The stock toon quantisation of a value |
| `lilMedian(float3)` | Three-channel median (the MSDF evaluation) |
| `lilCalcBlink(...)` | The stock blink curve used by emission animation |

Constants: `LIL_TIME` (lilToon's time value, used by the stock
animation code and the worked example); `LIL_RENDER`, a compile-time
constant lilToon sets per variant identifying the rendering-mode
family the variant belongs to (the transparent family is 2) — stock
stages branch on it for mode-specific behaviour, and a copied stage
keeps its branches meaningful only if the constant is honoured.

The full helper set is lilToon's internal API: it is not versioned for
custom shaders, and P-01's verification includes grepping the installed
version for any helper the design depends on.

## The condition: feature defines (LIL_FEATURE_*)

lilToon's features are project-wide: lilToon's project settings (the
settings asset where features are switched for every lilToon shader in
the project) emit `LIL_FEATURE_<FEATURE>` defines, and a feature's
uniforms, textures and stage code are compiled only under its define.
Two consequences for custom code:

1. **Referencing a feature's state requires the define.** Code that
   reads `_AlphaMaskMode` compiles only while `LIL_FEATURE_ALPHAMASK`
   is defined. In a project whose shader settings disable the feature,
   the uniform does not exist and the reference is a compile error.
2. **Replacements must hand the stage back.** The shipped pattern for
   an override that depends on a feature:

```
#if defined(LIL_FEATURE_ALPHAMASK)
    void ApplyCustomAlphaMask(inout lilFragData fd) { ... }
#else
    #undef OVERRIDE_ALPHAMASK
#endif
```

   With the feature off, the `#undef` removes the override and the
   stock default (an empty or stock stage) applies again.

The same gating applies to family differences: stages the lite family
does not have are excluded with `#if !defined(LIL_LITE)` around their
overrides, so the shared `custom.hlsl` compiles for every family the
project keeps.
