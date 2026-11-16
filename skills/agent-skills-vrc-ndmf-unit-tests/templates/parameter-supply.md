# Template: Parameter supply cases

## When it applies

A parameter-supplying plugin carries these cases: the
attributes of what it supplies, the guarantee that a
parameter the logic depends on is defined in Expression
Parameters, and the priority by which condition parameter
types resolve. A generating plugin that references parameters
by name carries the type-resolution case as well.

The samples run against the test base and schematic
components: `NDMFComponent`, which supplies a Bool parameter
(`NDMFComponent/Active`) and depends on an Int parameter
(`NDMFComponent/Index`); `NDMFConditionComponent`, which
carries free-text condition parameter names and generates a
layer whose transitions reference them, declaring each on
the controller with its resolved type; and
`NDMFConsumerComponent`, which carries an empty
`ParameterName` field that another component's recorded
assignment can fill; and `NDMFReferencingComponent`, which
declares the same depended-on parameter on its own layer
with a controller-side default of 0. The schematic
component follows the
effectiveness gate: while a generated Modular Avatar
component sits on its own GameObject, the component is
ineffective and supplies nothing. `ParameterSuggestions`
stands for the plugin's suggestion generation: given a
target component, it returns the candidate names in
deterministic scan order, uniquified by the designed rule.

## Skeleton

```csharp
// [Editor: unit tests]
// Stand-ins to implement (document-local; contracts in the introduction): NDMFComponent, NDMFConditionComponent, NDMFConsumerComponent, NDMFReferencingComponent, ParameterSuggestions, Suggested, Assignment
using System.Linq;
using nadena.dev.modular_avatar.core;
using nadena.dev.ndmf;
using NUnit.Framework;
using UnityEditor.Animations;
using UnityEngine;
using VRC.SDK3.Avatars.Components;
using VRC.SDK3.Avatars.ScriptableObjects;
using VRC.SDK3.Dynamics.PhysBone.Components;

public class ParameterSupplyTests : PluginTestBase
{
    // PARAM-01 (parameters): the supplied parameter is present while the component is effective, absent while gated off.
    [Test]
    public void SuppliedParameterFollowsEffectiveness()
    {
        var effective = CreateMinimalAvatar();
        CreateChild(effective, "Toggle").AddComponent<NDMFComponent>().Target = CreateChild(effective, "Target");

        var gated = CreateMinimalAvatar();
        var gatedHolder = CreateChild(gated, "Toggle");
        gatedHolder.AddComponent<NDMFComponent>().Target = CreateChild(gated, "Target");
        gatedHolder.AddComponent<ModularAvatarParameters>();

        var effectiveNames = SuppliedNames(effective);
        var gatedNames = SuppliedNames(gated);

        CollectionAssert.Contains(effectiveNames, "NDMFComponent/Active");
        CollectionAssert.DoesNotContain(gatedNames, "NDMFComponent/Active");
    }

    string[] SuppliedNames(GameObject root)
    {
        var built = Build(root, out _);
        return built.GetComponent<VRCAvatarDescriptor>().expressionParameters.parameters
            .Select(p => p.name).ToArray();
    }

    // PARAM-04: another component's renaming does not catch the supplied parameter's name.
    [Test]
    public void SuppliedNamesSurviveRenaming()
    {
        var root = CreateMinimalAvatar();
        CreateChild(root, "Toggle").AddComponent<NDMFComponent>().Target = CreateChild(root, "Target");
        var renamer = CreateChild(root, "Renamer").AddComponent<ModularAvatarParameters>();
        renamer.parameters = new System.Collections.Generic.List<ParameterConfig>
        {
            new ParameterConfig
            {
                nameOrPrefix = "NDMFComponent/Active",
                remapTo = "Renamed/Active",
            },
        };

        var names = SuppliedNames(root);

        CollectionAssert.Contains(names, "NDMFComponent/Active");
        CollectionAssert.DoesNotContain(names, "Renamed/Active");
    }

    // PARAM-06 and PARAM-07: suggested names are unique per target in deterministic scan order;
    // a same-name candidate of a different type is the collision.
    [Test]
    public void SuggestionsAreUniqueOrderedAndTyped()
    {
        var root = CreateMinimalAvatar();
        var component = CreateChild(root, "Toggle").AddComponent<NDMFComponent>();

        var first = ParameterSuggestions.For(component);
        var second = ParameterSuggestions.For(component);

        CollectionAssert.AreEqual(first, second);
        Assert.AreEqual(first.Length, first.Distinct().Count());
        Assert.IsFalse(ParameterSuggestions.Collides(
            new Suggested("Input", typeof(bool)), new Suggested("Input", typeof(bool))));
        Assert.IsTrue(ParameterSuggestions.Collides(
            new Suggested("Input", typeof(bool)), new Suggested("Input", typeof(float))));
    }

    // PARAM-08: a recorded assignment lands on the build copy only; a filled field is not overwritten.
    [Test]
    public void AssignmentsLandOnTheBuildCopyOnly()
    {
        var root = CreateMinimalAvatar();
        var consumer = CreateChild(root, "Consumer").AddComponent<NDMFConsumerComponent>();
        var filled = CreateChild(root, "Filled").AddComponent<NDMFConsumerComponent>();
        filled.ParameterName = "Authored";
        var assigner = CreateChild(root, "Toggle").AddComponent<NDMFComponent>();
        assigner.Assignments = new[]
        {
            new Assignment { Consumer = consumer, Name = "Assigned" },
            new Assignment { Consumer = filled, Name = "Assigned" },
        };

        var built = Build(root, out _);

        Assert.AreEqual("", consumer.ParameterName);
        Assert.AreEqual("Authored", filled.ParameterName);
        Assert.AreEqual("Assigned",
            built.transform.Find("Consumer").GetComponent<NDMFConsumerComponent>().ParameterName);
        Assert.AreEqual("Authored",
            built.transform.Find("Filled").GetComponent<NDMFConsumerComponent>().ParameterName);
    }

    // PARAM-09: two assignments to one consumer — the first applies, the second reports NonFatal.
    [Test]
    public void SecondAssignmentReportsNonFatal()
    {
        var root = CreateMinimalAvatar();
        var consumer = CreateChild(root, "Consumer").AddComponent<NDMFConsumerComponent>();
        var first = CreateChild(root, "Toggle").AddComponent<NDMFComponent>();
        first.Assignments = new[] { new Assignment { Consumer = consumer, Name = "First" } };
        var second = CreateChild(root, "Toggle").AddComponent<NDMFComponent>();
        second.Assignments = new[] { new Assignment { Consumer = consumer, Name = "Second" } };

        var built = Build(root, out var errors);

        Assert.AreEqual("First",
            built.transform.Find("Consumer").GetComponent<NDMFConsumerComponent>().ParameterName);
        Assert.IsTrue(errors.Any(e => e.TheError.Severity == ErrorSeverity.NonFatal));
    }

    // PARAM-02: the supplied parameter's type, default, sync, and persistence match the settings.
    [Test]
    public void SuppliedParameterAttributesMatch()
    {
        var root = CreateMinimalAvatar();
        CreateChild(root, "Toggle").AddComponent<NDMFComponent>().Target = CreateChild(root, "Target");

        var built = Build(root, out _);
        var parameters = built.GetComponent<VRCAvatarDescriptor>().expressionParameters.parameters;
        var supplied = System.Array.Find(parameters, p => p.name == "NDMFComponent/Active");

        Assert.NotNull(supplied);
        Assert.AreEqual(VRCExpressionParameters.ValueType.Bool, supplied.valueType);
        Assert.IsTrue(supplied.networkSynced);
        Assert.IsTrue(supplied.saved);
    }

    // PARAM-03: a depended-on parameter is defined in Expression Parameters with the designed attributes.
    [Test]
    public void DependedParameterDefinitionIsGuaranteed()
    {
        var root = CreateMinimalAvatar();
        CreateChild(root, "Toggle").AddComponent<NDMFComponent>().Target = CreateChild(root, "Target");

        var built = Build(root, out _);
        var parameters = built.GetComponent<VRCAvatarDescriptor>().expressionParameters.parameters;

        var definition = System.Array.Find(parameters, p => p.name == "NDMFComponent/Index");
        Assert.NotNull(definition);
        Assert.AreEqual(VRCExpressionParameters.ValueType.Int, definition.valueType);
        Assert.IsTrue(definition.networkSynced);
        Assert.IsTrue(definition.saved);
        Assert.AreEqual(0f, definition.defaultValue);
    }

    // PARAM-10: Expression Parameters (3), the controller (0), and the guarantee (0) disagree;
    // the existing definition wins under both processing orders.
    [Test]
    public void DefaultDisagreementResolvesTheSameUnderBothOrders()
    {
        float WinningDefault(bool referencingFirst)
        {
            var root = CreateMinimalAvatar();
            var descriptor = root.GetComponent<VRCAvatarDescriptor>();
            var parameters = descriptor.expressionParameters;
            parameters.parameters = new[]
            {
                new VRCExpressionParameters.Parameter
                {
                    name = "NDMFComponent/Index",
                    valueType = VRCExpressionParameters.ValueType.Int,
                    defaultValue = 3f,
                },
            };
            var toggle = CreateChild(root, "Toggle");
            var referencing = CreateChild(root, "Referencing");
            if (referencingFirst) referencing.transform.SetSiblingIndex(0);
            toggle.AddComponent<NDMFComponent>().Target = CreateChild(root, "Target");
            referencing.AddComponent<NDMFReferencingComponent>();

            var built = Build(root, out _);
            var definition = System.Array.Find(
                built.GetComponent<VRCAvatarDescriptor>().expressionParameters.parameters,
                p => p.name == "NDMFComponent/Index");
            var result = definition.defaultValue;
            TearDown();
            return result;
        }

        Assert.AreEqual(3f, WinningDefault(false));
        Assert.AreEqual(3f, WinningDefault(true));
    }

    // PARAM-05: types resolve by priority — an existing parameter by its own type, a built-in
    // by the catalogue, an unknown name as Float.
    [Test]
    public void ConditionTypesResolveByPriority()
    {
        var root = CreateMinimalAvatar();
        var bone = CreateChild(root, "Bone");
        bone.AddComponent<VRCPhysBone>().parameter = "Bone";
        var component = CreateChild(root, "Conditions").AddComponent<NDMFConditionComponent>();
        component.Conditions = new[] { "Bone_Angle", "IsLocal", "NoSuchParameter" };

        var built = Build(root, out var errors);
        var fx = FindFxController(built);

        Assert.AreEqual(AnimatorControllerParameterType.Float,
            fx.parameters.Single(p => p.name == "Bone_Angle").type);
        Assert.AreEqual(AnimatorControllerParameterType.Bool,
            fx.parameters.Single(p => p.name == "IsLocal").type);
        Assert.AreEqual(AnimatorControllerParameterType.Float,
            fx.parameters.Single(p => p.name == "NoSuchParameter").type);
        Assert.IsFalse(errors.Any(e => e.TheError.Severity == ErrorSeverity.Error));
    }
}
```
