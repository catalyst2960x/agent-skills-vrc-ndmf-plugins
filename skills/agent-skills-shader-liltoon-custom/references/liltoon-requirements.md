# Requirements that come from lilToon

Every constraint a lilToon custom shader must satisfy, traced to the
lilToon behaviour that imposes it. The SKILL.md NEVER rules are the
short form of this list; this file is the reasoning layer for readers
who do not know lilToon yet. "Source" names the lilToon mechanism
(lilToon Specification describes each).

## R-01 — Change only the template-side files

**Requirement.** All changes live in the containers, the three
lilblocks, `custom.hlsl`, `custom_insert.hlsl`, and the inspector.
Generated `.shader` files are never edited.
**Source.** The container importer regenerates the shader text from
those sources on every import, and applying lilToon's settings
regenerates lilToon's own shaders the same way.
**If violated.** The edit disappears at the next import or settings
application, silently.

## R-02 — Properties exist only through the properties lilblock

**Requirement.** A custom property is declared in
`lilCustomShaderProperties.lilblock` and, when the shader reads it,
given an HLSL counterpart through `LIL_CUSTOM_PROPERTIES` /
`LIL_CUSTOM_TEXTURES`.
**Source.** The generated shader's property block is assembled from
the lilblocks, and its uniform block from the insertion macros. There
is no other channel into either.
**If violated.** A property typed into a generated file vanishes
(R-01); a property without its HLSL counterpart is invisible to the
shader code; a texture without its `_ST` / `_TexelSize` / `_HDR`
companions fails at the first macro that expects them.

## R-03 — Names are a five-place agreement

**Requirement.** Renaming the shader means changing all five places
together: the folder, the datas lilblock's `ShaderName` and
`EditorName`, the inspector class name and its `shaderName` constant,
the asmdef name and file name.
**Source.** The generated shader names come from `ShaderName`; the
inspector binds through the generated shaders' `CustomEditor` line,
which names the class; `ReplaceToCustomShaders` finds variants by
`shaderName` + suffix; Unity resolves the editor assembly by asmdef.
**If violated.** The failures are split across systems — shaders under
one name, an inspector bound to another, mode switches finding
nothing — and none of them is a compile error in the file that is
wrong.

## R-04 — The inspector binding is part of the shader

**Requirement.** The datas lilblock keeps
`EditorName "lilToonInspector"` replaced by the custom inspector's
full class name, and the derived class sets `isCustomShader = true`.
**Source.** The generated shader's `CustomEditor` line is written from
`EditorName`; the base class changes its drawing and mode-switch
behaviour on `isCustomShader`.
**If violated.** Materials open with the stock inspector (custom
properties undrawn) or with lilToon's own variant set as switch
targets.

## R-05 — Sample through lilToon's samplers

**Requirement.** Textures are sampled with lilToon's shared sampler
states (`sampler_linear_repeat`, `sampler_linear_clamp`, and the
per-texture aliases such as `sampler_MainTex`). A dedicated
`SamplerState` is declared only in `custom_insert.hlsl`, for a stated
need the shared states do not cover.
**Source.** lilToon declares exactly two sampler states and aliases
them per texture, so every lilToon-based shader samples through the
same states. The working exception in shipped shaders places the
dedicated state in the insert file, not in the macro channel.
**If violated.** An unneeded private state compiles but breaks the
uniformity the aliases exist to keep, and adds a sampler slot every
variant pays for.

## R-06 — An OVERRIDE_ hook owns its stage

**Requirement.** Defining `OVERRIDE_<POINT>` means reproducing
whatever part of the stock stage the shader still needs. Prefer
`BEFORE_<POINT>` for adjustments.
**Source.** The hook's default define *is* the stage implementation;
the fragment pipeline calls the define, not a function with a fallback.
**If violated.** The stage's behaviour (for `OVERRIDE_MAIN`: the main
texture sample, tone correction, the `_Color` multiply) is silently
absent from the output.

## R-07 — Feature-dependent code is gated on LIL_FEATURE_*

**Requirement.** Code that references a feature's uniforms, textures
or stage is wrapped in `#if defined(LIL_FEATURE_<FEATURE>)`, with
`#undef OVERRIDE_<POINT>` in the `#else` branch; stages the lite
family lacks are additionally gated on `LIL_LITE`.
**Source.** Features are project-wide settings; with a feature off,
its uniforms and code are not compiled at all, and the define is the
only signal.
**If violated.** The shader compiles in the author's project and fails
in any project whose settings disable the feature — the failure
appears in the consumer's project, not the author's.

## R-08 — Pass differences are handled per pass

**Requirement.** Code inserted for all passes must be valid in every
pass it reaches; a hook that a pass cannot support is removed for that
pass with the `#undef` pattern guarded by the pass's
`LIL_PASS_*_INCLUDED` define.
**Source.** The insertion file is substituted into every pass's
program; passes differ in inputs and state (the shadow caster has no
`fd` lighting state), and only `UNPACK_V2F` of the fragment hooks runs
outside the forward pass.
**If violated.** A forward-only assumption becomes a compile error in
the shadow or depth pass, reported against generated code.

## R-09 — Variant trimming keeps the switch targets

**Requirement.** Deleting `.lilcontainer` files is free; the
inspector's `ReplaceToCustomShaders` lookups for every *kept* variant
must exist. Lookups for deleted variants resolve to null, are
unreachable, and may stay.
**Source.** The render-mode UI offers the modes of the family the
material's shader belongs to, so only kept variants are ever looked
up.
**If violated.** A missing lookup for a kept variant leaves that mode
unswitchable; nothing else breaks.

## R-10 — Spellings are verified against the installed lilToon

**Requirement.** Macro names, hook points, pass defines and inspector
virtuals are confirmed against the installed lilToon's generated
shader export and sources before use — including where the official
documentation and the source disagree (in 2.4.*: the hook count, two
no-op input defines, and the singular pass-guard spellings).
**Source.** The template system is lilToon's internal machinery
exposed for custom shaders; it is not a versioned public API, and the
documentation describes it at a different revision than any given
release.
**If violated.** Errors surface as missing defines and unexpanded
macros in generated code, with no message naming the real cause.

## R-11 — Distribution carries lilToon's license and dependency

**Requirement.** A distributed custom shader declares lilToon as a
`vpmDependencies` range in its `package.json`, and includes lilToon's
MIT license text beside its own.
**Source.** The containers resolve lilToon's CustomShaderResources
from the installed package at import time (so the dependency is real,
not optional), and the shipped containers, lilblocks and template
scaffolding are lilToon's MIT-licensed code, whose permission is
conditional on the notice travelling with the code.
**If violated.** Consumers without a compatible lilToon get shaders
that do not generate; and the distribution breaches the license of
the code it is mostly made of.

## R-12 — The platform ceiling is inherited, not chosen

**Requirement.** Treat lilToon's platform support as the custom
shader's own: PC-first, with the fur and tessellation families above
the mobile shader-stage ceiling and no mobile variant family,
fallback chain, or conversion tooling. A design that must run on
Quest/Android excludes those families and the GrabPass / opaque /
depth-texture inputs at design time (lilToon Platform Support).
**Source.** The template system generates variants of one design from
lilToon's resources; it has no mechanism to emit a second, reduced
design or to convert materials onto one, and VRChat's Android avatar
policy excludes every custom shader regardless of features.
**If violated.** The failure is discovered at the consumer's build:
geometry and tessellation passes do not compile for mobile targets,
and a "Quest compatible" claim in the distribution is false for
avatars in all cases.
