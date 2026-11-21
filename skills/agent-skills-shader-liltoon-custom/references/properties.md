# Properties

Declaring a custom shader's material properties: the properties lilblock,
the lil property attributes, and the HLSL declarations that make the
values readable in shader code.

## The properties lilblock

Properties are declared in `lilCustomShaderProperties.lilblock` in
ordinary ShaderLab Properties syntax — the same lines that would go in a
`.shader` Properties block. The containers inline this file after
lilToon's own `Default` and mode fragments, so custom properties extend
the stock set rather than replacing it.

Use a reserved name prefix for custom properties (the template and the
official documentation use `_Custom...`). lilToon's own property names
are unprefixed or feature-prefixed; a collision with a stock property
makes the generated Properties block declare the name twice.

The worked example from the official documentation:

```
[lilVec3]       _CustomVertexWaveScale      ("Vertex Wave Scale", Vector) = (10.0,10.0,10.0,0.0)
[lilVec3]       _CustomVertexWaveStrength   ("Vertex Wave Strength", Vector) = (0.0,0.1,0.0,0.0)
                _CustomVertexWaveSpeed      ("Vertex Wave Speed", float) = 10.0
[NoScaleOffset] _CustomVertexWaveMask       ("Vertex Wave Mask", 2D) = "white" {}
```

## lil property attributes

Attributes are `MaterialPropertyDrawer` subclasses in the lilToon editor;
in a property declaration they select a specialised inspector widget for
an ordinary property type. The set in 2.4.* (29 in use in the shipped
property blocks):

| Attribute | Widget |
| --- | --- |
| `[lilHDR]` | HDR colour field with hex-code input |
| `[lilToggle]`, `[lilToggleLeft]` | Toggle that sets no shader keyword (left-aligned variant) |
| `[lilAngle]` | Edits a radian-stored value in degrees |
| `[lilVec2]`, `[lilVec2R]` | Two-component editor (R: range sliders) |
| `[lilVec3]`, `[lilVec3B]`, `[lilVec3Float]` | Three-component editors (B: last component as a bool) |
| `[lilFF]`, `[lilFFFF]`, `[lilFFFB]`, `[lilFRFR]`, `[lil3Param]` | Packed editors for float4-carried parameter sets; the name encodes the per-component widget kinds (F float field, R range, B bool) |
| `[lilEnum]`, `[lilEnumLabel]` | Popup enum from the attribute arguments (Label: renders the value as a label) |
| `[lilHSVG]` | Hue/saturation/value/gamma tone-correction editor |
| `[lilUVAnim]` | UV animation parameters (ST plus scroll/rotate) |
| `[lilDecalAnim]`, `[lilDecalSub]` | Decal animation / decal sub-parameter editors |
| `[lilColorMask]` | Colour-channel mask editor |
| `[lilLOD]` | LOD value editor |
| `[lilBlink]` | Blink-timing editor |
| `[lilDissolve]`, `[lilDissolveP]` | Dissolve parameter groups (P: position/point variant) |
| `[lilOLWidth]` | Outline-width editor |
| `[lilGlitParam1]`, `[lilGlitParam2]` | Glitter parameter groups |
| `[lilALUVParams]`, `[lilALLocal]` | AudioLink UV parameters / AudioLink-as-local |

Standard Unity attributes (`[NoScaleOffset]`, `[HDR]`, `[Enum(...)]`,
...) work as usual. Prefer a lil attribute when the value's editing shape
matches one: the stock lilToon inspector draws lil-attributed properties
with the same widgets it uses for its own.

Decorator attributes are the exception. `[Header(...)]` and
`[Space]` in the lilblock attach to the property that follows
them, and the stock draw path (`MaterialEditor.ShaderProperty`)
adds the decorator heights to that property's row and draws the
header text inside the inspector box: a `[Space(30)]` +
`[Header]` + `[Space]` stack measured 60 px added to one row
(78 px against the plain 18 px). lilToon's own lilblocks use
no `[Header]` at all (0 of 109 files in 2.4.*). Group
properties with blank lines and order in the lilblock, and put
section structure in the inspector, not in decorators.

No attribute on a custom property generates a shader variant.
`[KeywordEnum]`, `[Toggle]` and `[Enum]` — like the lil attributes
above — only choose the inspector widget; lilToon emits no
`multi_compile` for custom properties. A mode or enable property
on a custom shader is therefore a float uniform, and the shader
selects on it with a runtime branch. Variants for a custom
property exist only where the author writes the
`lil_multi_compile_*` pragmas (Passes and Insert).

## The HLSL counterparts

Every property needs its declaration in `custom.hlsl`, in one of two
macros. Both expand inside `lil_common_input.hlsl`:

- `LIL_CUSTOM_PROPERTIES` — emitted inside the material uniform block,
  after the stock properties. Scalar and vector uniforms go here.
- `LIL_CUSTOM_TEXTURES` — emitted after all stock texture and sampler
  declarations. Textures go here, declared with `TEXTURE2D(name)` (and
  `TEXTURE2D_ARRAY` / `TEXTURECUBE` forms where needed).

```
#define LIL_CUSTOM_PROPERTIES \
    float4  _CustomVertexWaveScale; \
    float4  _CustomVertexWaveStrength; \
    float   _CustomVertexWaveSpeed;
#define LIL_CUSTOM_TEXTURES \
    TEXTURE2D(_CustomVertexWaveMask);
```

Types must match the property declarations: a `Vector` property is a
`float4` uniform, a `float` property a `float`, a `2D` property a
`TEXTURE2D` plus the implicit `_ST`/`_TexelSize` companions only if you
declare them — lilToon does not generate them for custom textures, so
declare `_CustomTex_ST` yourself if the code needs it.

## Samplers

Do not declare sampler states. lilToon declares two shared states and
aliases the familiar names onto them; custom code samples with the
aliases:

| Name in custom code | Resolves to |
| --- | --- |
| `sampler_linear_repeat`, `sampler_trilinear_repeat` | `lil_sampler_trilinear_repeat` |
| `sampler_linear_clamp`, `sampler_trilinear_clamp`, `lil_sampler_linear_repeat`, `lil_sampler_trilinear_clamp` | `lil_sampler_linear_clamp` |

Note the consequence: `sampler_linear_repeat` is the trilinear-repeat
sampler, not a distinct linear one. The sampling macros that consume
these states are catalogued in lilToon Surface.

The exception to "do not declare": a dedicated `SamplerState` declared
in `custom_insert.hlsl`, for a sampling need the two shared states do
not cover (a shipped shader samples its vertex-stage mask through its
own point-clamp state). It belongs in the insert file, with the reason
recorded; the macro channel (`custom.hlsl`) stays sampler-free.

## Display names as localisation keys

A property's display string does not have to be literal text. lilToon's
inspector resolves display names through its localisation lookup, so a
custom shader can write keys instead — a shipped package declares
properties like

```
[lilEnum] _CustomAlphaMask2ndMode ("sAlphaMaskModes", Int) = 0
[NoScaleOffset] _CustomAlphaMask2nd ("scAlphaMask2nd", 2D) = "white" {}
```

and ships a TSV language file (a `Language` header row, then one row
per key with a column per language) in its Editor folder. The derived
inspector loads it in `LoadCustomProperties` with
`LoadCustomLanguage("<the TSV asset's GUID>")`, after which the keys
resolve in every language the file carries. `[lilEnum]` option labels
are keys the same way. Use literal names when the shader ships in one
language; use keys when it ships localised — do not mix the two within
one property group.

The file format and the load mechanics are fixed by lilToon's
language manager; a file that deviates fails silently, so write to
this shape:

- Columns, in order: the key, then one column per language in
  lilToon's fixed set and order — en-US, ja-JP, ko-KR, zh-Hans,
  zh-Hant. Columns in any other order resolve to the wrong language
  with no error. A language setting outside the set resolves as
  en-US.
- The header row is skipped. At load, only the active language's
  column is read into the custom dictionary; the other columns are
  not held, so the file is re-read when the inspector loads
  properties again under another language.
- A row whose columns do not reach the active language removes the
  key from the custom dictionary instead of storing an empty string:
  resolution falls back to lilToon's built-in dictionary, and from
  there to the string as written. Partial translations degrade the
  same way.
- Lookup order is the custom dictionary first, then lilToon's
  built-in one, so a custom file can also shadow a built-in key;
  do this only deliberately.

A second route exists in shipped tooling and does not use
`LoadCustomLanguage` at all: the package's editor code reads a
per-language file (gettext `.po`, JSON and `.resx` forms are in
use) and merges the active language's entries straight into the
language manager's dictionary. That dictionary is a private
static field of `lilLanguageManager`, so the merge goes through
reflection; lookups consult the same dictionary, so injected
keys resolve exactly like loaded ones, and none of the loader
behaviour above (one column at load, short-row key removal)
applies. The cost is the private field itself: its owning class
has moved between lilToon versions (older versions hold the
dictionary on the inspector class), so this route needs a
version adaptation layer of its own, and if the field moves
again the reflection finds nothing and every key falls back to
the string as written.
