# Worked example: vertex wave

The example from lilToon's official custom-shader documentation,
assembled end to end so the file-by-file flow is visible in one place. It
adds a mask-driven sine displacement to the vertex stage. The code is
the documentation's own; the surrounding notes state which procedure
step each part belongs to.

## 1. Properties (P-04) — lilCustomShaderProperties.lilblock

```
[lilVec3]       _CustomVertexWaveScale      ("Vertex Wave Scale", Vector) = (10.0,10.0,10.0,0.0)
[lilVec3]       _CustomVertexWaveStrength   ("Vertex Wave Strength", Vector) = (0.0,0.1,0.0,0.0)
                _CustomVertexWaveSpeed      ("Vertex Wave Speed", float) = 10.0
[NoScaleOffset] _CustomVertexWaveMask       ("Vertex Wave Mask", 2D) = "white" {}
```

Two vectors edited as three-component fields (`[lilVec3]`), one float,
one texture with the standard `[NoScaleOffset]` attribute.

## 2. HLSL declarations (P-05) — custom.hlsl

```
#define LIL_CUSTOM_PROPERTIES \
    float4  _CustomVertexWaveScale; \
    float4  _CustomVertexWaveStrength; \
    float   _CustomVertexWaveSpeed;
#define LIL_CUSTOM_TEXTURES \
    TEXTURE2D(_CustomVertexWaveMask);
```

The mask has no `_ST` companion declared: the code samples it with the
raw `input.uv0`, so none is needed.

## 3. Vertex insertion (P-06) — custom.hlsl

```
#define LIL_CUSTOM_VERTEX_OS \
    float3 customWaveStrength = LIL_SAMPLE_2D_LOD(_CustomVertexWaveMask, sampler_linear_repeat, input.uv0, 0).r * _CustomVertexWaveStrength.xyz; \
    positionOS.xyz += sin(LIL_TIME * _CustomVertexWaveSpeed + dot(positionOS.xyz, _CustomVertexWaveScale.xyz)) * customWaveStrength;
```

Points the example demonstrates:

- Object-space hook: the displacement is in `positionOS`, so every pass
  that shares the vertex body (forward, shadow caster, depth) displaces
  identically and shadows stay consistent with the surface.
- The mask is sampled with `LIL_SAMPLE_2D_LOD` at LOD 0 — a vertex-stage
  texture fetch needs an explicit LOD.
- The sampler is lilToon's shared `sampler_linear_repeat` alias; no
  sampler state is declared.
- Time is `LIL_TIME` (lilToon's time macro), not `_Time` directly.

## 4. Inspector (P-09)

The documentation's flow declares a `MaterialProperty` per custom
property in the derived inspector, loads it in `LoadCustomProperties`
with `FindProperty`, and draws it in `DrawCustomProperties`. For this
example the four properties are `_CustomVertexWaveScale`,
`_CustomVertexWaveStrength`, `_CustomVertexWaveSpeed` and
`_CustomVertexWaveMask`; drawing them with
`m_MaterialEditor.ShaderProperty` inside the template's foldout section
is sufficient, because the `[lilVec3]` attributes already give the
vector properties their specialised widgets in any inspector that uses
lilToon's drawers.

## What the example does not need

- No `LIL_REQUIRE_APP_*` define: it reads only `input.uv0` and
  `positionOS`, both unconditional.
- No fragment hook and no `custom_insert.hlsl` code: nothing depends on
  Unity APIs beyond what lilToon's includes already provide.
- No variant trimming changes: the hook is in `custom.hlsl`, which every
  container in the set shares, so the wave applies to all variants the
  project keeps.
