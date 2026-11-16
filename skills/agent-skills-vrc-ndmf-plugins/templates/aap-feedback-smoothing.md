# Template: Feedback smoothing over animator parameters

## When it applies

A parameter must approach a target parameter gradually —
exponential smoothing (close a fixed fraction of the gap each
frame) or linear smoothing (move at most a fixed step each
frame) — with no runtime script. The animator can do this only
by feeding the output parameter back into its own computation:
the construction writes the output as a blend of "copy the input"
and "keep the output", and the parameter's persistence across
frames is the memory. This is the one place the arithmetic kit
(aap-parameter-arithmetic) depends on frame-to-frame state.

## Requirements

- **Exponential smoothing** is a 1D blend tree whose blend
  parameter is the smoothing amount and whose two children both
  write the output: at 0, a 1D remap of the input onto the
  output; at 1, a 1D remap of the output onto itself (identity
  over the output's own range). The blend yields
  `lerp(input, output, amount)` each frame, which is the
  exponential approach. The layer's state has Write Defaults
  off: the construction relies on nothing resetting the
  parameter between frames.
- **Linear smoothing** is a Direct Blend Tree sum of four terms,
  all writing through 1D remaps over the shared (min, max) range:
  a hidden delta parameter receives the input (weighted by the
  constant-one parameter) and the negated output, so delta =
  input − output; the output receives itself (the base term);
  and a three-threshold 1D tree over delta (clips writing −1, 0,
  +1 to the output at −step, 0, +step) is weighted by the step
  size, contributing `step × clamp(delta / step, −1, 1)`. The
  layer's state has Write Defaults on, because every term —
  including the base — is written every frame and the sum is
  only defined complete.
- The smoothing amount / step size is either a named parameter
  the author drives, or a hidden constant parameter holding the
  configured value (see aap-parameter-arithmetic).
- Smoothing may apply to the local client only, remote clients
  only, or both. For a one-sided target, the whole smoother is
  wrapped in a 1D tree over `IsLocal`: one side selects the
  smoother, the other selects a passthrough (a 1D remap of the
  input straight onto the output). `IsLocal` is declared Float
  in the generated controller — blend parameters must be Float.
  The platform supplies `IsLocal` as Bool; VRChat converts the
  value at runtime for a controller that declares the parameter
  with a different type (VRChat Avatar Platform, Parameter Type
  Conversion), so the tree reads 0 or 1.

## Skeleton

```csharp
// [Editor: generator]
// Exponential: blend between "copy input" and "keep output".
Motion ExponentialSmoothing(AAPParameter input, AAPParameter output, string amountSource)
{
    var copyInput = New1D($"{output} := {input}", input.Parameter,
        input.Min, ConstClip(output.Parameter, input.Min),
        input.Max, ConstClip(output.Parameter, input.Max));
    var keepSelf = New1D($"{output} := {output}", output.Parameter,
        output.Min, ConstClip(output.Parameter, output.Min),
        output.Max, ConstClip(output.Parameter, output.Max));
    return New1D($"ExpSmooth {output}", amountSource, 0, copyInput, 1, keepSelf);
    // Add as a layer with writeDefaultValues: false.
}

// Linear: output + step * clamp((input - output) / step, -1, 1).
Motion LinearSmoothing(AAPParameter input, AAPParameter output, float step, string stepSource)
{
    var deltaName = NewDeltaName(); // a hidden parameter, declared Float
    var root = NewDirect($"LinSmooth {output}", add =>
    {
        add(OneParameter, New1D($"Delta := {input}", input.Parameter,
            input.Min, ConstClip(deltaName, input.Min),
            input.Max, ConstClip(deltaName, input.Max)));
        add(OneParameter, New1D($"Delta := -{output}", output.Parameter,
            input.Min, ConstClip(deltaName, -input.Min),
            input.Max, ConstClip(deltaName, -input.Max)));
        add(OneParameter, New1D($"{output} := {output}", output.Parameter,
            input.Min, ConstClip(output.Parameter, input.Min),
            input.Max, ConstClip(output.Parameter, input.Max)));
        add(stepSource, New1DTriple($"clamp({input}-{output}, ±{step})", deltaName,
            -step, ConstClip(output.Parameter, -1f),
            0f, ConstClip(output.Parameter, 0f),
            +step, ConstClip(output.Parameter, +1f)));
    });
    return root;
    // Add as a layer with writeDefaultValues: true.
}

// One-sided smoothing: select smoother or passthrough by IsLocal.
Motion WrapWithIsLocal(Motion smoother, AAPParameter input, AAPParameter output, bool localOnly)
{
    var passthrough = New1D($"Passthrough {output}", input.Parameter,
        input.Min, ConstClip(output.Parameter, input.Min),
        input.Max, ConstClip(output.Parameter, input.Max));
    return localOnly
        ? New1D($"LocalOnly {output}", "IsLocal", 0, passthrough, 1, smoother)
        : New1D($"RemoteOnly {output}", "IsLocal", 0, smoother, 1, passthrough);
}
```
