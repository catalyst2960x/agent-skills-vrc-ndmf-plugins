---
name: agent-skills-shader-liltoon-custom
description: "Creating custom shaders based on lilToon with its official template system: .lilcontainer generation, .lilblock properties and shader data, HLSL insertion macros for uniforms, vertex inputs and interpolators, vertex and fragment hooks (BEFORE_/OVERRIDE_), pass-specific code, variant trimming, a custom inspector derived from lilToonInspector, and VPM distribution. Covers lilToon 2.4.* on Unity 2022 LTS."
---

# lilToon Custom Shaders

Build a custom shader on lilToon (jp.lilxyzw.liltoon, Unity 2022.3) without
forking lilToon: start from the official template, add properties, and insert
HLSL at the hook points lilToon exposes. The generated shader is a full
lilToon variant set (normal, lite, multi, fur, gem, refraction, tessellation)
whose SubShaders are assembled by lilToon's container importer from
`.lilcontainer` files, `.lilblock` fragments, and CustomShaderResources
— lilToon's bundled library of ready-made SubShader bodies and
property fragments, organised per pipeline, that containers compose by
name instead of containing the code themselves.

Version anchor: lilToon 2.4.* and the template shipped from the lilToon
repository's Document branch (Template.zip). Macro spellings, hook points and
inspector APIs move between lilToon versions, so Step 1 of the workflow
re-verifies them against the installed version before use.

Load this skill before writing or modifying a lilToon-based custom shader.
Do not use it for shaders written from scratch, for other shader frameworks,
or for editing lilToon's own shipped shaders.

## Table of Contents

| Section | Content | Reference |
| --- | --- | --- |
| [Common Mistakes](#common-mistakes-never-list) | Failure modes to design out | This file |
| [Core Workflow](#core-workflow) | The procedures, in order | This file |
| [Procedure Index](references/procedure-index.md) | Every task as a greppable procedure: steps and traps, facts live in the other references | `references/procedure-index.md` |
| [Template and Generation](references/template-and-generation.md) | The template files, the container format, the datas tags, how generation works, variant trimming | `references/template-and-generation.md` |
| [Properties](references/properties.md) | Declaring properties in the lilblock, lil property attributes, the HLSL counterparts, samplers | `references/properties.md` |
| [Vertex Hooks](references/vertex-hooks.md) | Vertex inputs, interpolators, object/world-space insertion points | `references/vertex-hooks.md` |
| [Fragment Hooks](references/fragment-hooks.md) | BEFORE_/OVERRIDE_ hooks, the 35 hook points, the lilFragData fields | `references/fragment-hooks.md` |
| [Passes and Insert File](references/passes-and-insert.md) | Per-pass defines, the insert lilblock, pass-specific code and `#undef` | `references/passes-and-insert.md` |
| [Custom Inspector](references/custom-inspector.md) | Deriving from lilToonInspector, binding, styles and helpers, language files | `references/custom-inspector.md` |
| [lilToon Specification](references/liltoon-specification.md) | Onboarding: what lilToon is — generation, the variant family, features, the fragment pipeline, passes, the inspector, pipelines. Read first if lilToon is new to you | `references/liltoon-specification.md` |
| [lilToon Requirements](references/liltoon-requirements.md) | Onboarding: every constraint a custom shader must satisfy, traced to the lilToon behaviour that imposes it (R-01–R-12) | `references/liltoon-requirements.md` |
| [lilToon Macros](references/liltoon-macros.md) | Onboarding: one index of the macro surface — insertion macros, vertex input defines, vertex body macros, the 35 fragment hooks, pass defines, sampling macros, constants | `references/liltoon-macros.md` |
| [lilToon Surface](references/liltoon-surface.md) | lilToon's own uniforms, samplers, sampling macros and helper functions that custom code can build on, and the feature defines that gate them | `references/liltoon-surface.md` |
| [lilToon Platform Support](references/liltoon-platform-support.md) | What lilToon cannot do on Quest, Android and mobile — the VRChat avatar policy, the geometry/tessellation stage ceiling, GrabPass and depth-texture inputs, SDF/MSDF degradation — and what a solution would take in each case | `references/liltoon-platform-support.md` |
| [VRChat Shader Globals](references/vrchat-shader-globals.md) | The data the VRChat client publishes to shaders: the camera/mirror and time globals, and the Material Contacts buffers (VRCContacts.cginc) — declaration rules, layouts, header macros | `references/vrchat-shader-globals.md` |
| [Migration](references/migration.md) | Moving materials from stock lilToon to a custom shader: what carries across a reassignment (properties, keywords, render queue), the requirements, and what a cross-package converter takes | `references/migration.md` |
| [Distribution](references/distribution.md) | Shipping as a VPM package: layout, the lilToon dependency range, the license duty for template-derived files | `references/distribution.md` |
| [Worked Example](references/worked-example.md) | The vertex-wave example from the official documentation, end to end | `references/worked-example.md` |
| [Cheatsheet](CHEATSHEET.md) | Every macro and hook on one page | `CHEATSHEET.md` |

## Common Mistakes (NEVER list)

1. **Never edit a generated `.shader`.** The container importer regenerates
   the shader text from the `.lilcontainer`, the `.lilblock` files and the
   package resources on import — and lilToon regenerates its own built-in
   shaders from the same kind of sources whenever its settings are
   applied. Every change belongs in the
   template files: the containers, the lilblocks, `custom.hlsl`,
   `custom_insert.hlsl`, or the inspector.
2. **Never rename the shader in fewer places than it is bound.** The name
   lives in the datas lilblock (`ShaderName`, `EditorName`), the inspector
   class name, the inspector's `shaderName` constant (used to find every
   variant for render-mode switching), and the asmdef `name` and file name.
   A mismatch does not error; the CustomEditor binding or the render-mode
   switch silently stops working. Procedure P-02 lists every edit.
3. **Never declare a property on only one side.** A property in
   `lilCustomShaderProperties.lilblock` without its uniform in
   `LIL_CUSTOM_PROPERTIES` (or its texture in `LIL_CUSTOM_TEXTURES`) is a
   material value nothing reads; the reverse is a uniform with no way to set
   it. Add both in the same change (P-04, P-05).
4. **Never put Unity-API-dependent code in `custom.hlsl`.** It is included
   into every pass, including passes where the API or variable does not
   exist. Code that depends on Unity functions or variables belongs in
   `custom_insert.hlsl`, guarded per pass, with `#undef` and redeclaration
   where a pass conflicts (P-08).
5. **Never rely on `LIL_REQUIRE_APP_POSITION` or
   `LIL_REQUIRE_APP_TEXCOORD0`–`TEXCOORD3`.** In 2.4.* those inputs are
   unconditional and the defines sit in a comment block: they do nothing.
   The defines that add inputs are TEXCOORD4–7, COLOR, NORMAL, TANGENT and
   VERTEXID (plus PREVPOS/PREVVEL for motion vectors).
6. **Never ship the template's full container set by default.** The template
   carries 68 `.lilcontainer` files; each one generates a complete shader
   with its own variant space. Delete the variants you do not ship, and trim
   the inspector's variant lookup list to match (P-03).
7. **Never copy a pass define's spelling from documentation without
   checking the generated shader.** The documentation writes
   `LIL_PASS_DEPTHNORMALS` and `LIL_PASS_MOTIONVECTORS`; the 2.4.* source
   uses the singular forms in its pass markers and include guards
   (`LIL_PASS_DEPTHNORMAL`, `LIL_PASS_MOTIONVECTOR`). Export the generated
   shader from the container importer and grep it for the exact define
   before writing pass-conditional code (P-08, P-10).
8. **Never declare a sampler state in `custom.hlsl`.** Declare textures
   with `TEXTURE2D(...)` in `LIL_CUSTOM_TEXTURES` and sample with
   lilToon's shared samplers (`sampler_linear_repeat`,
   `sampler_linear_clamp`, and their aliases) — that is the default, so
   every lilToon-based shader samples through the same two states. The one working exception is
   the insert file: a dedicated `SamplerState` declared in
   `custom_insert.hlsl` for a specific filter/addressing need (a shipped
   custom shader samples its vertex-stage mask through its own
   point-clamp state this way). Treat it as a deliberate exception with
   a stated reason, not as the default.
9. **Never override an `OVERRIDE_` hook without reproducing the stock
   behaviour you still need.** The default define of an `OVERRIDE_` hook
   contains the whole stage implementation (for example `OVERRIDE_MAIN`
   samples the main texture, applies tone correction and multiplies
   `_Color`). Defining it replaces the stage entirely; `BEFORE_` only
   inserts before it.
10. **Never replace or wrap a lilToon feature's stage without gating on
    its feature define.** lilToon's features are switched for the whole
    project in its settings, and a feature's uniforms and code are
    compiled only when the settings enable it (signalled by a
    `LIL_FEATURE_*` define). Custom code that references them breaks
    compilation in projects where the feature is off. Gate the
    replacement on the define and `#undef` your `OVERRIDE_` in the
    `#else` branch, and gate stages the lite family lacks with
    `LIL_LITE` the same way.

## Core Workflow

- **P-01 — Verify the target version's surface.** Against the installed
  lilToon, confirm: the template files exist and import, the hook macros
  named in the references are present in `Shader/Includes`, and the
  inspector virtuals match. Record the lilToon version the shader targets.
- **P-02 — Set up from the template and rename.** Unpack Template.zip,
  rename the root folder, then make every rename edit: folder name,
  `ShaderName` and `EditorName` in `lilCustomShaderDatas.lilblock`, the
  inspector class name, its `shaderName` constant, and the asmdef `name`
  plus file name. Details: Template and Generation.
- **P-03 — Trim the variants.** Decide the shipped variant set (base,
  rendering modes, outline forms, lite/multi, fur/gem/refraction,
  tessellation) and delete the other `.lilcontainer` files. Keep the
  `ltspass_*` containers your kept variants reference. The inspector's
  `ReplaceToCustomShaders` lookups for kept variants must exist;
  lookups for deleted variants resolve to null, are unreachable for
  materials on the kept families, and may stay or be trimmed.
- **P-04 — Add properties.** Declare each custom property in
  `lilCustomShaderProperties.lilblock`, in ShaderLab Properties syntax,
  with a lil property attribute where one fits. Keep names in a reserved
  prefix (the template convention is `_Custom...`) so they cannot collide
  with lilToon's own properties. Details: Properties.
- **P-05 — Declare the HLSL counterparts.** Add the uniforms to
  `LIL_CUSTOM_PROPERTIES` and the textures to `LIL_CUSTOM_TEXTURES` in
  `custom.hlsl`, with types matching the property declarations.
- **P-06 — Add vertex behaviour.** Request extra vertex inputs with
  `LIL_REQUIRE_APP_*`, add interpolators with `LIL_CUSTOM_V2F_MEMBER` (or
  force stock members with `LIL_V2F_FORCE_*`), fill them in
  `LIL_CUSTOM_VERT_COPY` when they are not stock copies, and modify the
  vertex in `LIL_CUSTOM_VERTEX_OS` (object space) or
  `LIL_CUSTOM_VERTEX_WS` (world space). Details: Vertex Hooks.
- **P-07 — Add fragment behaviour.** For each stage you touch, choose
  `BEFORE_<POINT>` to insert or `OVERRIDE_<POINT>` to replace, and write
  against the `fd` fields that stage has already produced. Check the hook
  point exists in the passes you ship: non-forward passes expose only
  `UNPACK_V2F`. Details: Fragment Hooks.
- **P-08 — Add pass-specific code.** Put Unity-dependent functions and
  includes in `custom_insert.hlsl` (reached through
  `lilCustomShaderInsert.lilblock`), branch on the `LIL_PASS_*` defines,
  and `#undef` a hook in a pass where it errors, redeclaring what that
  pass needs. Verify each define's spelling in the exported shader text.
- **P-09 — Extend the inspector.** In the derived inspector, set
  `isCustomShader = true`, load the custom properties in
  `LoadCustomProperties` with `FindProperty`, draw them in
  `DrawCustomProperties` with the shared styles and helpers, and add a
  language file if the labels need localisation. Details: Custom
  Inspector.
- **P-10 — Verify.** Reimport and confirm: the generated shaders appear
  under the new name, the CustomEditor binding opens the custom inspector,
  every kept variant compiles on every pipeline the project targets, and
  the exported shader text contains the custom properties, the hook code
  and the expected `LIL_PASS_*` spellings. Report variant counts per kept
  container, not source lines. State plainly what was not verified (for
  example pipelines not present in the test project).
- **P-11 — Distribute.** Ship as a VPM package whose `package.json`
  declares the lilToon dependency as a `vpmDependencies` range, with the
  shader and editor folders at the package root, and include lilToon's
  MIT license in the distribution: the shipped containers, lilblocks and
  template-derived files are lilToon's code. Details: Distribution.
- **P-12 — Migrate materials from stock lilToon.** Generate the full
  variant set first, verify it (P-10), then reassign each material to
  the corresponding custom variant: property values carry by name, the
  keyword set and render queue are the material's own, and lilToon's
  keywords regenerate from the carried properties. Trim variants only
  after the move. Details: Migration.

## Design decisions to settle before starting

- **Variant set.** Which of the template's families ship (full, lite,
  multi) and which rendering modes and outline forms. This is the largest
  cost decision: it multiplies containers, compile time and variant counts.
- **Where each effect lives.** Vertex stage (cheap, per-vertex resolution),
  fragment hook (per-pixel, ordered against lilToon's own stages), or
  inspector-only (material setup, no shader cost). The hook order in
  Fragment Hooks fixes what a fragment change can see and what it can no
  longer change.
- **Platform target.** lilToon's ceiling is inherited: PC-first, fur
  and tessellation above the mobile stage ceiling, no mobile variant
  family. Decide at P-01 whether the shader targets PC only, Android
  worlds/standalone (with those families and the GrabPass/depth inputs
  excluded), or claims nothing about mobile — and record it where the
  distribution states support (lilToon Platform Support).
- **Property surface.** Which values are per-material properties (artist
  control, variant-free) and which are compile-time switches. lilToon
  features are switched project-wide in lilToon's settings and reach
  the shader as defines, not as material keywords; a custom shader follows the same split — do not invent
  per-material keywords for feature switches.
