# Template: Object-keyed tracking registry

## When it applies

A generated layer must follow the active state of GameObjects the
author's hierarchy controls (the object itself and its ancestors),
where other build systems (merge tools, optimizers) may move those
objects to different paths during the build. The follow mechanism
is: one Float proxy parameter per object, and a late pass that
copies the object's `m_IsActive` animation curves onto the proxy.

## Requirements

- The registry is keyed by the object itself
  (`Dictionary<GameObject, string>`), never by path or binding. A
  path recorded at registration time may not exist at copy time:
  the path remapper's records are a snapshot, and an object moved
  by a merge after the snapshot is absent from both the map and
  the graph under its registration-time path.
- Registration records only the object, the proxy name, and the
  default (the object's active state at build time). It does not
  record a path or a binding.
- At registration, the proxy is declared on every virtual
  controller the context holds, so merges keep it. The plugin's
  own generated controller declares it as part of its
  construction, like every parameter its transitions reference.
- Registration also records, per generated component, the
  component's object, its source's object where the design has
  one, and the generated layer's name. Chain membership is
  completed at the head of the tracking pass: the chains are
  recomputed from the hierarchy as it stands there,
  unregistered members are registered through the same route,
  and the layer's chain-derived transitions are rebuilt from
  the recomputed chains. A member that left a chain keeps its
  proxy but leaves the conditions. Reparents in the
  Optimizing phase are out of scope.
- The tracking pass runs after the merging systems. For each
  registered object it resolves the object's path at copy time
  from the remapper, also consults the remapper's path translation
  for that path, reads the source `m_IsActive` curve through each
  clip's own stored bindings, and writes the curve onto the
  proxy's Animator parameter binding in the same clip.
- A destroyed object's entry is skipped by a null comparison, with
  no report. The pass is idempotent.

## Skeleton

```csharp
// [Editor: NDMF pass]
class TrackingState
{
    public readonly Dictionary<GameObject, string> Proxies = new Dictionary<GameObject, string>();
    public readonly List<ConsumerRecord> Consumers = new List<ConsumerRecord>();
    public int NextIndex;
}

// Per generated component: the objects its chains are computed
// from, and the layer it generated — recorded at registration,
// consumed by the chain completion in the tracking pass.
class ConsumerRecord
{
    public GameObject Component;
    public GameObject Source;
    public string LayerName;
}

// Generating: register one proxy per object in a chain (the target
// object and its ancestors up to, not including, the avatar root).
List<KeyValuePair<string, float>> RegisterChain(
    List<GameObject> chain, TrackingState state, AnimatorServicesContext services)
{
    var proxies = new List<KeyValuePair<string, float>>();
    foreach (var obj in chain)
    {
        if (!state.Proxies.TryGetValue(obj, out var proxy))
        {
            proxy = "Example/ActiveSelfProxy/" + obj.name + "##" + state.NextIndex++;
            state.Proxies[obj] = proxy;
            foreach (var virtualController in services.ControllerContext.GetAllControllers())
            {
                virtualController.Parameters = virtualController.Parameters.SetItem(proxy,
                    new AnimatorControllerParameter
                    {
                        name = proxy,
                        type = AnimatorControllerParameterType.Float,
                        defaultFloat = obj.activeSelf ? 1f : 0f,
                    });
            }
        }
        proxies.Add(new KeyValuePair<string, float>(proxy, obj.activeSelf ? 1f : 0f));
    }
    return proxies;
}

// Transforming, after the merging plugins, inside
// AnimatorServicesContext: copy each object's m_IsActive curves
// onto its proxy.
void TrackingPass(BuildContext ctx)
{
    var state = ctx.GetState<TrackingState>();
    var services = ctx.Extension<AnimatorServicesContext>();

    // Chain completion first, before any curve is copied: the
    // chains are recomputed from the hierarchy as it stands,
    // unregistered members are registered, and each layer's
    // chain-derived transitions are rebuilt from them.
    foreach (var consumer in state.Consumers)
        CompleteChains(consumer, state, services);
    if (state.Proxies.Count == 0) return;

    var index = services.AnimationIndex;
    var remapper = services.ObjectPathRemapper;
    var realPaths = remapper.GetVirtualToRealPathMap();

    foreach (var pair in state.Proxies)
    {
        var obj = pair.Key;
        if (obj == null) continue; // destroyed during the build

        var virtualPath = remapper.GetVirtualPathForObject(obj);
        string realPath = null;
        if (realPaths.TryGetValue(virtualPath, out var mapped))
            realPath = mapped;

        var source = EditorCurveBinding.FloatCurve(virtualPath, typeof(GameObject), "m_IsActive");
        var parameterBinding = EditorCurveBinding.FloatCurve("", typeof(Animator), pair.Value);
        var clips = new HashSet<VirtualClip>(index.GetClipsForBinding(source));
        if (!string.IsNullOrEmpty(realPath) && realPath != virtualPath)
        {
            var realBinding = EditorCurveBinding.FloatCurve(realPath, source.type, source.propertyName);
            clips.UnionWith(index.GetClipsForBinding(realBinding));
        }

        foreach (var clip in clips)
        {
            var curve = FindSourceCurve(clip, virtualPath, realPath, source);
            if (curve != null)
                clip.SetFloatCurve(parameterBinding, curve);
        }
    }
}

// The completion for one consumer: register whatever its
// current chains add, then rebuild the layer's chain-derived
// transitions — the forward transition carries every own-chain
// proxy, the returns are one per own-chain proxy, and the
// source-gated transition, where the design has one, carries
// every source-chain proxy. Transitions that do not derive
// from the chains stay as generated.
void CompleteChains(ConsumerRecord consumer, TrackingState state, AnimatorServicesContext services)
{
    var own = RegisterChain(ChainOf(consumer.Component), state, services);
    var source = consumer.Source != null
        ? RegisterChain(ChainOf(consumer.Source), state, services)
        : null;
    RebuildChainTransitions(consumer.LayerName, own, source);
}

static AnimationCurve FindSourceCurve(VirtualClip clip, string path, string altPath, EditorCurveBinding source)
{
    foreach (var binding in clip.GetFloatCurveBindings())
    {
        if (binding.isPPtrCurve) continue;
        if (binding.type != source.type || binding.propertyName != source.propertyName) continue;
        if (binding.path != path && binding.path != altPath) continue;
        return clip.GetFloatCurve(binding);
    }
    return null;
}
```
