# Template: Logic gates from nested 1D blend trees

## When it applies

Bool-valued animator parameters (carried as Floats, 0 and 1)
must be combined at runtime — AND, OR, NOT, or an arbitrary
two-input truth table — with no runtime script. A 1D blend tree
over one input, whose children are constant parameter clips
(aap-parameter-arithmetic), is a one-input function; nesting a
second 1D tree as one of the children makes it a two-input
function whose shape is exactly the truth table.

## Requirements

- Inputs are read as Floats with thresholds 0 and 1; the output
  is written by constant clips of 0 and 1 (or, for the arbitrary
  gate, by the four configured truth values, which may be any
  floats).
- AND: the outer tree over input A maps 0 → clip(0) and 1 → an
  inner tree over input B mapping 0 → clip(0), 1 → clip(1).
- OR: the outer tree maps 0 → the inner tree (0 → clip(0),
  1 → clip(1)) and 1 → clip(1).
- NOT: one tree mapping 0 → clip(1), 1 → clip(0).
- Arbitrary 2-bit gate: two inner trees over B, one per value of
  A, holding the truth table's rows (A=0: values for B=0 and
  B=1; A=1: likewise), selected by the outer tree over A. AND,
  OR, and NOT are special cases of this shape; the dedicated
  forms exist so the common gates stay small.
- Gate layers have Write Defaults off: the output is fully
  determined by the tree each frame, and nothing else may write
  defaults over it.

## Skeleton

```csharp
// [Editor: generator]
Motion AndGate(string inputA, string inputB, string output)
{
    var clip0 = ConstClip(output, 0f);
    var clip1 = ConstClip(output, 1f);
    var inner = New1D($"AND inner {output}", inputB, 0f, clip0, 1f, clip1);
    return New1D($"AND {output}", inputA, 0f, clip0, 1f, inner);
}

Motion OrGate(string inputA, string inputB, string output)
{
    var clip0 = ConstClip(output, 0f);
    var clip1 = ConstClip(output, 1f);
    var inner = New1D($"OR inner {output}", inputB, 0f, clip0, 1f, clip1);
    return New1D($"OR {output}", inputA, 0f, inner, 1f, clip1);
}

Motion NotGate(string input, string output)
{
    return New1D($"NOT {output}", input, 0f, ConstClip(output, 1f), 1f, ConstClip(output, 0f));
}

Motion Arbitrary2BitGate(string inputA, string inputB, string output,
    float truth00, float truth01, float truth10, float truth11)
{
    var innerA0 = New1D($"Arb2Bit A=0 {output}", inputB, 0f, ConstClip(output, truth00), 1f, ConstClip(output, truth01));
    var innerA1 = New1D($"Arb2Bit A=1 {output}", inputB, 0f, ConstClip(output, truth10), 1f, ConstClip(output, truth11));
    return New1D($"Arb2Bit {output}", inputA, 0f, innerA0, 1f, innerA1);
}
```
