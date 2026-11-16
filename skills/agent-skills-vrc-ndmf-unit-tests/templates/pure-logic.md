# Template: Pure logic cases

## When it applies

A plugin with logic that computes without an avatar carries
these cases: degenerate and special values return the
defined result, and a long operation sequence preserves the
structure's invariants. They run without a build, over
tables and seeded sequences, so a boundary defect is pinned
by a row, not discovered in a built controller.

The samples run against document-local stand-ins whose
contracts the intro states: `Remap.Convert`, the linear
remap of a value from a source range to a destination
range, where a coincident source range (min equal to max)
returns the destination minimum, and inputs outside the
source range clamp to it; and `NameSet`, a set of layer
names that numbers a repeated name on insertion, whose
invariant is that its names are always distinct.

## Skeleton

```csharp
// [Editor: unit tests]
// Stand-ins to implement (document-local; contracts in the introduction): Remap, NameSet
using System;
using System.Linq;
using NUnit.Framework;

public class PureLogicTests
{
    // LOGIC-01 (pure logic): degenerate and special inputs return the defined, finite result.
    [TestCase(0.0, 0.0, 1.0, -1.0, 1.0, -1.0)]
    [TestCase(1.0, 0.0, 1.0, -1.0, 1.0, 1.0)]
    [TestCase(0.5, 0.0, 1.0, -1.0, 1.0, 0.0)]
    [TestCase(0.0, 2.0, 2.0, -1.0, 1.0, -1.0)]
    [TestCase(-3.0, 0.0, 1.0, -1.0, 1.0, -1.0)]
    [TestCase(3.0, 0.0, 1.0, -1.0, 1.0, 1.0)]
    public void DegenerateAndSpecialValuesAreDefined(
        double value, double srcMin, double srcMax,
        double dstMin, double dstMax, double expected)
    {
        var result = Remap.Convert(value, srcMin, srcMax, dstMin, dstMax);

        Assert.AreEqual(expected, result);
        Assert.IsFalse(double.IsNaN(result));
    }

    [Test]
    public void InfiniteInputClampsToTheRangeEnd()
    {
        Assert.AreEqual(1.0, Remap.Convert(double.PositiveInfinity, 0, 1, -1, 1));
        Assert.AreEqual(-1.0, Remap.Convert(double.NegativeInfinity, 0, 1, -1, 1));
    }

    // LOGIC-02: a seeded random insertion sequence preserves the invariant after every step.
    [Test]
    public void RandomSequencesPreserveTheInvariant()
    {
        var random = new Random(20261115);
        var set = new NameSet();
        var pool = new[] { "A", "B", "C" };

        for (var i = 0; i < 200; i++)
        {
            set.Add(pool[random.Next(pool.Length)]);
            var names = set.Names.ToArray();
            Assert.AreEqual(names.Length, names.Distinct().Count());
        }
    }
}
```
