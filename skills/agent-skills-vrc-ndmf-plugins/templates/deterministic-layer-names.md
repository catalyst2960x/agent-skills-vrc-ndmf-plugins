# Template: Deterministic generated layer names

## When it applies

A pass adds one layer per component to a shared generated
controller, named after the component's GameObject. Two
components on same-named GameObjects would produce the same
layer name. Builds must be reproducible: the disambiguation is a
deterministic function of the collection order, never of
dictionary or hash iteration order, so repeated builds of the
same avatar produce the same controller.

## Requirements

- The base name is used as-is when free. Collisions take the
  suffixes ` (2)`, ` (3)`, … in the order components are
  collected.
- The used-name set is per build and per generated controller,
  shared by every layer the pass adds to that controller.
- Clip names derived from the layer name inherit the
  disambiguation, so a layer and its clips stay associated.

## Skeleton

```csharp
// [Editor: NDMF pass]
static string LayerName(string baseName, HashSet<string> usedLayerNames)
{
    if (usedLayerNames.Add(baseName)) return baseName;
    for (var i = 2; ; i++)
    {
        var candidate = baseName + " (" + i + ")";
        if (usedLayerNames.Add(candidate)) return candidate;
    }
}

// Use, per component, in collection order:
var layerName = LayerName("NDMFComponent [" + component.gameObject.name + "]", usedLayerNames);
controller.AddLayer(layerName);
```
