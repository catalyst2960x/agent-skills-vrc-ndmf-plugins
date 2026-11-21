# Platform support: what lilToon cannot do on Quest, Android and mobile

lilToon is a PC-first shader system, and a custom shader inherits its
platform ceiling unchanged. This file lists, per item, what lilToon
cannot do on mobile targets and what a solution would take in each
case — including the cases where the honest entry is "no solution
inside lilToon". "Quest" means the Meta Quest line (Snapdragon XR
SoCs, Adreno GPUs, tile-based rendering, Unity's Android build on
OpenGL ES 3 / Vulkan); "Android / mobile" means Unity Android and iOS
targets generally.

The items fall in three groups, by why they fail: platform policy,
shader stages the mobile APIs do not have, and inputs or costs the
mobile GPU does not provide cheaply. The group decides what kind of
solution is even possible.

## Group 1 — excluded by platform policy (VRChat)

**lilToon cannot appear on a Quest/Android avatar at all.** VRChat
permits only the avatar shaders shipped in the SDK under
`VRChat/Mobile` on Android; no lilToon shader, and therefore no
lilToon custom shader with any feature set, is permitted. A PC avatar
seen from Quest renders through a fallback or an impostor, not
through its lilToon materials.

*What a solution would take:* a separate mobile avatar built on
`VRChat/Mobile` materials — a second material set maintained beside
the PC one. That is a product decision outside the shader; no
lilToon setting, variant, or feature selection changes it. (Worlds
are different: custom shaders are allowed in Android worlds, and
Groups 2 and 3 decide what works there. Outside VRChat, in a
standalone Unity app, this group does not apply.)

## Group 2 — shader stages mobile does not have

Unity's compilation targets draw this line (Unity Manual, Shader
Compilation Targets): OpenGL ES targets top out at `es3.1`
(`#pragma target 4.5`), which has **no geometry shaders and no
tessellation shaders**; geometry requires target 4.0 (unsupported on
every OpenGL ES), and tessellation compiles only for D3D11-class
targets. Two lilToon families sit above the line:

**Fur cannot run.** The fur family (`lts_fur*`) builds its fins in a
geometry stage (`#pragma geometry`, `maxvertexcount` 32–46). The
passes do not compile for mobile targets, and lilToon has no
vertex-only fur path to fall back to. The template's own
geometry-shader example (`LIL_CUSTOM_SAFEVERT`) is PC-only for the
same reason. Mobile also loses the vertex ID on this path
(`LIL_NOT_SUPPORT_VERTEXID` is defined for `SHADER_API_MOBILE`), which
the fur randomisation and the ID mask use where available.

*What a solution would take:* a second, vertex-only implementation of
the effect (displacement or pre-built fin geometry instead of a
geometry stage) shipped as a separate mobile variant. lilToon does
not ship one; inside the template system there is no solution — a
mobile target excludes the fur family.

**Tessellation cannot run.** The tessellation family (`lts_tess*`)
runs hull and domain stages (`lil_tessellation.hlsl`). Same
consequence: no compilation for mobile targets, and no reduced form.

*What a solution would take:* geometry-side substitution — a denser
base mesh, or displacement baked into the mesh — chosen per asset.
Nothing in the shader can replace a stage the API does not have.

## Group 3 — compiles, but the input or the budget is not there

**Refraction and Gem cannot afford their background.** On BRP these
families capture the framebuffer with
`GrabPass { "_lilBackgroundTexture" }`. A GrabPass forces a tile-based
GPU to flush its on-chip tile memory to main memory mid-frame and
read it back — at VR resolutions and frame rates, the most expensive
thing a material can do. On URP the same families read the opaque
texture instead, which costs the pipeline a full-screen opaque pass
first. lilToon offers no cheaper approximation of either family.

*What a solution would take:* replacing the captured background with
a static approximation (a cubemap or matcap-style reflection) in a
mobile variant, or dropping refraction/gem looks on mobile. Both are
design changes to the shader, not settings.

**Depth fade has no input.** The feature samples the scene depth
texture, which exists only if the pipeline renders a depth prepass.
Mobile pipelines do not by default — the prepass is another
full-screen pass — so the feature silently evaluates against nothing
unless the project pays for the prepass.

The same condition governs custom depth-driven code on every
platform, not only mobile. Availability is decided by the camera and
pipeline side, and it is independent of shadow-casting lights: a
scene lit only by light probes may or may not provide the texture,
and on VRChat the world's camera configuration decides. Gate the
sampling on lilToon's availability flag, `LIL_ENABLED_DEPTH_TEX`,
and return the feature's neutral value when the flag is off, so the
material renders exactly as stock lilToon where the texture is
absent. Sampling first and interpreting whatever comes back is not
equivalent: an unbound depth texture does not read as a clean "far"
on every platform.

The Editor obeys the same rule: a Scene or Game view whose camera
does not request a depth texture leaves the gate closed, so a
depth-driven effect passes through silently in the Editor while
working in a world that requests one. A test fixture for such an
effect must set its camera's `depthTextureMode` to `Depth`.

*What a solution would take:* enabling the depth texture in the
pipeline (a project-level cost decision), or designing the look
without depth fade on mobile.

**SDF and MSDF silently degrade.** lilToon's MSDF mode (the
"As MSDF" option of the 2nd/3rd main layers) reconstructs coverage
from the sampled distance field as
`saturate((sd - 0.5) / clamp(fwidth(sd), 0.01, 1.0))` — the edge width
is entirely derivative-driven, and lilToon places no platform guard
or mobile adaptation around it. On mobile, two things change under
it without any error: derivative behaviour on tile GPUs (2×2-quad
derivatives, dynamic resolution, stereo rendering) shifts the
computed edge width, and the distance data itself is fragile —
multi-channel distance fields do not survive the block compression
mobile builds apply to textures, so the median reconstruction reads
corrupted distances. The same applies to SDF techniques generally,
including SDF-based custom shaders built on this template: they
compile, and produce a wrong or broken edge. Nothing reports it.

*What a solution would take:* keeping distance-field textures
uncompressed in the mobile build and accepting a wider, softer edge —
or, for text and decals that must stay crisp, baking them into the
main texture for the mobile variant. lilToon provides neither a
mobile MSDF path nor a warning.

**Extra passes are a budget lilToon does not manage.** Two-pass
transparent and ForwardAdd variants are legal on mobile, but each
additional pass re-reads and re-writes the framebuffer through main
memory on a tile GPU, and VRChat's own Android guidance recommends
avoiding transparency for fill-rate reasons. The lite family reduces
fragment cost within the same design; it is not a mobile version of
the shader and changes none of the groups above.

*What a solution would take:* variant discipline — trim at P-03, cut
variant groups with `lil_skip_variants_*`, prefer one-pass forms —
applied as a budget decision per product. There is no automatic
reduction.

## The pattern behind the solutions

Every solution above has the same shape: a second, reduced
implementation, plus a way to land mobile materials on it. That is
what a mobile-caring suite actually builds — Unlit_WF, for
comparison, ships dedicated Mobile variants with a reduced feature
set, names them as the full variants' ShaderLab `FallBack`, and
converts scene materials onto them at build time on mobile targets,
including a vertex-only outline in place of its geometry-shader one.
lilToon has none of the three parts (no mobile variant family, no
fallback chain, no conversion tooling), and the template system
cannot add them from inside: it generates variants of one design,
not a second reduced design with a chain onto it. A custom shader
author who needs mobile therefore designs the reduction themselves,
outside lilToon — or states the ceiling honestly in the distribution
(Distribution): "PC, and Android worlds/standalone with the Group 2
and 3 exclusions", never "Quest compatible".
