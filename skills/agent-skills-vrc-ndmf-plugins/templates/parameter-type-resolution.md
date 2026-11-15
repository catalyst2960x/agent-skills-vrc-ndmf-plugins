# Template: Parameter type resolution order

## When it applies

A condition names a parameter as free text. To declare that
parameter in a generated controller, and to compute reversed
conditions and thresholds, the generator needs the parameter's
type — without validating the name. A name the build cannot type
is still accepted: the platform converts some type mismatches at
runtime, but a mismatch inside generated output is a defect, so
registration, inversion, and threshold arithmetic all work in the
resolved source type.

## Requirements

- Resolution consults three sources, in this order, and stops at
  the first hit:
  1. The build-time parameter set (NDMF `ParameterInfo` for the
     avatar), keyed by effective name. PhysBone parameters are
     expanded through `SubParameters()`: each derived name
     (effective name + the suffix beyond the original name) is
     indexed with its own type.
  2. A built-in catalogue of the platform's own parameters
     (name → type), because built-ins are typed by the platform,
     not by `ParameterInfo`.
  3. No hit: the type is unknown, and the caller registers the
     parameter as Float.
- The name itself is never validated for existence, and an
  unresolvable name never skips the component and never raises a
  report. Mistakes are prevented at input time by suggestions,
  not detected at build time.
- Built-in parameters are declared in the generated controller
  with the catalogue's type; the plugin does not declare them as
  its own parameters or rename them.

## Skeleton

```csharp
// [Editor: NDMF pass]
static Dictionary<string, AnimatorControllerParameterType?> BuildIndex(BuildContext ctx)
{
    var index = new Dictionary<string, AnimatorControllerParameterType?>();
    var provided = ParameterInfo.ForContext(ctx).GetParametersForObject(ctx.AvatarRootObject);
    foreach (var parameter in provided)
    {
        if (parameter.Source is VRCPhysBone)
        {
            foreach (var sub in parameter.SubParameters())
            {
                var suffix = sub.OriginalName.Substring(parameter.OriginalName.Length);
                var key = parameter.EffectiveName + suffix;
                if (!index.ContainsKey(key)) index[key] = sub.ParameterType;
            }
        }
        else if (!index.ContainsKey(parameter.EffectiveName))
        {
            index[parameter.EffectiveName] = parameter.ParameterType;
        }
    }
    return index;
}

static AnimatorControllerParameterType? ResolveType(
    Dictionary<string, AnimatorControllerParameterType?> index, string name)
{
    if (!string.IsNullOrEmpty(name) && index.TryGetValue(name, out var type)) return type;
    if (!string.IsNullOrEmpty(name) && BuiltinParameters.TryGetValue(name, out var builtin)) return builtin;
    return null;
}

// At registration, unknown resolves to Float:
controller.AddParameter(new AnimatorControllerParameter
{
    name = name,
    type = ResolveType(index, name) ?? AnimatorControllerParameterType.Float,
});
```
