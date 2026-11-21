# agent-skills-shader-liltoon-custom

A skill for creating custom shaders based on lilToon with its official
template system.

## Sources

Procedure P-10 is the compile check to run
on a shader this skill produces.

| Source | What was taken from it | License |
| --- | --- | --- |
| lilToon 2.4.* (package source and Template.zip) | The template system, macro and hook surface, generation rules | MIT |
| lilToon official documentation (custom shader pages) | The reference procedure and stated behaviour; three points where it differs from the 2.4.* source are recorded in the skill | — |
| Two shipped custom shaders built on the template | Evidence for working patterns (hook dispatch shape, feature gating, localisation, VPM packaging); no code taken | Their own |

## What it covers

- A procedure index (`references/procedure-index.md`): every task as a
  greppable entry with steps and traps, facts single-sourced in the
  other references
- Setup from Template.zip and the full rename set (datas lilblock,
  inspector class and its `shaderName` constant, asmdef)
- The `.lilcontainer` generation model and the datas tag vocabulary
  (`ShaderName`, `EditorName`, `Replace`, `InsertPassPre/Post`,
  `InsertUsePassPre/Post`)
- Trimming the template's 68-container variant set
- Properties in `lilCustomShaderProperties.lilblock`, the lil property
  attributes, and the `LIL_CUSTOM_PROPERTIES` / `LIL_CUSTOM_TEXTURES`
  HLSL counterparts
- Vertex hooks: `LIL_REQUIRE_APP_*` input requests,
  `LIL_CUSTOM_V2F_MEMBER` / `LIL_V2F_FORCE_*` interpolators,
  `LIL_CUSTOM_VERT_COPY`, `LIL_CUSTOM_VERTEX_OS` / `LIL_CUSTOM_VERTEX_WS`,
  `LIL_CUSTOM_SAFEVERT`
- Fragment hooks: `BEFORE_` / `OVERRIDE_` at 35 hook points, and the
  `lilFragData` (`fd`) fields they operate on
- Pass-specific code in `custom_insert.hlsl`, the `LIL_PASS_*` defines,
  and the `#undef` pattern
- A custom inspector derived from `lilToonInspector`
  (`LoadCustomProperties` / `DrawCustomProperties`,
  `ReplaceToCustomShaders`, shared styles and helpers)
- lilToon's own surface available to custom code (stock uniforms,
  per-texture samplers, sampling macros, helper functions) and the
  `LIL_FEATURE_*` gating that conditions it
- Distribution as a VPM package, including the lilToon dependency
  range and the MIT license duty for template-derived files

For readers new to lilToon, three onboarding references carry the
`liltoon-` prefix: `liltoon-specification.md` (what lilToon is and how
its pieces fit), `liltoon-requirements.md` (the constraints lilToon's
behaviour imposes on a custom shader, R-01–R-11), and
`liltoon-macros.md` (one index of the macro surface). The
task references above cite them instead of assuming the reader already
knows lilToon's side of the system.

Points taken from shipped custom shaders (a vertex-intrusion shader
and an MSDF mask extension) are folded in: hooks as dispatch calls to
insert-file functions, feature-define gating with `#undef` in the
else branch, localisation keys as property display names with a TSV
language file, and the maintenance cost of OVERRIDE-by-copy stages.
Two statements from the first version were corrected against the same
evidence: a dedicated `SamplerState` in the insert file is a working
pattern, and inspector lookups for deleted variants may stay.

## Basis

The procedure follows lilToon's official custom-shader documentation.
Where the documentation and the source differ (five undocumented hook points, the no-op
`LIL_REQUIRE_APP_*` defines, the DepthNormals / MotionVectors pass-define
spellings), the skill states both and tells the reader to check the
generated shader.

Target: lilToon 2.4.*, Unity 2022 LTS (the package declares Unity
2022.3).
