# Template: Material Contacts — Receivers writing to shaders

## When it applies

A shader must react to proximity directly, without an
Animator parameter in between: a Receiver with Write to
Shaders updates its target renderers' MaterialPropertyBlock
every frame, and the shader reads the contact buffers. The
Receiver Type is Proximity; one Receiver updates at most 4
renderers.

## Skeleton

The Receiver configuration. The Shader Parameter Override
names the buffer; Parameter is left empty, so the Receiver
stays off the Animator path and is dedicated to shaders:

```csharp
// [Editor: generator]
var receiver = obj.AddComponent<VRCContactReceiver>();
receiver.shapeType = ContactBase.ShapeType.Sphere;
receiver.radius = 0.5f;
receiver.receiverType = ContactReceiver.ReceiverType.Proximity;
receiver.parameter = "";
receiver.separateShaderParameter = "Example";
receiver.writeToMaterials = true;
receiver.writeToMaterialsContactTags = true;
receiver.allowSelf = true;
receiver.allowOthers = true;
receiver.collisionTags = new List<string> { "ExampleTag" };
receiver.shaderTargetRenderers = new[] { targetRenderer };
```

Declaring the buffer and reading the nearest Sender. The
declaration name must match the Receiver's buffer name:

```hlsl
// [Shader: fragment]
#include "Packages/com.vrchat.base/ShaderLibrary/VRCContacts.cginc"

VRC_CONTACT_DECLARE(Example);

int count = VRC_CONTACT_TARGETCOUNT(Example);
if (count > 0)
{
    float4x4 sender = VRC_CONTACT_TRANSFORM(Example, 0);
    float3 senderPos = sender._m03_m13_m23;
}
```

Reading the self frame, the tags, and the owner:

```hlsl
// [Shader: fragment]
float4x4 selfFrame = VRC_CONTACT_TRANSFORM_SELF(Example);
uint tags = VRC_CONTACT_TAGS(Example, 0);
bool hasFirstTag = VRC_CONTACT_GET_TAG(Example, 0, 0) != 0;
int owner = VRC_CONTACT_PLAYERID(Example, 0);
```

A complete Material Contacts shader (ForwardBase only, Built-in
Render Pipeline). The buffer name matches the `Example` of the
Receiver configuration above. The buffers are
StructuredBuffers, so `#pragma target 4.5` is required; the
declaration sits at program scope. With zero targets the
effect is disabled; the nearest Sender is index 0. The vertex
shader turns the closeness to the nearest Sender into a factor
and pushes the vertex along its normal; the fragment shader
blends the texture color toward a highlight color by the same
factor:

```hlsl
// [Shader: complete file]
Shader "Example/Contact Reactive"
{
    Properties
    {
        _MainTex ("Texture", 2D) = "white" {}
        _Color ("Color", Color) = (1, 1, 1, 1)
        _HighlightColor ("Highlight Color", Color) = (1, 0.5, 0, 1)
        _EffectRadius ("Effect Radius", Float) = 0.5
        _PushAmount ("Push Amount", Float) = 0.05
    }
    SubShader
    {
        Tags { "RenderType" = "Opaque" }
        LOD 100

        Pass
        {
            Name "ForwardBase"
            Tags { "LightMode" = "ForwardBase" }

            CGPROGRAM
            #pragma vertex vert
            #pragma fragment frag
            #pragma target 4.5

            #include "UnityCG.cginc"
            #include "Lighting.cginc"
            #include "Packages/com.vrchat.base/ShaderLibrary/VRCContacts.cginc"

            sampler2D _MainTex;
            float4 _MainTex_ST;
            fixed4 _Color;
            fixed4 _HighlightColor;
            float _EffectRadius;
            float _PushAmount;

            // Declaration of the Material Contacts buffer. The name must match the Receiver configuration
            VRC_CONTACT_DECLARE(Example);

            struct appdata
            {
                float4 vertex : POSITION;
                float3 normal : NORMAL;
                float2 uv : TEXCOORD0;
            };

            struct v2f
            {
                float4 pos : SV_POSITION;
                float2 uv : TEXCOORD0;
                float3 normal : TEXCOORD1;
                float proximity : TEXCOORD2;
            };

            v2f vert(appdata v)
            {
                v2f o;
                // Closeness to the nearest Sender as a factor. Zero when there is no target
                float proximity = 0.0;
                if (VRC_CONTACT_TARGETCOUNT(Example) > 0)
                {
                    float4x4 sender = VRC_CONTACT_TRANSFORM(Example, 0);
                    float3 senderPos = sender._m03_m13_m23;
                    float3 worldPos = mul(unity_ObjectToWorld, v.vertex).xyz;
                    float dist = distance(worldPos, senderPos);
                    proximity = saturate(1.0 - dist / max(_EffectRadius, 1e-4));
                }
                // Vertex position processing. Push along the normal by the factor
                v.vertex.xyz += v.normal * (proximity * _PushAmount);
                o.pos = UnityObjectToClipPos(v.vertex);
                // UV processing and the world-space normal
                o.uv = TRANSFORM_TEX(v.uv, _MainTex);
                o.normal = UnityObjectToWorldNormal(v.normal);
                o.proximity = proximity;
                return o;
            }

            fixed4 frag(v2f i) : SV_Target
            {
                // Texture sampling and the base color
                fixed4 tex = tex2D(_MainTex, i.uv) * _Color;
                // Per-pixel lighting
                float3 normal = normalize(i.normal);
                float ndl = max(0.0, dot(normal, _WorldSpaceLightPos0.xyz));
                float3 lighting = UNITY_LIGHTMODEL_AMBIENT.rgb + _LightColor0.rgb * ndl;
                float3 col = tex.rgb * lighting;
                // Blend toward the highlight color by the closeness factor
                col = lerp(col, _HighlightColor.rgb * lighting, i.proximity);
                return fixed4(col, tex.a);
            }
            ENDCG
        }
    }
}
```
