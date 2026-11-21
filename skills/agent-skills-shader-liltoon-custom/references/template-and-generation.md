# Template and generation

How a lilToon custom shader is put together: the template's files, the
container format the importer expands, the shader-data tags, and the
variant set.

## The template

Template.zip unpacks to a single root folder (`Template`) containing two
folders. Rename the root folder to the product's name.

### Shaders/ — the files you edit

| File | Role |
| --- | --- |
| `custom.hlsl` | The main insertion file. Holds the `LIL_CUSTOM_*` macro definitions: custom uniforms and textures, vertex-input requests, interpolator additions, vertex insertion code, and the fragment `BEFORE_*` / `OVERRIDE_*` hooks. Ships with every hook present as a commented-out menu plus a field reference for `fd`. |
| `custom_insert.hlsl` | Code included through the insert lilblock (below): functions and includes that depend on Unity APIs or variables. Empty in the template. |
| `lilCustomShaderDatas.lilblock` | Two lines in the template: `ShaderName "TemplateFull"` and `EditorName "lilToon.TemplateFullInspector"`. Read by the importer from the container's own folder. Tag vocabulary: below. |
| `lilCustomShaderInsert.lilblock` | One line: `#include "custom_insert.hlsl"`. This is the file the containers' `lilSubShaderInsert` directive points at, so its content lands inside every pass's HLSLPROGRAM. |
| `lilCustomShaderProperties.lilblock` | The custom property block. The containers inline it into the generated Properties block after lilToon's `Default` and mode-specific property fragments. |
| `*.lilcontainer` | 68 container files: the variant set (next section). Each imports to one generated shader. |

### Editor/ — the files you edit

| File | Role |
| --- | --- |
| `CustomInspector.cs` | The derived inspector (class `TemplateFullInspector : lilToonInspector`). Holds the `shaderName` constant and the variant lookup table used for render-mode switching. See Custom Inspector. |
| `TemplateFull.asmdef` | Editor-only assembly definition, `name` `TemplateFull`, referencing `lilToon.Editor`. Rename both the `name` and the file. |

Nothing else in a custom shader project is yours: the SubShader bodies,
property fragments and pass implementations come at generation time
from `CustomShaderResources` — the lilToon package's bundled library
of ready-made SubShader bodies and property fragments, organised per
pipeline, which the containers compose by name.

## The container format

A `.lilcontainer` is a ShaderLab-shaped text file that lilToon's
container importer (`lilShaderContainerImporter`, a ScriptedImporter —
an importer Unity runs when the file is imported) expands at import
time into a real Shader asset. The generated text is also kept beside
it as a hidden "Shader Source" TextAsset, and the importer's custom
editor has an "Export Shader" button that writes the generated
`.shader` text to a file — the ground truth for verification (P-10).

The template's `lts.lilcontainer` in full:

```
Shader "*LIL_SHADER_NAME*/lilToon"
{
    Properties
    {
        lilProperties "Default"
        lilProperties "DefaultOpaque"
        lilProperties "lilCustomShaderProperties.lilblock"
    }

    lilPassShaderName "Hidden/*LIL_SHADER_NAME*/ltspass_opaque"
    lilSubShaderTags {"RenderType" = "Opaque" "Queue" = "Geometry"}
    lilSubShaderBRP "DefaultUsePass"
    lilSubShaderURP "DefaultUsePass"
    lilSubShaderHDRP "DefaultUsePass"

    CustomEditor "*LIL_EDITOR_NAME*"
}
```

Container directives the importer recognises:

| Directive | Effect |
| --- | --- |
| `lilProperties "<name>"` | Inlines a properties lilblock. A name containing `Default` resolves to the package's `CustomShaderResources/Properties/<name>.lilblock`; any other name resolves relative to the container's folder. |
| `lilSubShaderBRP` / `LWRP` / `URP` / `HDRP "<name>"` | Inlines the SubShader body for that pipeline only. `Default*` names resolve to `CustomShaderResources/<RP>/<name>.lilblock`; others are relative to the container folder. The `DefaultUsePass` body contains no pass code of its own: the generated SubShader borrows the passes of the Hidden `ltspass_*` pass shader by name (ShaderLab's UsePass mechanism). The `Default` body instead inlines the pass code into the SubShader. |
| `lilSubShaderTags { ... }` | The SubShader Tags block. |
| `lilPassShaderName "<name>"` | The Hidden pass shader this variant assembles (for example `Hidden/<name>/ltspass_opaque`). |
| `lilSubShaderInsert "<file>"` / `lilSubShaderInsertPost "<file>"` | File content substituted inside every pass's HLSLPROGRAM (the pre/post insertion points). This is how `lilCustomShaderInsert.lilblock` — and through it `custom_insert.hlsl` — reaches every pass. |
| `lilSkipSettings` | Omits, for this container, the block of defines generated from lilToon's project settings (the feature switches and quality options every lilToon shader normally receives). |

Placeholders the importer substitutes: `*LIL_SHADER_NAME*` and
`*LIL_EDITOR_NAME*` (from the datas lilblock), `*LIL_SUBSHADER_TAGS*`,
`*LIL_PASS_SHADER_NAME*`, `*LIL_INSERT_PASS_PRE*` / `*LIL_INSERT_PASS_POST*`,
`*LIL_INSERT_USEPASS_PRE*` / `*LIL_INSERT_USEPASS_POST*`,
`*LIL_SHADER_SETTING*` and `*LIL_SHADER_SETTING_MULTI*` (the define blocks
built from the lilToon shader settings), `*LIL_SRP_VERSION*`, and the
per-pipeline LightMode names.

The importer also appends a dummy pass (`ReferenceUVs.lilblock`,
LightMode "Never" — compiled but never rendered) to UsePass-style
SubShaders. A SubShader that borrows its passes contains no code of its
own reading the mesh's UV channels, so Unity would treat those channels
as unused and strip them, breaking the borrowed passes that do read
them; the dummy pass references the channels so they are kept.

## The stock keyword catalogues

The `Default*` names the two composition directives accept are a fixed
catalogue (official documentation, custom shader format). For
`lilSubShader*`: `Default`, `DefaultFakeShadow`, `DefaultFur`,
`DefaultGem`, `DefaultRefraction`, `DefaultRefractionBlur`,
`DefaultTessellation`, `DefaultLite`, `DefaultMulti`,
`DefaultMultiOutline`, `DefaultMultiFur`, `DefaultMultiRefraction`,
`DefaultMultiGem`, `DefaultUsePass`, `DefaultUsePassOutline`,
`DefaultUsePassFurOnly`, `DefaultUsePassOutlineOnly`,
`DefaultUsePassOverlay`. For `lilProperties`: `Default`, `DefaultLite`,
`DefaultOpaque`, `DefaultCutout`, `DefaultTransparent`,
`DefaultFurCutout`, `DefaultFurTransparent`, `DefaultRefraction`,
`DefaultGem`, `DefaultFakeShadow`. A name outside the catalogue is
treated as a file path relative to the container's folder — a
misspelled keyword therefore fails as a missing file, not as an
unknown keyword.

## The datas lilblock tags

`lilCustomShaderDatas.lilblock` sits beside the containers; the importer
reads the one in the folder of the container being unpacked. Tags are
line-based string directives:

| Tag | Effect |
| --- | --- |
| `ShaderName "<name>"` | Substituted for `*LIL_SHADER_NAME*`. Also the match key for the conditional forms below. |
| `EditorName "<class>"` | Substituted for `*LIL_EDITOR_NAME*` — the generated shader's `CustomEditor`. The stock value is `lilToon.lilToonInspector`. |
| `Replace "<from>" "<to>"` | Literal string replacement applied to the generated shader text. Conditional form `Replace "<name>" "<from>" "<to>"` applies only when the shader name contains `<name>` (a leading `!` inverts). |
| `InsertPassPre "<file>"` / `InsertPassPost "<file>"` | Inserts the file's content before/after all passes in the SubShader blocks. Same optional name-condition prefix; an RP-suffixed sibling file (for example `NameURP.lilblock`) takes precedence when it exists. When several lines match one shader, the one written last wins — declare the general file first and the name-conditioned overrides after it. |
| `InsertUsePassPre "<file>"` / `InsertUsePassPost "<file>"` | The same mechanism at the insertion points of a UsePass-style SubShader — before the first and after the last of the passes it borrows. |

Two shipped applications of these directives are worth knowing
as patterns:

- A geometry stage is attached with `Replace`, not with a
  template change. The pass shaders' build-option blocks
  (`// Build Option` followed by the `#pragma vertex` /
  `#pragma fragment` lines) are part of the generated text, so
  a `Replace` that rewrites that block can add
  `#pragma geometry <name>` and `#pragma require geometry`,
  plus a define the custom code tests. The replacement has to
  match the pass text exactly, so the datas first normalises
  line endings with a `Replace` pair (`\r\n` and `\r` to `\n`),
  and the name-conditional form separates the fur passes'
  blocks from the rest.
- A grab texture for the shader's own passes is added with
  `InsertPassPre`: the inserted lilblock holds
  `GrabPass { "_lilBackgroundTexture" }`. In stock lilToon
  only the Gem and Refraction passes carry a GrabPass, so the
  pattern pairs the general insertion with empty
  name-conditioned overrides for those two variants (the last
  matching line wins, per the table above).

## Generated shader names

With `ShaderName` = `TemplateFull`, the visible shaders are
`TemplateFull/lilToon`, `TemplateFull/lilToonLite`,
`TemplateFull/lilToonMulti`, and the `[Optional]` variants
(`TemplateFull/[Optional] OutlineOnly/Opaque`,
`TemplateFull/[Optional] FurOnly/...`, `TemplateFull/[Optional] FakeShadow`,
`TemplateFull/[Optional] Overlay`, ...). Mode variants the inspector
switches between are Hidden (`Hidden/TemplateFull/Cutout`,
`Hidden/TemplateFull/Tessellation/OpaqueOutline`, ...). The inspector's
`ReplaceToCustomShaders` table (Custom Inspector) enumerates them all;
lookups for variants you delete resolve to null and may be left in
place — see Custom Inspector for the maintenance rule.

A shader counts as a custom (optional) shader for the inspector when it is
bound through a derived inspector that sets `isCustomShader`; lilToon also
detects the `[Optional] ` name segment for its optional variants.

## The variant set, and trimming it

The 68 template containers form three families plus the pass shaders:

- **Full (`lts*`, 37 files)** — base, cutout, trans, onetrans, twotrans,
  each in plain, `_o` (outline) and some in `_oo` (one-pass outline)
  forms; fur, fur_cutout, fur_two, furonly set; gem; ref and ref_blur;
  fakeshadow; overlay and overlay_one; and the tessellation set
  (`lts_tess*`, 9 files) repeating the mode/outline matrix.
- **Lite (`ltsl*`, 13 files)** — the same mode matrix without
  tessellation, fur, gem or refraction.
- **Multi (`ltsmulti*`, 5 files)** — base, outline, fur, gem, refraction.
- **Pass shaders (`ltspass_*`, 9 files)** — the Hidden pass shaders the
  UsePass variants assemble: opaque/cutout/transparent, tessellation
  versions, and lite versions.

Trimming procedure (P-03): delete a variant's container to remove the
whole shader. Deleting containers does not touch the inspector's
lookup table — lookups for deleted variants are unreachable and may
stay (Custom Inspector holds the rule). Never delete an
`ltspass_*` container that a kept variant's `lilPassShaderName` still
references.

## Replacing a lilToon-internal include

Some lilToon internals have no hook — the tessellation stage is
one (`lil_tessellation.hlsl`). The shipped way to replace such a
file is include-guard preemption, in three parts:

1. A stub file the pass containers include before the original
   (the template's tessellation passes already include a cancel
   file at that point). The stub defines the original's include
   guard (`LIL_TESSELLATION_INCLUDED`) and nothing else, so the
   original later expands to nothing.
2. The replacement file: a copy of the original under a
   different guard (`LIL_TESSELLATION_INCLUDED1`), with the
   intended changes applied.
3. The pass containers include the stub, then the replacement,
   in place of the original.

Removing the stub undoes the replacement: the original is read
again — or, if the replacement is still included, the two
definitions collide and compilation fails. The replacement is a
copy of a lilToon internal, so the copy trap applies in full:
it diverges silently when lilToon changes the original, and
each lilToon release requires a re-diff against the new
original before the floor in Distribution can stay.
