# Distribution

Shipping a finished custom shader so other projects can install it.
The shipped form in current use is a VPM package; a `.unitypackage`
export of the same folder layout is the older form and follows the same
layout rules.

## Package layout

The package root carries the manifest and the two template folders,
renamed with the product:

```
<package root>/
    package.json
    LICENSE.txt                (the product's own license)
    LICENSE-lilToon.txt        (lilToon's license — see the duty below)
    Shader/
        custom.hlsl, custom_insert.hlsl
        lilCustomShaderDatas.lilblock, lilCustomShaderInsert.lilblock,
        lilCustomShaderProperties.lilblock
        *.lilcontainer         (the trimmed variant set, P-03)
    Editor/
        CustomInspector.cs
        <product>.Editor.asmdef
        strings.tsv            (only with a language file, P-09)
    Examples/                  (optional sample materials/scenes)
```

Nothing in the layout references the lilToon package's files: the
containers resolve lilToon's CustomShaderResources (its bundled
SubShader bodies and property fragments) from the installed
package at import time, so the distribution stays small and a lilToon
update does not require re-shipping lilToon's own files.

## package.json

The fields a custom-shader package needs, beyond the standard VPM
manifest fields (name in reverse-domain form, displayName, version,
unity, license, author):

```
"vpmDependencies": {
    "jp.lilxyzw.liltoon": ">=2.0.0"
}
```

- Declare lilToon under `vpmDependencies`, as a range, never under
  `dependencies` (which is for Unity registry packages). The range's
  floor is the oldest lilToon version the shader's used surface
  (P-01's record: macros, hooks, inspector virtuals, helpers) is
  verified against — not the version the author happens to run.
- Keep the floor honest across releases: when a lilToon update changes
  a hook, a helper, or a stage the shader copies (Fragment Hooks, the
  copy trap), re-verify and raise the floor or adapt the code in the
  same release. A copied stage silently diverging from its original is
  the failure mode this rule exists to prevent.
- A shader that includes a VRChat SDK header also declares the SDK
  side. `VRCContacts.cginc` lives in `com.vrchat.base`, which
  `com.vrchat.avatars` brings in as its own dependency: declare
  `com.vrchat.avatars`, not `com.vrchat.base` directly.
- `legacyFolders` / `legacyFiles`: declare the Assets-path of any
  earlier Assets-based or unitypackage distribution, so installing the
  VPM package removes the old copy instead of duplicating the shaders.
  The value is the folder's GUID when the GUID is fixed; when the
  distribution does not pin GUIDs (no tracked .meta files), use an
  empty string — the entry then matches by path only.

## The license duty

The shipped files are not all the author's code. The `.lilcontainer`
files, the lilblocks, `custom.hlsl`'s template scaffolding and the
inspector's template body derive from lilToon's template, and lilToon
is MIT-licensed: the distribution must carry lilToon's license text and
copyright notice for those parts. Shipped packages do this with a
separate file (for example `LICENSE-lilToon.txt`) beside the product's
own license. Removing lilToon's license because "the template was
provided for this use" is not an option the MIT license offers — the
permission is conditional on the notice travelling with the code.

## Release checklist

1. Variant set in the package = the set P-03 decided (no template
   leftovers, no missing `ltspass_*`).
2. `vpmDependencies` floor = the version P-01 last verified.
3. Both license files present.
4. Install test into a clean project that has only the declared lilToon
   range installed: shaders generate, the inspector binds, a material
   switches rendering modes (P-10's checks, run in the consumer's
   position).
