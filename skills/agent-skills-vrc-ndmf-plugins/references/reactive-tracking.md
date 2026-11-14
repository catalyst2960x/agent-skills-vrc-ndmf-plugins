# Reactive Tracking Reference

How a plugin's generated layers follow a GameObject's active state:
the build supplies the state as a Float animator parameter (the
proxy) whose value mirrors the object's own animation. A layer
cannot read a GameObject's active state directly, and no author
code runs on an uploaded avatar, so the state is obtained
indirectly: every animation curve that writes the object's
`m_IsActive` is mirrored onto a curve that writes the proxy.
States and blend trees then read the proxy instead of the object.

The scope is the input side: obtaining a tracked value a layer can
read. What a component does with the value is the product's own
design; the layer that reads the proxy is built by P-09. The
primitive is general: the binding being mirrored can be any
animated float property, and active state is the primary
application. Everything below is stated in NDMF terms only. The
mechanism is the plugin's own: a component that uses it remains a
generator of its own animator output, whose layers read the proxy.

## Components

The mechanism has four parts.

- **Registry.** A per-build store recording each tracked object's
  proxy name. Registration happens during Generating: a lookup
  that finds the object already recorded returns the existing
  proxy name, so registration is idempotent, and a new
  registration assigns a name from the provider's prefix, the
  object's name, and a sequential index in registration order.
  Registration order is the order consumers visit objects on the
  build clone, so repeated builds of the same avatar assign the
  same names. The registry is keyed by the object itself: it
  records the object, its proxy name, and its build-time default,
  and registration declares the proxy in every controller held at
  that point. It records no path and no binding — a path captured
  at registration cannot be relied on at mirroring time, for the
  reason stated under "The mirroring pass".
- **Proxy.** One Float animator parameter per tracked object,
  declared with the recorded default. A proxy is an animator
  parameter only: it is never declared in Expression Parameters,
  so it consumes no sync budget and nothing outside the build can
  write it.
- **Mirroring pass.** A single pass in Transforming, after the
  avatar's animation generation is complete, that turns the
  registry into curves (see "The mirroring pass" below).
- **Consumer.** The plugin code that registers objects and reads
  proxies in the layers it generates. A consumer declares every
  proxy its layers read on its generated controller, Float, with
  the recorded default, and reads them in one of two forms.

  *State machine form* — the layer is built by P-09, with the
  proxies as its conditions:
  1. Forward: a single transition from the idle state to the
     active state, carrying `AnimatorConditionMode.Greater`,
     threshold 0.5, on the object's proxy.
  2. Return: one transition per condition, carrying
     `AnimatorConditionMode.Less`, threshold 0.5, on that proxy
     (reversal per Animator Generation).
  3. Hierarchy activation: register the chain from the consumer's
     GameObject up to (but not including) the avatar root. The
     forward transition carries every chain proxy's Greater
     condition; the returns are one transition per chain proxy.
     An empty chain (the consumer sits on the avatar root) leaves
     the forward transition unconditional.

  *Blend tree form* — the layer is a Direct Blend Tree layer
  (Animator Generation, Direct Blend Tree layers), with the
  proxies as blend parameters:
  1. Build a Simple1D blend tree keyed on the object's proxy,
     automatic thresholds off: the reaction's inactive clip at
     threshold 0, its active clip at threshold 1.
  2. Hierarchy activation: nest Simple1D trees, one level per
     chain proxy. A branch that fails its level ends at the
     inactive clip; only the path where every level passes
     reaches the active clip. Equivalently, the active clip's
     weight is the product of the chain proxies.
  3. Intermediate values blend proportionally: a crossfade in the
     original curves crossfades the reaction, where the state
     machine form switches at 0.5.

The registry and the mirroring pass live in one of two shapes. In
a single product, the registry is per-build state shared by the
product's own passes through `BuildContext.GetState<T>()`
(Framework), and no other plugin can register. As a shared
provider, the registry sits in an extension context published by a
package other plugins take a dependency on; a consumer declares
the context where it registers its pass (P-03), reads the context
inside the pass, and calls its registration method, while the
provider plugin owns the mirroring pass. The rest of the mechanism
is identical in both shapes, and "the registry" names whichever
store the shape uses.

## Invariants

The mechanism holds only under these properties.

| # | Property | Reason |
|---|---|---|
| R1 | Registration completes before the mirroring pass runs; the contract is "register by the end of Generating" | The mirroring pass processes the recorded set once; a later registration has no pass left to mirror it |
| R2 | The mirroring pass runs in Transforming, after the avatar's animation generation is complete | The pass copies curves that other passes generate; a curve generated after it is never copied |
| R3 | The mirroring pass obtains the source binding at mirroring time, resolved from the recorded object; the registry holds no path and no binding | A captured path cannot be relied on across the span between registration and mirroring: the remapper's records are a snapshot from the context's activation, the context can reactivate within a build, and a snapshot taken after a reparenting records the moved path as the original — the captured path then exists in neither the map nor the graph. Resolving from the object at mirroring time is independent of which snapshot is in force; its cost is the liveness test on every entry |
| R4 | One proxy per object, shared by every consumer that registers the same object | Two proxies for one object diverge as soon as one consumer's registration is missed by a clip |
| R5 | The proxy is a Float parameter whose default is the object's build-time active state (1 active, 0 inactive) | Before any clip plays, the parameter must already read as the object's current state |
| R6 | Only mirrored curves write the proxy; the generating plugin's own layers never animate it | A layer writing the proxy fights the mirrored curves, and layer order decides the winner |
| R7 | Coverage is the set of clips the AnimationIndex enumerates over the VirtualControllerContext's controllers at mirroring time | The set is searched under the paths as they stand at mirroring time — the path resolved from the object, and its map translation when the map gives a different one — so a reparented object is reached under the form its clips carry. A clip whose binding names a path that is neither is not reached. Clips outside the set (for example from a generator ordered after the mirroring pass) are out of scope by construction, not by error |

Semantics that follow from the mechanism:

- An object whose active state no clip animates keeps its build-time
  state for the whole session. Absence of a curve is a constant,
  not a defect, and draws no report (Error Reporting: inapplicability
  is not a defect).
- "Active in the hierarchy" is a conjunction over the object and its
  ancestors. The mechanism supplies one proxy per object; the
  consumer composes the conjunction, as described under Components.
- The proxy is not synced by itself. Each client runs the same
  controllers and the same clips, so the mirrored value re-derives
  per client from the synced inputs that drive the original curves.

## The mirroring pass

The pass is registered in Transforming with
`.AfterPlugin("nadena.dev.modular-avatar")` in an NDMF + Modular
Avatar project — a weak constraint, ignored when Modular Avatar is
absent — and requires the AnimatorServicesContext; declare the pass
and the context requirement per P-03, and read the context as in
Animator Generation (Merging into the avatar, case 1). The
requirement the constraint serves is general: the pass runs after
every pass that generates or edits the avatar's animation clips.

The pass runs once per build over the registry; an empty registry
ends it. For each surviving entry (the registry account below),
the pass resolves the object's path from the object through the
ObjectPathRemapper as it stands at that point
(`GetVirtualPathForObject`), builds the source
binding — `EditorCurveBinding.FloatCurve(path,
typeof(GameObject), "m_IsActive")` — and queries the
AnimationIndex under it. It also queries under the path the
remapper's virtual-to-real map translates the resolved path to,
when that differs: that second form covers a snapshot that
predates a reparenting, where the resolved path is the pre-move
one and the clips carry the current one. For each clip the index
returns, the pass reads the clip's float curve through the clip's
own curve binding for the object and, when one exists, sets that
same curve on the target binding in the same clip —
`EditorCurveBinding.FloatCurve("", typeof(Animator), proxyName)`,
a curve on an Animator-type binding writes the named animator
parameter. Reading through the clip's own binding finds the curve
in whatever form the clip stores it. The copy lives in the same
clip as the original, so the proxy shares the original curve's
lifecycle exactly: the same states drive it, and Write Defaults
applies to it the same way.

The registry is keyed by the object. Registration — in a pass
segment where the AnimatorServicesContext is active — records
the GameObject and its proxy name, reads the build-time
`activeSelf` as the proxy's default, and declares the proxy in
every controller the VirtualControllerContext holds, by
assigning a fully constructed AnimatorControllerParameter into
each controller's Parameters map (Animator Generation, Merging
into the avatar, case 1) — declaration is complete at
registration because a clip that writes an undeclared parameter
does not drive it, and the object's clips can live in any
controller.

At mirroring, each entry is first tested: an entry whose object
was destroyed before mirroring (Unity's fake null) is dropped —
no copy, and no report, since an object that no longer exists
has no activation to follow. The liveness test is load-bearing:
a pass that omits it dereferences a destroyed object and the
build fails on that entry, so the mechanism's safety rests on
the test being present in every implementation. A surviving
entry's path is resolved from the object at mirroring time, so
reparenting and renaming of the same object between registration
and mirroring are followed. An entry does not transfer to a
different object: a pass that replaces a tracked object leaves
the entry pointing at the removed object, and tracking does not
continue onto the replacement.

The registry records no path because a captured path cannot be
relied on across the span between registration and mirroring.
The ObjectPathRemapper's records are a snapshot of the hierarchy
taken when the AnimatorServicesContext activates, and the
context can deactivate and reactivate within a build. A snapshot
taken after a pass has reparented a tracked object — a hierarchy
fold is the common case — records the moved path as the object's
original path: the registration-time path then exists in neither
the remapper's map nor the clips in the graph, which by then
carry the current path, and no lookup built from the captured
path reaches them. Resolving the path from the object at
mirroring time is independent of which snapshot is in force; the
lookup's second form — the map translation of the resolved
path — covers the opposite case, a snapshot that predates the
reparenting.

The mirroring itself has two implementation forms. The copy is one; the other is a generated
state machine: instead of writing into the existing
clips, the provider generates, per tracked object, a layer whose
states reproduce the states that drive the object's `m_IsActive`.
Each generated state holds a clip that writes the proxy with the
value the corresponding original state's clip gives `m_IsActive`
— the curve transplanted per state, or a constant where the
original curve is constant — and the generated layer's transitions
replicate the original transitions, conditions and timing, that
select among those states. Choose this form when the original
clips must not be edited, for example when they are shared assets
owned by another package. Its costs are one layer per tracked
object, and a structure that mirrors the original state machines,
so it is built at the same point as the copy (R2) and from the
same registry.

Two properties of the framework surface matter here. The
AnimatorServicesContext is activated per pass segment, so the pass
receives an index and a path remapper built over the controllers
as they stand at that point of the build (R2, R7) — and the
remapper is a snapshot from the context's activation, which is
why paths are resolved from objects at that point rather than
captured earlier (R3). And
`GetClipsForBinding` results can change as curves are set, so the
enumeration is materialized before curves are written into the
clips. Setting a curve with a null value removes it; the copy is
guarded on the source curve existing.

## Generalization

Tracking a different float property instead of an active state
changes only two things: the source binding is the property's
binding (path, component type, property name), and how the
recorded default is determined. Registration, naming, declaration,
and copying are unchanged.

The recorded default is the property's resting value: the value
the tracked value reads before any clip plays. Determine it under
this precedence. When the property's resting value is driven by a
parameter that a provider declares — an Expression Parameters
asset or a parameters component — the provider's declared default
governs, following the platform's own precedence of Expression
Parameter defaults over controller parameter defaults; a build-time
read of the component can reflect an edit-mode pose rather than
the runtime resting value, so it does not override a declared
default. Read the value from the component at build time only when
no provider declares the driving parameter. For `m_IsActive` the
precedence never engages: no provider declares an object's active
state, so the build-time `activeSelf` is the resting value (R5).
