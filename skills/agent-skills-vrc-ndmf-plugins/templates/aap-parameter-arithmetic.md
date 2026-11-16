# Template: Animator parameter arithmetic with blend trees

## When it applies

An avatar feature needs arithmetic over animator parameters at
runtime — remap one parameter's range onto another, add or
multiply two parameters, divide — and no script runs on an
uploaded avatar. The only runtime computer is the animator
itself. This construction kit makes blend trees do the
arithmetic: a clip can write a constant INTO an animator
parameter (an "animated animator parameter"), a 1D blend tree
over an input parameter interpolates between two such constants,
and a Direct Blend Tree sums weighted contributions. Every
operation below is an assembly of those three facts.

## Requirements

- A constant parameter clip has one keyframe at time 0 on the
  binding `("", typeof(Animator), parameterName)`. Clips are
  cached per (parameter, value) pair; the same constant is never
  built twice.
- Blend weights must be parameters, so numeric constants the
  author configures become parameters too: a `One` parameter with
  default 1 for "always fully weighted" children, and one hidden
  parameter per distinct constant value, named by an encoding of
  the value (formatted with G9, with `.`/`-`/`+`/`E` replaced so
  the name is a valid identifier) and declared with that value as
  its default. Hidden constants are deduplicated by value.
- Every parameter the construction touches is declared Float
  (a 1D blend tree's blend parameter must be Float), default 0
  unless it is `One` or a hidden constant.
- An input is described by (parameter name, min, max): the range
  the 1D trees interpolate over.
- Operations that are pure sums collect their children into one
  shared root Direct Blend Tree, which becomes the first layer.
  Operations that must control Write Defaults per operation get
  their own layer: a single state holding the motion, weight 1.
- Remap with a 1D tree: children are the constant clips for the
  output's min at threshold = input min and the output's max at
  threshold = input max. Without the 1D form, a Direct child
  weighted by the input and writing the output's max gives a
  plain proportional copy.
- Addition: one Direct child per input, weighted by `One`,
  holding a 1D tree that remaps that input onto the output
  (input min → output value input min, input max → output value
  input max); the Direct sum is the total. Subtraction negates
  the second input's two clip values.
- Multiplication: a Direct child weighted by input1 whose motion
  is a Direct tree with one child weighted by input2 writing the
  constant 1 to the output — the nested weights multiply.
- Division (by 1 + input): a Direct tree with normalized blend
  values — children: the input-weighted empty clip, and a
  `One`-weighted clip writing the output's max. Normalization
  divides by the weight sum, giving max × 1/(1 + input). The
  normalized flag has no public setter; it is set through a
  SerializedObject property on the tree.

## Skeleton

```csharp
// [Editor: generator]
// A constant written into a parameter; cached per (parameter, value).
AnimationClip ConstClip(string parameter, float value)
{
    if (!_clips.TryGetValue((parameter, value), out var clip))
    {
        clip = new AnimationClip { name = $"Const {parameter} -> {value}", hideFlags = HideFlags.HideInHierarchy };
        clip.SetCurve("", typeof(Animator), parameter,
            new AnimationCurve { keys = new[] { new Keyframe { time = 0, value = value } } });
        _clips.Add((parameter, value), clip);
    }
    return clip;
}

// A configured constant becomes a hidden parameter holding it.
string HiddenConst(float value)
{
    if (!_hiddenConsts.TryGetValue(value, out var name))
    {
        name = $"__Const_{value:G9}__".Replace('.', '_').Replace('-', 'm').Replace('+', 'p').Replace('E', 'e');
        _hiddenConsts.Add(value, name);
    }
    return name;
}

BlendTree New1D(string name, string parameter, float startThreshold, Motion start, float endThreshold, Motion end)
{
    return new BlendTree
    {
        name = name,
        hideFlags = HideFlags.HideInHierarchy,
        blendType = BlendTreeType.Simple1D,
        useAutomaticThresholds = false,
        blendParameter = parameter,
        children = new[]
        {
            new ChildMotion { motion = start, threshold = startThreshold },
            new ChildMotion { motion = end, threshold = endThreshold },
        },
    };
}

// The three-child form, for clamps and other three-point remaps.
BlendTree New1DTriple(string name, string parameter,
    float t0, Motion m0, float t1, Motion m1, float t2, Motion m2)
{
    return new BlendTree
    {
        name = name,
        hideFlags = HideFlags.HideInHierarchy,
        blendType = BlendTreeType.Simple1D,
        useAutomaticThresholds = false,
        blendParameter = parameter,
        children = new[]
        {
            new ChildMotion { motion = m0, threshold = t0 },
            new ChildMotion { motion = m1, threshold = t1 },
            new ChildMotion { motion = m2, threshold = t2 },
        },
    };
}

// Builder state shared by the operations: the constant caches
// above, the shared empty clip, and the children collected for
// the root Direct tree (the first layer, assembled from
// _rootChildren when the layer is built).
readonly Dictionary<(string, float), AnimationClip> _clips = new();
readonly Dictionary<float, string> _hiddenConsts = new();
readonly List<ChildMotion> _rootChildren = new();
AnimationClip _empty;
int _deltaSeq;

const string OneParameter = "One"; // declared Float, default 1

// An input or output: a parameter and the range the 1D trees
// interpolate over. A plain struct, not a record: a positional
// record does not compile in a Unity 2022.3 Editor assembly
// without an IsExternalInit polyfill.
readonly struct AAPParameter
{
    public AAPParameter(string parameter, float min, float max)
    {
        Parameter = parameter;
        Min = min;
        Max = max;
    }

    public string Parameter { get; }
    public float Min { get; }
    public float Max { get; }

    public override string ToString() => Parameter;
}

// The clip with no curves, for children that contribute weight
// but write nothing (the division's normalizer).
AnimationClip EmptyClip() => _empty ??= new AnimationClip
{
    name = "Empty",
    hideFlags = HideFlags.HideInHierarchy,
};

// A Direct tree built by callback: the callback receives an add
// function taking a weight parameter and a motion.
BlendTree NewDirect(string name, Action<Action<string, Motion>> build)
{
    var children = new List<ChildMotion>();
    build((weight, motion) => children.Add(new ChildMotion
    {
        motion = motion,
        directBlendParameter = weight,
    }));
    return new BlendTree
    {
        name = name,
        hideFlags = HideFlags.HideInHierarchy,
        blendType = BlendTreeType.Direct,
        children = children.ToArray(),
    };
}

// Pure sums collect into the shared root Direct tree.
void AddRootChild(string weightParameter, Motion motion) =>
    _rootChildren.Add(new ChildMotion
    {
        motion = motion,
        directBlendParameter = weightParameter,
    });

// A fresh hidden parameter name for a smoother's delta; declared
// Float with default 0 like every generated parameter.
string NewDeltaName() => $"__Delta_{_deltaSeq++}";

// Addition: two remapped inputs summed by the root Direct tree.
void Addition(AAPParameter input1, AAPParameter input2, AAPParameter output)
{
    AddRootChild(OneParameter, New1D($"{output} += {input1}", input1.Parameter,
        input1.Min, ConstClip(output.Parameter, input1.Min),
        input1.Max, ConstClip(output.Parameter, input1.Max)));
    AddRootChild(OneParameter, New1D($"{output} += {input2}", input2.Parameter,
        input2.Min, ConstClip(output.Parameter, input2.Min),
        input2.Max, ConstClip(output.Parameter, input2.Max)));
}

// Multiplication (Direct form): nested weights multiply.
void Multiplication(AAPParameter input1, AAPParameter input2, AAPParameter output)
{
    AddRootChild(input1.Parameter, NewDirect("Multiply", add =>
        add(input2.Parameter, ConstClip(output.Parameter, 1))));
}

// Division by (1 + input): normalized weights do the division.
BlendTree Division(AAPParameter input, AAPParameter output)
{
    var direct = NewDirect($"{output} = {output.Max} / (1 + {input})", add =>
    {
        add(input.Parameter, EmptyClip());
        add(OneParameter, ConstClip(output.Parameter, output.Max));
    });
    var so = new SerializedObject(direct);
    so.FindProperty("m_NormalizedBlendValues").boolValue = true;
    so.ApplyModifiedProperties();
    return direct;
}
```
