# lilToon specification (what a custom shader is built on)

The parts of lilToon itself that a custom shader depends on, stated as
lilToon's specification. This file is the onboarding layer: it assumes
no prior knowledge of lilToon. The procedures in this skill cite it;
the detail references assume it. Everything below is lilToon 2.4.*.

## What lilToon is

lilToon (`jp.lilxyzw.liltoon`) is a toon shader package for Unity
2022.3, used mainly for VRChat avatars. A "lilToon shader" is not one
shader but a generated family of shader variants sharing one material
property set and one inspector. lilToon ships:

- A library of ready-made SubShader bodies and property fragments,
  `CustomShaderResources`, organised per render pipeline (BRP, URP,
  HDRP). Containers compose these by name.
- A set of pass shaders, `ltspass_*`, holding the actual pass
  implementations (forward, shadow caster, depth, outline, ...). The
  default SubShader bodies borrow their passes by name (ShaderLab
  UsePass) instead of containing the pass code.
- A container importer that turns `.lilcontainer` text files into
  Shader assets.
- A project settings asset and a settings application step: feature
  switches and quality options are project-wide, and applying the
  settings regenerates lilToon's shaders from their sources with the
  chosen defines.
- A custom inspector base class, `lilToonInspector`, that draws the
  material UI and switches a material between rendering modes by
  reassigning its shader to the mode's variant.
- Language files (TSV) that localise the inspector, including property
  display names.

## Generation

A shader exists as a `.lilcontainer`: a ShaderLab-shaped text file
listing which property fragment, which SubShader bodies, and which
insertions compose it. On import, the container importer expands it
into a real Shader asset and keeps the generated text beside it as a
hidden "Shader Source" TextAsset. Consequences that shape everything
in this skill:

- The generated `.shader` is an output. Editing it is overwritten on
  the next import.
- Two containers that differ only in a tag produce different variant
  sets from the same shared fragments — this is how the variant family
  below is produced from one template.
- A UsePass-style SubShader contains no code reading the mesh's UV
  channels, so the importer appends a never-rendered dummy pass that
  references them; otherwise Unity would strip the channels the
  borrowed passes need.

## Package layout (the parts a custom shader touches)

- `CustomShaderResources/` — the composition library for custom
  shaders: `BRP/`, `URP/`, `HDRP/` hold the SubShader bodies,
  `Properties/` the property fragments. (lilToon's own built-in
  shaders compose from the separate `BaseShaderResources/`.)
- `Shader/Includes/` — the HLSL the passes share. The files custom
  code interacts with: `lil_common_input.hlsl` (material uniform
  declarations — the block the insertion macros extend),
  `lil_common_macro.hlsl` (pipeline-difference absorption),
  `lil_common_functions.hlsl` (the helper functions),
  `lil_common_vert.hlsl` / `lil_common_frag.hlsl` (the vertex and
  fragment bodies the hooks hang off), `lil_pass_<name>.hlsl` (one per
  pass kind).
- `Shader/ltspass_*.shader` — the pass shaders the variants borrow.
  The suffix names the form: `_o` outline, `_cutout`, `_trans`
  transparent, `_fur`, `_ref` refraction, `_gem`, `_tess`
  tessellation, `_overlay` (forward-only transparent), `_fakeshadow`,
  `_one` / `_two` (ForwardAdd folded into the vertex stage, without /
  with a back-face pass).
- `Editor/` — `lilStartup.cs` generates the shader settings at
  startup; `lilToonSetting.cs` is the settings ScriptableObject;
  `Editor/Resources/lang.txt` is lilToon's own language file, in the
  same TSV format a custom shader's language file uses.

## The variant family

Each shader name (for example `lilToon`) heads a family. The template
ships 68 containers: the full family (the rendering modes in plain,
outline and one-pass-outline forms, plus fur, gem, refraction, fake
shadow, overlay and tessellation sets), the lite family (the mode
matrix with a reduced feature set, `LIL_LITE` defined), the multi
family (SubShaders serving several pipelines), and the pass shaders
the UsePass variants assemble. The file-level breakdown — counts,
prefixes, and which set a container belongs to — is in Template and
Generation (The variant set, and trimming it); that is the list P-03
trims against.

Generated shaders are named `Hidden/` + the container's `ShaderName` +
the variant suffix (`Hidden/lilToonCutout`,
`Hidden/lilToonTessellation/OpaqueOutline`, ...); the two user-facing
entry shaders (normal and optional) are not Hidden. Rendering modes in
the inspector are these variants: switching a material's mode means
assigning a different family member as its shader.

The UsePass composition has exceptions: a few built-in shaders carry
their own passes instead of borrowing ltspass ones — fake shadow, fur
(both forms), gem, refraction (both forms), and the multi family
(`ltsmulti`, `ltsmulti_fur`, `ltsmulti_gem`, `ltsmulti_ref`). A custom
shader modelled on one of those families inherits the exception: its
passes live in its own containers, not in a shared pass shader.

The multi family differs in one more way: where the normal family
receives lilToon's settings as defines, multi variants switch features
with shader keywords. The keywords are deliberately the same names
Unity's built-in shaders use, which keeps the project's keyword budget
from exhausting and avoids SDK keyword warnings — a custom shader
does not get to choose this axis; it follows the family it extends.

## Features

A feature (alpha mask, emission, shadow, rim, ...) is a unit of
lilToon with three parts: a set of material properties, a stage in the
fragment pipeline, and a project-wide switch. The switch is the part
custom code most often trips on:

- Features are enabled or disabled for the whole project in lilToon's
  settings, not per material.
- An enabled feature is signalled to the shader by a define,
  `LIL_FEATURE_<FEATURE>`. The feature's uniforms, textures and stage
  code are compiled only under that define.
- A custom shader therefore cannot assume a feature's state exists;
  it must test the define (lilToon Requirements, R-07).

The lite family is the other compile-time split: lite variants define
`LIL_LITE` and omit several stages entirely.

## The fragment pipeline and `fd`

In the forward pass, lilToon evaluates the surface in a fixed sequence
of stages (main colour, lighting, shadow, emission layers, reflections,
rim, final composition, ...). All stages read and write one struct,
`fd` (`lilFragData`): the surface colour, normals, view and light
vectors, per-feature results, and the values later stages consume. The
custom-shader hook points are positions in this sequence (Fragment
Hooks lists all 35). Two properties of the design matter for custom
code:

- Each stage has a default implementation that lives in the hook's
  default define. Inserting *before* a stage keeps the implementation;
  *overriding* a hook replaces it.
- `fd` is fully initialised before the first hook, so every hook can
  rely on the fields the earlier stages own.

## Passes

A variant's passes are the forward pass plus, per pipeline and mode:
shadow caster, depth-only, depth-normals, motion vectors, meta,
outline, fur shell passes, and a 2D pass. Each pass compiles with a
pass-kind define (`LIL_PASS_FORWARD`, `LIL_PASS_SHADOWCASTER`, ...)
that lilToon writes into the generated text during generation — the
defines exist in the generated shader, not in any editable file. Only
the forward pass runs the full fragment pipeline; the other passes run
a reduced one (of the fragment hooks, only `UNPACK_V2F` is invoked
there).

## Inside a pass

Each pass's HLSL file has the same structure, which fixes where custom
code lands: `lil_pass_<name>.hlsl` includes `lil_common.hlsl` (which
brings in the settings defines, the pipeline macros, the material
uniform block — the insertion macros' home — and the helper
functions), then the pass's `appdata` and `v2f` declarations, then
`lil_common_vert.hlsl` and `lil_common_frag.hlsl`, then the pass's own
pixel shader. Two consequences: everything a hook can see is decided
by this include order, and the per-variant differences are declared as
macros in the shader's HLSLINCLUDE before the pass files branch on
them — the passes are one shared codebase, not per-variant copies.

## The inspector

`lilToonInspector` draws the material editor: property groups with
foldouts and headers, the render-mode switch, preset and language
support. Its mechanics that a custom inspector inherits:

- One Shader field per family variant holds the switch targets;
  `ReplaceToCustomShaders()` repoints them.
- Property display names and enum labels are localisation keys,
  resolved through the loaded language files (`GetLoc`); a display
  string that matches no key is shown as written, and a custom shader
  that ships keys must load its own language file for them to resolve.
- Drawing helpers (headers, foldouts, sliders with text fields, RGBA
  colour fields) are shared protected methods, so a derived inspector
  renders in the same style.

## Pipelines

The same container set generates BRP, URP and HDRP SubShaders. lilToon
absorbs the pipeline differences itself: separate SubShader bodies per
pipeline, and macros for the constructs whose syntax differs (sampler
passing is the one custom code meets most often — `LIL_SAMP_IN`).
A custom shader written against the template inherits this split; it
does not write per-pipeline code unless it deliberately adds a
pipeline-specific insertion.
