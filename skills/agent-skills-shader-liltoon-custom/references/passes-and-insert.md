# Passes and the insert file

Writing code that only some passes may see: the per-pass defines, the
`custom_insert.hlsl` channel, and the `#undef` pattern for passes where
a hook or function conflicts.

## Two insertion channels

| Channel | File | Where it lands |
| --- | --- | --- |
| Macro channel | `custom.hlsl` | Included with lilToon's common includes; the `LIL_CUSTOM_*` macros and fragment hooks are consumed by the vertex/fragment bodies. Suited to code written against lilToon's own types and helpers. |
| Insert channel | `custom_insert.hlsl`, via `lilCustomShaderInsert.lilblock` and the containers' `lilSubShaderInsert` directive | Substituted inside every pass's HLSLPROGRAM. Suited to functions and includes that depend on Unity APIs or variables, which may exist in one pass and not another. |

The official documentation's rule: functions or includes that depend on
Unity functions or variables go in `custom_insert.hlsl`. If a specific
pass errors on them, `#undef` the offending macro in that pass and
redeclare what the pass needs.

## Per-pass defines

Branch with `#if defined(LIL_PASS_...)` in either channel. The
documented set:

| Define | Passes |
| --- | --- |
| `LIL_PASS_FORWARD` | ForwardBase, UniversalForward, ForwardOnly, SRPDefaultUnlit |
| `LIL_PASS_FORWARDADD` | ForwardAdd (BRP only) |
| `LIL_PASS_SHADOWCASTER` | ShadowCaster |
| `LIL_PASS_DEPTHONLY` | DepthOnly (URP, HDRP) |
| `LIL_PASS_DEPTHNORMALS` | DepthNormals (URP) — see the spelling note |
| `LIL_PASS_MOTIONVECTORS` | MotionVectors (HDRP) — see the spelling note |
| `LIL_PASS_META` | Meta |

How they arise: these defines are not written in any file you edit.
lilToon writes them into the generated shader text as it generates
each pass, which is why their spelling is verified in the exported
shader, not in the sources (P-01). Each pass's include file also
carries an include guard of the same family
(`LIL_PASS_FORWARD_INCLUDED`, `LIL_PASS_SHADOWCASTER_INCLUDED`, ...).

The generation mechanism is a family of lilToon pragmas that a pass
file declares and the importer expands into that pass's multi-compile
set plus its pass-kind define (official documentation, custom shader
format): `#pragma lil_multi_compile_forward`, `_forwardadd`,
`_shadowcaster`, `_depthonly`, `_depthnormals`, `_motionvectors`,
`_sceneselection`, `_meta`. A custom shader does not write these — the
stock pass files carry them — but they are the reason the define set
differs per pass, and the list is the checklist when a generated pass
is missing an expected define.

## Suppressing variant groups: lil_skip_variants_*

A second pragma family, usable in pass and insert code, removes whole
variant groups from the generated shader (official documentation,
same page): `#pragma lil_skip_variants_shadows` (shadow receiving),
`_lightmaps`, `_decals`, `_addlight` (vertex lights),
`_addlightshadows` (point/spot shadow receiving), `_probevolumes`,
`_ao`, `_lightlists`, `_reflections` (environment reflections). A
custom shader whose feature makes one of these groups meaningless
(for example a shader that never samples lightmaps) declares the
matching pragma instead of paying the variant count — this is the
sanctioned way to cut variants, as opposed to editing generated
multi-compile lines, which R-01 forbids.

**Spelling note (2.4.* source).** The include guards use the singular
forms `LIL_PASS_DEPTHNORMAL_INCLUDED` and `LIL_PASS_MOTIONVECTOR_INCLUDED`
for the DepthNormals and MotionVectors passes, while the documentation
table writes the pass-kind defines in the plural. Before writing
pass-conditional code for those two passes, export the generated shader
from the container importer and grep the exact define spelling your
lilToon version emits (P-10); do not copy either form on trust.

## The #undef pattern

A hook or helper that is valid in the forward pass can fail in a pass
whose inputs differ (the shadow caster has no `fd` lighting state; the
meta pass has no view vector). The documented recovery, applied in
`custom_insert.hlsl` under a pass guard:

```
#if defined(LIL_PASS_SHADOWCASTER)
    #undef LIL_CUSTOM_VERTEX_WS
    // redeclare only what this pass needs
#endif
```

Rules for using it:

- Guard the `#undef` with the pass define so other passes keep the
  original macro.
- Remember the fragment hooks exist in non-forward passes only at
  `UNPACK_V2F`; a `BEFORE_MAIN` in the shadow caster is never invoked,
  not an error — check invocation coverage in Fragment Hooks before
  assuming a hook runs everywhere.
- A `#undef` removes the macro for the rest of that pass's translation
  unit, including lilToon's own later uses in the same pass. Redeclare
  an empty or pass-appropriate form when lilToon's code still references
  the name.

## Pass coverage summary

| Pass family | Vertex hooks | Fragment hooks | Notes |
| --- | --- | --- | --- |
| Forward (normal, lite, gem, fur, refraction blur) | OS + WS + copy | All 35 points | The fur forward pass additionally has `LIL_CUSTOM_V2G_MEMBER` |
| ShadowCaster, DepthOnly, DepthNormals, Meta, MotionVectors, Universal2D | OS + WS + copy (shared `vert()` body) | `UNPACK_V2F` only | Motion vectors also auto-request PREVPOS |
