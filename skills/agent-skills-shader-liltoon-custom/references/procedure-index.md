# Procedure Index (reverse lookup)

Procedures for building a lilToon-based custom shader, one entry per
task, written to be found by grep: search this file (or the whole skill)
for the task word or for the identifier you are about to write, and
read only that entry. Entries carry the steps and the traps; the facts
and reasoning stay single-sourced in the reference named under Depth, so
when a fact changes, it changes in one place and this index only changes
when a procedure itself changes.

Entry format: `## P-nn <task>` heading, a Keys line (grep vocabulary), a
Scope line (which variants or channels the procedure touches), a
Requires line (what must already hold before step 1), numbered steps,
and a Depth pointer. The numbering matches SKILL.md's Core Workflow.

## Requirements applying to every procedure

Keys: requirements, prerequisites, environment, versions

- Environment: Unity 2022.3, with lilToon (jp.lilxyzw.liltoon) in the
  target project. Identifiers are those of lilToon 2.4.* and the
  template shipped from the lilToon repository's Document branch. At
  any other version, re-verify before applying a procedure (P-01).
- The design decisions recorded in SKILL.md — the shipped variant set,
  where each effect lives, the property surface — are inputs to these
  procedures, not outputs of them. A Requires line that names one
  assumes it is already decided and written down.
- Each entry's Requires line lists only what that procedure adds beyond
  this block and beyond its own Scope field.

## P-01 Verify the target version's surface
Keys: version check, re-verify, lilToon version, macro spelling, hook exists, Template.zip, importer, Export Shader
Scope: the whole skill · Depth: SKILL.md Core Workflow P-01; Template and Generation
Requires: the lilToon version the shader targets chosen; the template for that version at hand.

1. Import the template into a scratch project with the target lilToon
   and confirm the containers import without errors and the generated
   shaders appear.
2. Grep the installed package's `Shader/Includes` for every macro the
   design uses — the `LIL_CUSTOM_*` set, the fragment hook points, the
   `LIL_REQUIRE_APP_*` defines — and record any that are absent or
   spelled differently from the references.
3. Check the inspector surface: `lilToonInspector` still exposes
   `LoadCustomProperties` and `DrawCustomProperties` as virtuals, and
   the template's derived class compiles against it.
4. Export one generated shader from the container importer and grep it
   for the `LIL_PASS_*` define spellings the design will branch on;
   record the spellings actually emitted (the DepthNormals and
   MotionVectors pair is the known divergence).
5. Write the verified version and spellings into the project's notes;
   later procedures trust that record, not the documentation.

## P-02 Set up from the template and rename
Keys: setup, rename, ShaderName, EditorName, shaderName, asmdef, TemplateFull, folder name, CustomEditor binding, render mode switch
Scope: template-wide · Depth: Template and Generation (The template; The datas lilblock tags); Custom Inspector (The derived class)
Requires: the product/shader name decided once — it becomes the shader name prefix, the inspector class name, and the assembly name.

1. Unpack Template.zip and rename the root folder to the product name.
2. In `Shaders/lilCustomShaderDatas.lilblock`, set `ShaderName` to the
   new name and `EditorName` to the derived inspector class's full name
   (for example `lilToon.<Name>Inspector`).
3. In `Editor/CustomInspector.cs`, rename the class to match
   `EditorName`, and set its `shaderName` constant to the same name as
   `ShaderName`.
4. Rename the asmdef's `name` and the asmdef file itself to the new
   name.
5. Reimport and confirm before going further: the shaders appear under
   the new name, a material on one opens the derived inspector (not the
   stock one), and switching rendering modes in the inspector does not
   lose the shader. A failure here is a missed rename from steps 2–4 —
   the binding fails silently, with no error to grep for (NEVER #2).

## P-03 Trim the variants
Keys: trim, delete variants, lilcontainer, variant set, ltspass, tessellation, lite, multi, fur, gem, ReplaceToCustomShaders, lookup table
Scope: the container set and the inspector · Depth: Template and Generation (The variant set, and trimming it); Custom Inspector (Render-mode switching)
Requires: the shipped variant set decided (SKILL.md, Design decisions); P-02 done, so the names in the inspector's lookup table are final.

1. List the template's 68 containers against the decided set: full
   family modes and outline forms, lite, multi, and the special
   families (fur, gem, refraction, fake shadow, overlay, tessellation).
2. Delete the containers outside the set. Deleting a container removes
   the whole generated shader; there is no partial variant.
3. Keep every `ltspass_*` container that a kept variant's
   `lilPassShaderName` references. Deleting one breaks generation of
   the kept variant, not of the deleted one.
4. Check the inspector's `ReplaceToCustomShaders` lookup table against
   the kept set: a lookup for every kept variant must exist. Lookups
   for deleted variants return null and are unreachable for materials
   on the kept families — delete them for tidiness or leave them;
   shipped packages do both. A missing lookup for a kept variant is
   the defect, not a leftover one.
5. Reimport and switch a test material through every remaining
   rendering mode to prove the kept set and the table agree.

## P-04 Add properties
Keys: add property, lilCustomShaderProperties, lilblock, attribute, lilVec3, prefix, name collision, Properties block
Scope: all variants (the properties lilblock is shared) · Depth: Properties
Requires: the property surface decided (SKILL.md, Design decisions): names, types, defaults; the reserved name prefix fixed.

1. Declare each property in `lilCustomShaderProperties.lilblock` in
   ShaderLab Properties syntax, under the reserved prefix (the template
   convention is `_Custom...`).
2. Attach a lil property attribute where the value's editing shape
   matches one (`[lilVec3]` for a three-component vector, `[lilToggle]`
   for a keyword-free toggle, and so on — the table is in Properties);
   standard Unity attributes otherwise.
3. Check each name against lilToon's own property blocks: the generated
   Properties block inlines the stock fragments first, so a name that
   collides with a stock property is declared twice in the generated
   shader.
4. Do not stop here: a property with no HLSL counterpart is a value
   nothing reads. P-05 is part of the same change (NEVER #3).

## P-05 Declare the HLSL counterparts
Keys: LIL_CUSTOM_PROPERTIES, LIL_CUSTOM_TEXTURES, uniform, TEXTURE2D, CBUFFER, sampler, declaration, counterpart
Scope: all passes (custom.hlsl is shared) · Depth: Properties (The HLSL counterparts; Samplers)
Requires: P-04's property list final — names and types are copied from it, not re-decided here.

1. Add each scalar/vector property's uniform to
   `LIL_CUSTOM_PROPERTIES` in `custom.hlsl`, with the HLSL type
   matching the property type (Vector → float4, float → float,
   int → int).
2. Add each texture property to `LIL_CUSTOM_TEXTURES` as
   `TEXTURE2D(<name>)`. If code needs the `_ST` or `_TexelSize`
   companions, declare them explicitly in `LIL_CUSTOM_PROPERTIES` —
   lilToon does not generate them for custom textures.
3. Never declare a sampler state (NEVER #8): sample with the shared
   aliases (`sampler_linear_repeat`, `sampler_linear_clamp`). Note
   that `sampler_linear_repeat` resolves to the trilinear-repeat
   sampler.
4. Reimport and export one generated shader; grep it for each property
   name and confirm it appears in both the Properties block and the
   uniform declarations.

## P-06 Add vertex behaviour
Keys: vertex, LIL_REQUIRE_APP, appdata, LIL_CUSTOM_V2F_MEMBER, LIL_V2F_FORCE, LIL_CUSTOM_VERT_COPY, LIL_CUSTOM_VERTEX_OS, LIL_CUSTOM_VERTEX_WS, interpolator, displacement, LIL_CUSTOM_SAFEVERT, geometry shader
Scope: all passes share the vertex body; interpolator indices are per-pass · Depth: Vertex Hooks
Requires: the effect's stage decided as vertex (SKILL.md, Design decisions); the properties it reads declared (P-04, P-05).

1. Request the vertex inputs the code reads with `LIL_REQUIRE_APP_*`.
   Only define what is not already unconditional (`positionOS`,
   `uv0`–`uv3` always exist; the POSITION and TEXCOORD0–3 defines are
   no-ops in 2.4.*). Do not combine PREVPOS/PREVVEL with uv4/uv5 —
   they occupy the same TEXCOORD channels.
2. For data the fragment stage needs, declare interpolators with
   `LIL_CUSTOM_V2F_MEMBER(id0,…,id7)`, using only the indices you need
   from the macro's parameters; or force a stock member present with
   `LIL_V2F_FORCE_<MEMBER>` instead of redeclaring it. The fur pass
   uses `LIL_CUSTOM_V2G_MEMBER` at its geometry stage.
3. Assign the custom members in `LIL_CUSTOM_VERT_COPY`. A member
   declared in step 2 and not assigned here is undefined in the
   fragment stage.
4. Modify the vertex in `LIL_CUSTOM_VERTEX_OS` (object space) or
   `LIL_CUSTOM_VERTEX_WS` (world space). Prefer object space for
   displacement: the same vertex body serves the forward, shadow and
   depth passes, so object-space displacement keeps them consistent
   for free. In the world-space hook, remember it runs a second time
   for the one-pass-outline position.
5. If a geometry stage will call the vertex code, define
   `LIL_CUSTOM_SAFEVERT`; otherwise leave it undefined.
6. Texture reads in the vertex stage need an explicit LOD
   (`LIL_SAMPLE_2D_LOD` at LOD 0 in the worked example).

## P-07 Add fragment behaviour
Keys: fragment, BEFORE, OVERRIDE, hook point, fd, lilFragData, stage order, UNPACK_V2F, replace stage
Scope: forward passes for the full hook set; non-forward passes expose UNPACK_V2F only · Depth: Fragment Hooks
Requires: the stages the effect touches chosen from the hook table, in invocation order; the properties it reads declared (P-04, P-05).

1. For each stage, choose the form: `BEFORE_<POINT>` to insert code
   before the stock stage, `OVERRIDE_<POINT>` to replace it. An
   OVERRIDE_ default body is the whole stage implementation — an
   override that still needs part of the stock behaviour must
   reproduce that part (NEVER #9).
2. Write against `fd` as it stands at that point in the invocation
   order: a hook sees only what the stages above it have produced
   (`fd.col` has no main colour before MAIN; `fd.lightColor` is final
   only after SHADOW; `fd.N` is the geometric normal until
   NORMAL_1ST).
3. Write the hook body as a dispatch call to a function in
   `custom_insert.hlsl` (Fragment Hooks, Shape of a hook body), passing
   a sampler with `LIL_SAMP_IN(...)` when the function needs one. Keep
   inline bodies for one-line adjustments only.
4. Gate every hook that touches a lilToon feature's stage or uniforms
   on that feature's `LIL_FEATURE_*` define, with `#undef OVERRIDE_<POINT>`
   in the `#else` branch (NEVER #10); gate stages the lite family
   lacks with `LIL_LITE`. An ungated reference to a disabled feature's
   uniform is a compile error in the projects that disable it.
5. Check the hook point exists in the passes you ship. Only
   `UNPACK_V2F` is invoked in the shadow caster, depth, meta, motion
   vectors and 2D passes; a `BEFORE_MAIN` written for those passes is
   silently never called.
6. Remember `UDIMDISCARD` has an OVERRIDE_ form only — there is no
   `BEFORE_UDIMDISCARD` to define.

## P-08 Add pass-specific code
Keys: custom_insert, lilSubShaderInsert, LIL_PASS, pass define, undef, Unity API, per-pass, spelling
Scope: the insert channel, all passes · Depth: Passes and Insert File
Requires: P-01's record of the emitted `LIL_PASS_*` spellings for the target version; the code's pass coverage decided (which passes it must run in, which it must stay out of).

1. Put functions and includes that depend on Unity APIs or variables
   in `custom_insert.hlsl`, not `custom.hlsl` (NEVER #4). It reaches
   every pass through `lilCustomShaderInsert.lilblock` and the
   containers' `lilSubShaderInsert` directive.
2. Branch on the pass defines (`LIL_PASS_FORWARD`, `FORWARDADD`,
   `SHADOWCASTER`, `DEPTHONLY`, `META`, and the DepthNormals /
   MotionVectors pair) using the spellings from P-01's record — not
   the documentation's, not this index's (NEVER #7).
3. Where a pass errors on a hook or helper, `#undef` it inside that
   pass's guard in `custom_insert.hlsl`, and redeclare an empty or
   pass-appropriate form when lilToon's own code in the same pass
   still references the name: the `#undef` lasts for the rest of that
   pass's compilation.
4. Reimport and read the errors per pass: a fix that clears the
   forward pass can move the failure to the shadow caster, which is a
   separate compilation.

## P-09 Extend the inspector
Keys: inspector, lilToonInspector, LoadCustomProperties, DrawCustomProperties, FindProperty, isCustomShader, isShowRenderMode, styles, Foldout, language file, GetLoc
Scope: the Editor assembly · Depth: Custom Inspector
Requires: P-02's rename complete (class name, EditorName, asmdef); the custom properties final (P-04), since the loader names them.

1. Declare one `MaterialProperty` field per custom property on the
   derived class.
2. In `LoadCustomProperties`, set `isCustomShader = true`, call
   `ReplaceToCustomShaders()`, set `isShowRenderMode` (the template
   derives it from the shader name not containing "Optional"), load a
   custom language file with `LoadCustomLanguage` if the labels need
   one, and load each field with `FindProperty("<name>", props)`.
3. In `DrawCustomProperties`, draw the section with the shared styles
   (`boxOuter`, `boxInnerHalf`/`boxInner`, `customToggleFont`) and the
   shared helpers (`Foldout`, `DrawLine`), rendering each property
   with `m_MaterialEditor.ShaderProperty` or a `lilEditorGUI` control,
   so the custom section reads as part of the stock inspector.
4. If the section needs copy / paste / reset, draw the section menu
   for it (Custom Inspector, Section menu): the stock menu covers
   only lilToon's registered properties, so the foldout provides its
   own over its property set, with the stock placement, labels and
   semantics.
5. Labels through `GetLoc("...")` when a language file is in play;
   literal labels otherwise — do not mix the two for one section.
6. Open a material in the editor and confirm the section renders and
   every control writes its property; a `FindProperty` name that
   matches nothing fails here, not at compile time.

## P-10 Verify
Keys: verify, reimport, export shader, compile, variant count, pipeline, unconfirmed, audit, handover
Scope: the whole shader · Depth: SKILL.md Core Workflow P-10; Template and Generation (The container format)
Requires: the shader complete through the procedures it uses; the NEVER list (SKILL.md). Unity-side import is the user's environment to confirm when the work happens off-machine; it is reported as unconfirmed and is never a gate the user must clear before the work proceeds.

1. Reimport the shader folder and confirm the generated shaders appear
   under the P-02 name, with the derived inspector bound.
2. Export the generated shader text from the container importer and
   grep it: each custom property appears in the Properties block and
   the uniforms (P-05), the hook code is present, and the `LIL_PASS_*`
   spellings match P-01's record.
3. Compile every kept variant on every pipeline the project targets.
   Read errors per pass and per pipeline — BRP, URP and HDRP
   SubShaders are separate compilations of the same sources.
4. Report size as variant counts per kept container, not source lines.
5. Switch a test material through every rendering mode the inspector
   offers (P-03's agreement check) and confirm the custom section of
   the inspector drives the properties (P-09).
6. Hand over with what was verified and what was not: pipelines not
   present in the test project, and any hook or define whose spelling
   could not be confirmed against a generated shader, stay listed as
   unconfirmed.

## P-11 Distribute the shader
Keys: distribute, VPM, package.json, vpmDependencies, license, MIT, LICENSE-lilToon, legacyFolders, release
Scope: the finished shader · Depth: Distribution
Requires: P-10 passed; the lilToon version floor from P-01's record; the product's own license chosen.

1. Lay the package out per Distribution: package.json at the root, the
   trimmed Shader/ set, the Editor/ set, Examples/ if any.
2. Declare lilToon in package.json's `vpmDependencies` as a range
   whose floor is the oldest version P-01 verified the used surface
   against. If an Assets-based predecessor exists, declare its folder
   in `legacyFolders`.
3. Include lilToon's MIT license beside the product's own license: the
   containers, lilblocks and template-derived files are lilToon's
   code, and the license's permission is conditional on the notice
   travelling with them.
4. If any fragment stage is an OVERRIDE-by-copy (Fragment Hooks, The
   copy trap), re-diff each copied stage against the lilToon version
   the floor now names before releasing.
5. Install-test into a clean project carrying only the declared
   lilToon range: generation, inspector binding, and a render-mode
   switch (P-10's checks from the consumer's position).

## P-12 Migrate materials from stock lilToon
Keys: migrate, migration, stock lilToon, reassign, material, keyword, render queue, variant correspondence, converter
Scope: existing materials on stock lilToon moving to the finished custom shader · Depth: Migration
Requires: the custom shader built from the installed lilToon version's template (P-01) with its full variant set, and P-10 passed.

1. Decide the scope: which materials move, and the variant kind each
   one sits on. Materials that do not need the custom behaviour stay
   on stock lilToon.
2. Confirm the custom set still contains every variant kind in the
   scope. If P-03 already trimmed one away, restore that container
   before moving any material.
3. Reassign each material to the corresponding custom variant (Full
   to Full, Lite to Lite, Multi to Multi). The material is a shared
   asset; one reassignment covers every user of it.
4. Verify the carry-over on each kind: same-name property values are
   in place, the inspector shows the custom sections and drives them
   (P-09), and the keyword state matches what the carried properties
   derive — for Multi materials, the borrowed keywords are present.
5. Only after every scoped material is moved and verified, trim the
   variants nothing uses (P-03).
