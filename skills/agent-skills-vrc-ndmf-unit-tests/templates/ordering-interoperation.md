# Template: Ordering and interoperation cases

## When it applies

Every plugin carries these cases: the generated output
survives the systems that legitimately restructure the
avatar — Modular Avatar's merges and Avatar Optimizer — and
does not depend on where in the hierarchy its target sits.
The defects this area exists to catch are positional and
temporal: output that works for a target under the root and
fails for a nested one, and references resolved from a
snapshot taken before a restructuring pass moved the target.

The samples run against the test base and the schematic
`NDMFComponent`, generating one FX layer per component with
a constant `m_IsActive` curve on its target.

Two cases in this area are stated, not coded: ORD-02 (ordering and interoperation),
survival through an Avatar Optimizer build. Its fixture is
the Avatar Optimizer package itself, which the adopting
product adds to its own test project; the assertion form is
the one the other cases use — after the optimized build,
the generated layer is present in the FX controller and its
curves' paths resolve to objects in the optimized
hierarchy.

ORD-07 (ordering and interoperation), chain membership
after a reparent by another system, is also stated, not
coded: its consumer is a reactive component, which this
file's stand-in is not, so the adopting product codes it
against its own consumer. The fixture places the component
where its author-side chain is empty — or gated, for the
mirror case — with a ModularAvatarBoneProxy whose target
sits under a toggled gate (or outside it), so the build
moves the component between generation and tracking. The
assertion reads the built layer's forward transition: the
gained ancestor's proxy is among its conditions in the
first form, and the lost ancestor's proxy is absent in the
mirror form. Toggling the gate at runtime is outside the
static scope; the case asserts the condition sets the
built hierarchy implies. The runtime form of the same
fixture — driving the moved component in Av3Emulator — is
in Runtime Verification (references).

## Skeleton

```csharp
// [Editor: unit tests]
// Stand-ins to implement (document-local; contracts in the introduction): NDMFComponent
using System.Linq;
using nadena.dev.modular_avatar.core;
using NUnit.Framework;
using UnityEngine;

public class OrderingInteroperationTests : PluginTestBase
{
    // ORD-01 with ORD-06: the generated layer is in the built FX after the full build.
    [Test]
    public void GeneratedLayerLandsInTheBuiltFx()
    {
        var root = CreateMinimalAvatar();
        CreateChild(root, "Toggle").AddComponent<NDMFComponent>().Target = CreateChild(root, "Target");

        var built = Build(root, out _);

        RequireLayer(FindFxController(built), "NDMFComponent [Target]");
    }

    // ORD-03: the generated curve points at the moved target, under its post-merge path.
    [Test]
    public void CurvesFollowArmatureMerge()
    {
        var root = CreateMinimalAvatar();
        CreateChild(CreateChild(root, "Armature"), "Hips");
        var outfit = CreateChild(root, "Outfit");
        var outfitHips = CreateChild(CreateChild(outfit, "Armature"), "Hips");
        var target = CreateChild(outfitHips, "Target");
        outfit.transform.Find("Armature").gameObject
            .AddComponent<ModularAvatarMergeArmature>().mergeTarget.referencePath = "Armature";
        CreateChild(root, "Toggle").AddComponent<NDMFComponent>().Target = target;

        var built = Build(root, out _);
        var layer = RequireLayer(FindFxController(built), "NDMFComponent [Target]");
        var clip = FindClip(RequireState(layer, "Active"));
        var paths = clip.GetCurveBindings().Select(b => b.path).ToArray();

        // The merge renames the moved object; the expected
        // path is resolved from the built hierarchy (PathOf),
        // and a binding that disagrees with it fails the case.
        var moved = built.GetComponentsInChildren<Transform>()
            .FirstOrDefault(t => t.name == "Target" || t.name.StartsWith("Target$"));
        Assert.NotNull(moved, "the moved target is not in the built avatar");
        CollectionAssert.Contains(paths, PathOf(moved.gameObject));
        Assert.IsFalse(paths.Any(p => p.StartsWith("Outfit/")));
    }

    // ORD-05: component order changes nothing in the generated set.
    [Test]
    public void ComponentOrderDoesNotChangeTheGeneratedSet()
    {
        string[] GeneratedFor(string[] order)
        {
            var root = CreateMinimalAvatar();
            foreach (var name in order)
                CreateChild(root, "Toggle").AddComponent<NDMFComponent>().Target = CreateChild(root, name);
            var built = Build(root, out _);
            var result = FindFxController(built).layers
                .Select(l => l.name + "|" + string.Join(",",
                    FindClip(RequireState(l, "Active")).GetCurveBindings().Select(b => b.path).OrderBy(p => p)))
                .OrderBy(s => s).ToArray();
            TearDown();
            return result;
        }

        var forward = GeneratedFor(new[] { "First", "Second", "Third" });
        var reversed = GeneratedFor(new[] { "Third", "Second", "First" });

        CollectionAssert.AreEqual(forward, reversed);
    }

    // ORD-04: root-level and deeply nested targets generate the same way, each under its own path.
    [Test]
    public void RootAndNestedTargetsBothGenerate()
    {
        var root = CreateMinimalAvatar();
        CreateChild(root, "Toggle").AddComponent<NDMFComponent>().Target = CreateChild(root, "Shallow");
        var deep = CreateChild(CreateChild(CreateChild(root, "A"), "B"), "Deep");
        CreateChild(root, "Toggle").AddComponent<NDMFComponent>().Target = deep;

        var built = Build(root, out _);
        var fx = FindFxController(built);

        RequireLayer(fx, "NDMFComponent [Shallow]");
        var deepLayer = RequireLayer(fx, "NDMFComponent [Deep]");
        var clip = FindClip(RequireState(deepLayer, "Active"));
        Assert.IsTrue(clip.GetCurveBindings().Any(b => b.path == "A/B/Deep"));
    }
}
```
