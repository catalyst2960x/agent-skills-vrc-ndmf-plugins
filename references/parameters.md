# Parameters Reference

The VRChat avatar parameter set as a plugin sees it at build time: how
the parameters declared by every source are aggregated into one view,
and the conversions between the representations of a parameter. How
defects found against this set are reported is Error Reporting.

## Getting the build-time parameter set

- Parameters are an NDMF service: `ParameterInfo.ForContext(ctx)` (type in
  `nadena.dev.ndmf`, Editor/VRChat/ParameterIntrospection), then
  `GetParametersForObject(avatarRoot)` returns the ProvidedParameters that
  all plugins have declared for the avatar so far.
- Index them by `EffectiveName` — the name after any renaming earlier passes
  applied. If a name appears more than once, keep the first entry and do not
  treat duplicates as an error.
- A ProvidedParameter carries its type (`ParameterType`, an
  AnimatorControllerParameterType?). Registering it on a generated
  controller takes an AnimatorControllerParameter built from that name and
  type: construct it directly from the name and the type, in both
  dependency cases. Neither framework provides this conversion; a
  standalone conversion library exists
  precisely because the frameworks do not provide one, and it is not a
  dependency in either case. The conversion is a few lines — never a reason to change
  the dependency case.
- Which names a component needs: every parameter named by its conditions
  (and pre-conditions when enabled), plus feature parameters such as a
  Parameter-order selector. Collect them deduplicated, skipping empty
  names, and register each defined one on the generated controller once.

## Parameter conversions

The conversions a plugin needs between the three representations of a
parameter (the plugin implements the ones it needs itself — each is
a small type switch, and none of them justifies a dependency):

- ProvidedParameter, or a VRCExpressionParameters entry, → an
  AnimatorControllerParameter with the same name and type (above). A
  ProvidedParameter whose type is null converts as Float.
- An AnimatorControllerParameter → a VRCExpressionParameters entry, when a
  product writes Expression Parameters (for example when it must declare a
  parameter the avatar does not have yet, in case 2 via Modular
  Avatar's parameter
  component instead).
- Value type ↔ animator parameter type, in both directions: Int, Float and
  Bool map one-to-one; anything else has no animator counterpart and is a
  design error to route here.
- A controller's existing parameters, read back as expression-parameter
  entries, when a pass must reconcile what it generates with what the
  avatar's controllers already declare.
- The parameters a PhysBone or a Contact Receiver provides, which exist
  without any Expression Parameters entry; a name lookup that only reads
  declared parameters misses them, which is exactly what ParameterInfo
  aggregates.
- Filters over provided parameters that recur in validation: deduplicate
  by name (first entry wins), drop empty names, and drop entries that are
  not visible to the avatar being built.
