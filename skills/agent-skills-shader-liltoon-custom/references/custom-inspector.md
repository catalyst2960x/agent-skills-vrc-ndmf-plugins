# Custom inspector

Extending lilToon's material inspector for a custom shader: the derived
class, how it binds to the generated shader, the two virtual methods,
the render-mode variant table, and the shared styles and helpers.

## The derived class

The template's inspector is a class in namespace `lilToon` deriving from
`lilToonInspector` (a `ShaderGUI`, split across `Editor/lilInspector/`),
in an Editor-only assembly whose asmdef references `lilToon.Editor`:

```
public class TemplateFullInspector : lilToonInspector
{
    private const string shaderName = "TemplateFull";
    ...
}
```

The rename set (P-02) for the inspector side: the class name, the
`shaderName` constant, the `EditorName` value in
`lilCustomShaderDatas.lilblock` (set to the class's full name, for
example `lilToon.TemplateFullInspector`), and the asmdef `name` plus
file name. The generated shader binds the inspector through its
`CustomEditor "*LIL_EDITOR_NAME*"` line, which receives the `EditorName`
value — there is no name field on the base class; binding is by class
name only.

The `ShaderName` value in `lilCustomShaderDatas.lilblock` and the
inspector's `shaderName` constant are two spellings of one string
and must match exactly. If they differ, every `Shader.Find` in
`ReplaceToCustomShaders` returns null and the render-mode switch
has no targets — silently: the inspector still draws, and the
material still renders on whichever variant it already has.

## The two virtual methods

`lilToonInspector` exposes exactly two extension points for custom
properties (declared in `Editor/lilInspector/lilInspector.cs`):

```
protected virtual void LoadCustomProperties(MaterialProperty[] props, Material material)
protected virtual void DrawCustomProperties(Material material)
```

The template's implementations:

- `LoadCustomProperties` sets `isCustomShader = true`, calls
  `ReplaceToCustomShaders()` (below), sets
  `isShowRenderMode = !material.shader.name.Contains("Optional")`, and
  loads each custom property with `FindProperty("_Custom...", props)`
  into `MaterialProperty` fields declared on the class. A custom
  language file is loaded here with `LoadCustomLanguage("<guid>")` —
  the argument is the language file asset's GUID; a shipped package
  passes the GUID of the TSV it carries in its Editor folder, which is
  what makes the key-form property display names resolve (Properties,
  Display names as localisation keys).
- `DrawCustomProperties` is called by the base GUI under a "Custom
  Properties" heading when `isCustomShader` is set. It draws a foldout
  section with the shared styles and renders each property with
  `m_MaterialEditor.ShaderProperty(prop, "Label")` or the
  `lilEditorGUI` helpers.

## Render-mode switching: ReplaceToCustomShaders

lilToon's inspector switches a material between rendering modes by
reassigning its shader to the mode's variant. To do that it holds one
Shader field per variant (lts, ltsc, ltst, ...), which for lilToon's own
shaders point at the built-in variants. For a custom shader those
fields must point at the custom variants instead, which the
derived class does by overriding `ReplaceToCustomShaders()` and filling
every field with `Shader.Find(shaderName + "/...")` /
`Shader.Find("Hidden/" + shaderName + "/...")` lookups — the template
enumerates about 60 variants (base, lite and multi families; cutout,
transparent, one/two-pass transparent; outline and one-pass-outline
forms; tessellation set; refraction, fur, gem, fake shadow, overlays).

Maintenance rule: this table and the `.lilcontainer` set are two views
of the same variant list, but they do not have to be trimmed together.
A lookup for a **kept** variant must exist — without it the mode switch
has no target. A lookup for a **deleted** variant returns null and is
unreachable: the render-mode UI offers the modes of the family the
material's shader belongs to, so a material on a kept family never
takes a deleted family's field. Shipped packages do both — one trims
its containers to the full family only and keeps all 52 lookups. Treat
trimming the table as tidiness, not correctness; treat a missing lookup
for a kept variant as a defect. When `isShowRenderMode` is false (the
template sets it false for `[Optional]` shaders), the table is unused
but harmless.

## Shared styles and helpers

Styles (from the static `lilEditorGUI` class, also reachable as
inherited members in the template's usage):

| Style | Use |
| --- | --- |
| `boxOuter` | Outer box of a property section |
| `boxInnerHalf` | Inner box when a label is drawn (square top corners) |
| `boxInner` | Inner box without a label (rounded top corners) |
| `customBox` | Box close to Unity's default, with an outline for visibility |
| `customToggleFont` | Bold font for section labels |

Helpers used by custom inspectors:

| Helper | Use |
| --- | --- |
| `bool Foldout(string title, string help, bool display)` | Section foldout |
| `void DrawLine()` | Separator line |
| `void DrawWebButton(string text, string URL)` | Web link button |
| `void LoadCustomLanguage(string langFileGUID)` | Loads an additional localisation file |
| `GetLoc("...")` | The current language's text for a localisation key, from lilToon's language files plus any custom file loaded |

Further shared controls exist on `lilEditorGUI` (`DrawSimpleFoldout`,
`Popup`, `Slider`, `IntSlider`, `FloatField`, `Toggle`, `Vector2Field`,
`LocalizedProperty`, `AutoFixHelpBox`, `DrawHelpButton`). Prefer these
over raw `EditorGUILayout` calls for controls that also appear in the
stock inspector, so the custom section reads as part of the same
inspector.

## Section menu (copy, paste, reset, manual)

Every stock section header carries a popup menu at its right end,
drawn by `DrawMenuButton` in the base inspector: Copy, Paste, Paste
With Texture, Reset and Open Manual (localisation keys `sCopy`,
`sPaste`, `sPasteWithTexture`, `sReset`, `sOpenManual`). The
operations run per `PropertyBlock` over lilToon's property registry
(`AllProperties()` in lilMaterialProperties.cs): each entry is a
readonly `lilMaterialProperty` field tagged with the blocks it
belongs to, and `PropertyBlock` is a closed enum in lilToon's
assembly.

Custom properties are not in that registry. A property loaded with
`FindProperty` in `LoadCustomProperties` is never registered, the
registry and `AllProperties()` are private, and a derived inspector
cannot add entries or blocks. A custom foldout therefore has no
section menu unless the inspector draws its own; the stock
implementations cannot be reused for custom properties.

To provide the menu in the stock form:

1. After drawing the foldout header, place the button on the header
   rect: take `GUILayoutUtility.GetLastRect()`, offset `x` by
   `width - 24`, set `width = 24`, and draw
   `EditorGUIUtility.IconContent("_Popup")` with the `middleButton`
   style (a protected static member, reachable from a derived
   inspector).
2. Open a `GenericMenu` with the same labels through `GetLoc`:
   `sCopy`, `sPaste`, `sPasteWithTexture`, `sReset` (guarded by
   `UNITY_2019_3_OR_NEWER`, as in the base inspector) and
   `sOpenManual`.
3. Match the semantics, not only the labels. Copy stores the
   foldout's property values typed (float, int, colour, vector,
   texture) plus each texture's tiling and offset. Paste matches by
   name and type and excludes texture properties; Paste With Texture
   also writes the textures and their tiling and offset. Reset
   assigns the defaults declared in the shader through
   `Shader.FindPropertyIndex` and
   `Shader.GetPropertyDefaultFloatValue` /
   `GetPropertyDefaultVectorValue`, and null for textures — the same
   APIs the stock `ResetProperties` uses. Open Manual composes
   `GetLoc("sManualURL")` + an anchor key, as `OpenHelpPage` does;
   a custom foldout has no anchor of its own in lilToon's language
   files, so reuse the anchor of the stock section the foldout is
   closest to.
4. Keep the clipboard scope to the foldout's own property set. A
   static field survives a change of the selected material and is
   lost on a domain reload; state that lifetime where the product
   documents the feature.

## Feature toggles and the menu button

When a custom section groups one switchable feature, give the feature
an On/Off checkbox and keep the section menu on the foldout header
only:

- The enable state is a dedicated toggle property of its own (an int
  with `[MaterialToggle]`), and the checkbox displays that property —
  inverted when the property is phrased as a disable flag. Do not
  derive the checkbox from a value property (an amount, a strength):
  a value of zero would read as "off", and the row that switches the
  feature back on must stay operable independently of the values it
  gates.
- Turning the feature off disables its property rows; the section,
  its values and the checkbox stay drawn, so what is set can be
  inspected and the feature switched back on from the same place.
- The section menu is drawn once per section, on the foldout header.
  Do not draw a second menu button inside the section — not on the
  checkbox row, not on a label row: the row's operation is the
  switch, the menu's property-block operations belong to the header,
  and a second button duplicates them.
- Keep the menu's item set the same in every section of the
  inspector. A section adds an item only when that section has the
  item's target (a manual anchor of its own, for example).

With several features in one inspector, each feature carries its own
toggle property and its own checkbox under these rules; the toggles
are independent, and one feature's state never gates another
feature's rows.

## Material conversion menu (optional)

The template carries a commented-out `MenuItem` example that converts
selected materials to the custom shader through
`ConvertMaterialToCustomShader`. Uncomment and rename it if the product
wants an Assets-menu conversion entry; it is not required for the
inspector to work.
