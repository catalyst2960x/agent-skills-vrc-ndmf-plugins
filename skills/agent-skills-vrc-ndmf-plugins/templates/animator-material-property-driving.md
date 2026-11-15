# Template: Driving material properties from clips

## When it applies

A generated clip drives a material property on a renderer:
toggling a shader feature, supplying an override value, or
forwarding mesh-side state to the shader. The binding is the
renderer's path, the renderer's type, and `material.<name>`;
the property must be declared in the shader's Properties.

## Skeleton

Driving a Float. The target is the SkinnedMeshRenderer of an
object named `Body` directly under the root; the driven
property is the Float `_ExampleValue`:

```csharp
// [Editor: generator]
var clip = new AnimationClip();
var binding = EditorCurveBinding.FloatCurve(
    "Body", typeof(SkinnedMeshRenderer), "material._ExampleValue");
AnimationUtility.SetEditorCurve(clip, binding, AnimationCurve.Constant(0f, 0f, 1f));
```

On the shader side, declare the property in Properties and
read it as a plain uniform:

```hlsl
// [Shader: fragment]
Properties
{
    _ExampleValue ("Example Value", Float) = 0
}
```

```hlsl
// [Shader: fragment]
float _ExampleValue;
```

Driving a Color — four curves, one per component:

```csharp
// [Editor: generator]
var clip = new AnimationClip();
foreach (var channel in new[] { "r", "g", "b", "a" })
{
    var binding = EditorCurveBinding.FloatCurve(
        "Body", typeof(SkinnedMeshRenderer), "material._ExampleColor." + channel);
    AnimationUtility.SetEditorCurve(clip, binding, AnimationCurve.Constant(0f, 0f, 1f));
}
```

Driving a texture offset — through the components of
`<texture name>_ST` (x and y are the scale, z and w the
offset):

```csharp
// [Editor: generator]
var clip = new AnimationClip();
var binding = EditorCurveBinding.FloatCurve(
    "Body", typeof(SkinnedMeshRenderer), "material._MainTex_ST.z");
AnimationUtility.SetEditorCurve(clip, binding, AnimationCurve.Linear(0f, 0f, 1f, 1f));
```

Per-renderer values — declare the same name in Properties
and in the instancing buffer, and read it through the macro.
The clip-side binding is the same as for a plain Float; the
values separate per renderer because of how the shader reads
them:

```hlsl
// [Shader: fragment]
Properties
{
    [HideInInspector] _ExamplePerRenderer ("Per Renderer Value", Float) = 0
}
```

```hlsl
// [Shader: fragment]
UNITY_INSTANCING_BUFFER_START(ExampleProps)
    UNITY_DEFINE_INSTANCED_PROP(float, _ExamplePerRenderer)
UNITY_INSTANCING_BUFFER_END(ExampleProps)

float value = UNITY_ACCESS_INSTANCED_PROP(ExampleProps, _ExamplePerRenderer);
```
