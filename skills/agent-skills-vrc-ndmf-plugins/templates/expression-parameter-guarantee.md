# Template: Guaranteeing an Expression Parameters definition

## When it applies

A generated controller references a parameter whose value the
author (or another system) sets through the avatar's Expression
Parameters — for example a clip-index parameter an SDK behaviour
reads. The platform requirement is that such a parameter be
defined in Expression Parameters; a controller declaration alone
does not define it, and an undefined parameter silently behaves
as its zero value. The plugin therefore guarantees the definition
itself instead of documenting it as a setup step.

## Requirements

- Collect the names to guarantee during generation; emit the
  definitions only when at least one component actually generated
  output.
- Reuse a `ModularAvatarParameters` component already on the
  avatar root; add one only when absent. Never add a second one.
- The definition matches the semantics the feature needs: for a
  selection index, Int, synced, saved, with an explicit default
  of 0.

## Skeleton

```csharp
// [Editor: NDMF pass]
// During generation, per component that needs the guarantee:
if (!expressionParameters.Contains(parameterName))
    expressionParameters.Add(parameterName);

// After generation, only when something was generated:
if (expressionParameters.Count > 0)
{
    var parameters = ctx.AvatarRootObject.GetComponent<ModularAvatarParameters>()
                     ?? ctx.AvatarRootObject.AddComponent<ModularAvatarParameters>();
    foreach (var name in expressionParameters)
    {
        parameters.parameters.Add(new ParameterConfig
        {
            nameOrPrefix = name,
            syncType = ParameterSyncType.Int,
            localOnly = false,
            saved = true,
            defaultValue = 0f,
            hasExplicitDefaultValue = true,
        });
    }
}
```
