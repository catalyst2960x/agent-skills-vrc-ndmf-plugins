# Template: Mesh processing cases

## When it applies

A mesh-processing plugin carries these cases: against a
fixture mesh whose values are known exactly, the processed
mesh matches the expected result in full — every vertex,
not sampled points; degenerate inputs produce the defined
result without an exception or a non-finite value;
surviving vertices keep every data channel; and the
author's mesh is unchanged, because processing works on a
copy. The fixture mesh is built in code, so the expected
values are computed in the test from the same numbers.

The samples run against a document-local stand-in,
`NDMFMeshComponent`, whose contract the intro states: it
removes the blend shape named by its `RemoveBlendShape`
field from its target renderer and renormalizes the
remaining bone weights of every vertex to sum to 1.

## Skeleton

```csharp
// [Editor: unit tests]
// Stand-ins to implement (document-local; contracts in the introduction): NDMFMeshComponent
using System.Linq;
using NUnit.Framework;
using UnityEngine;

public class MeshProcessingTests : PluginTestBase
{
    // A two-vertex skinned mesh with UV and colour channels:
    // vertex 0 fully on bone 0, vertex 1 split 0.25/0.75
    // across bones 0 and 1, and one blend shape frame that
    // moves vertex 0 up by 0.5.
    Mesh CreateFixtureMesh()
    {
        var mesh = Track(new Mesh());
        mesh.vertices = new[] { new Vector3(0, 0, 0), new Vector3(1, 0, 0) };
        mesh.uv = new[] { new Vector2(0, 0), new Vector2(1, 0) };
        mesh.colors = new[] { Color.red, Color.green };
        mesh.boneWeights = new[]
        {
            new BoneWeight { boneIndex0 = 0, weight0 = 1f },
            new BoneWeight { boneIndex0 = 0, weight0 = 0.25f, boneIndex1 = 1, weight1 = 0.75f },
        };
        mesh.bindposes = new[] { Matrix4x4.identity, Matrix4x4.identity };
        mesh.AddBlendShapeFrame("Smile", 100f,
            new[] { new Vector3(0, 0.5f, 0), Vector3.zero },
            new Vector3[2], new Vector3[2]);
        return mesh;
    }

    GameObject CreateSkinnedTarget(GameObject root, string name, Mesh mesh)
    {
        var target = CreateChild(root, name);
        var renderer = target.AddComponent<SkinnedMeshRenderer>();
        renderer.sharedMesh = mesh;
        renderer.bones = new[] { root.transform, root.transform };
        return target;
    }

    void AddMeshComponent(GameObject root, GameObject target)
    {
        var component = CreateChild(root, "Mesh").AddComponent<NDMFMeshComponent>();
        component.Target = target;
        component.RemoveBlendShape = "Smile";
    }

    static Mesh BuiltMesh(GameObject built, string name) =>
        built.transform.Find(name).GetComponent<SkinnedMeshRenderer>().sharedMesh;

    // MESH-01 (mesh processing): the full result matches the fixture — positions, renormalized weights, blend shape gone.
    [Test]
    public void ProcessedMeshMatchesInFull()
    {
        var root = CreateMinimalAvatar();
        var target = CreateSkinnedTarget(root, "Face", CreateFixtureMesh());
        AddMeshComponent(root, target);

        var built = Build(root, out _);
        var mesh = BuiltMesh(built, "Face");

        Assert.AreEqual(-1, mesh.GetBlendShapeIndex("Smile"));
        CollectionAssert.AreEqual(
            new[] { new Vector3(0, 0, 0), new Vector3(1, 0, 0) }, mesh.vertices);
        Assert.AreEqual(1f, mesh.boneWeights[0].weight0);
        Assert.AreEqual(0.25f, mesh.boneWeights[1].weight0);
        Assert.AreEqual(0.75f, mesh.boneWeights[1].weight1);
        foreach (var weight in mesh.boneWeights)
            Assert.AreEqual(1f, weight.weight0 + weight.weight1 + weight.weight2 + weight.weight3, 1e-5f);
    }

    // MESH-02: degenerate inputs (an empty mesh, a fully selected mesh) produce the defined result.
    [Test]
    public void DegenerateMeshesAreDefined()
    {
        var emptyRoot = CreateMinimalAvatar();
        var emptyMesh = Track(new Mesh());
        var emptyTarget = CreateSkinnedTarget(emptyRoot, "Face", emptyMesh);
        AddMeshComponent(emptyRoot, emptyTarget);

        var builtEmpty = Build(emptyRoot, out var emptyErrors);
        Assert.AreEqual(0, BuiltMesh(builtEmpty, "Face").vertexCount);
        Assert.IsFalse(emptyErrors.Any(e =>
            e.TheError.Severity == nadena.dev.ndmf.ErrorSeverity.Error));

        var fullRoot = CreateMinimalAvatar();
        var fullTarget = CreateSkinnedTarget(fullRoot, "Face", CreateFixtureMesh());
        AddMeshComponent(fullRoot, fullTarget);

        var builtFull = Build(fullRoot, out _);
        Assert.IsFalse(BuiltMesh(builtFull, "Face").vertices.Any(v =>
            float.IsNaN(v.x) || float.IsNaN(v.y) || float.IsNaN(v.z)));
    }

    // MESH-03: surviving vertices keep every data channel at the fixture's values.
    [Test]
    public void SurvivingChannelsAreKept()
    {
        var root = CreateMinimalAvatar();
        var target = CreateSkinnedTarget(root, "Face", CreateFixtureMesh());
        AddMeshComponent(root, target);

        var built = Build(root, out _);
        var mesh = BuiltMesh(built, "Face");

        CollectionAssert.AreEqual(new[] { new Vector2(0, 0), new Vector2(1, 0) }, mesh.uv);
        CollectionAssert.AreEqual(new[] { Color.red, Color.green }, mesh.colors);
    }

    // MESH-04: the author's mesh is unchanged; the processed mesh is a different object.
    [Test]
    public void AuthorsMeshIsUnchanged()
    {
        var root = CreateMinimalAvatar();
        var fixture = CreateFixtureMesh();
        var target = CreateSkinnedTarget(root, "Face", fixture);
        AddMeshComponent(root, target);

        var built = Build(root, out _);

        Assert.AreNotSame(fixture, BuiltMesh(built, "Face"));
        Assert.AreNotEqual(-1, fixture.GetBlendShapeIndex("Smile"));
        Assert.AreEqual(2, fixture.vertexCount);
    }
}
```
