# Parameters Reference

The VRChat avatar parameter set as a plugin sees it at build time: how
the parameters declared by every source are aggregated into one view,
and the conversions between the representations of a parameter. How
defects found against this set are reported is Error Reporting. The
edit-time side — how the author enters the names this set later
interprets — is Expression Parameter Input.

## Getting the build-time parameter set

- Parameters are an NDMF service: `ParameterInfo.ForContext(ctx)` (type in
  `nadena.dev.ndmf`, Editor/VRChat/ParameterIntrospection), then
  `GetParametersForObject(avatarRoot)` returns the ProvidedParameters that
  all plugins have declared for the avatar so far.
- Index them by `EffectiveName` — the name after any renaming earlier passes
  applied. If a name appears more than once, keep the first entry and do not
  treat duplicates as an error. This governs the index used for type
  resolution; a plugin's own generated name colliding with an entry of a
  different type is a separate case (Expression Parameters and Expressions Menu, dependency
  case 1).
- The set holds declared parameters only. VRChat's built-in parameters
  (IsLocal, GestureLeft, IsAnimatorEnabled, and the rest of the platform
  catalogue) are supplied by VRChat at runtime; no provider declares them,
  so they never appear here. Generated output that uses them follows
  Animator Generation (Built-in parameters in generated output).
- Expand the returned entries through `ProvidedParameter.SubParameters()`
  when indexing. An entry in the PhysBones prefix namespace stands for
  the derived parameters the platform defines for it (VRChat Avatar
  Platform, Component-Generated Parameters), and only the expansion
  carries their names and types; the prefix entry itself reports no
  type. A Raycast component's Parameter value is likewise a prefix,
  standing for the derived parameters the platform defines for it
  (VRChat Avatar Platform, Raycast); a reference to the prefix itself
  names none of them.
- The set is a type source, not a gate: a name absent from it is not
  thereby defective, and no report follows from absence.
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
  (and pre-conditions when enabled), plus the parameters the product
  defines for its own features. Collect them deduplicated, skipping empty
  names, and register each defined one on the generated controller once.
- A name the plugin's own generated logic reads to drive its
  behaviour — a selector the author names, for example — is a
  stronger dependency than a name a condition references. Where
  the platform defines what happens when the name is not defined
  (a selection falls back to a fixed choice), the design decides
  one of two: the plugin guarantees the definition, declaring the
  name itself with the type and default its logic assumes and
  reusing an existing definition of that name; or the requirement
  and the fallback behaviour are stated as the product's intended
  behaviour. Silent degradation is a defect, not a third option.

## Parameter conversions

The conversions a plugin needs between the three representations of a
parameter (the plugin implements the ones it needs itself — each is
a small type switch, and none of them justifies a dependency):

- ProvidedParameter, or a VRCExpressionParameters entry, → an
  AnimatorControllerParameter with the same name and type (above). A
  ProvidedParameter whose type is null converts as Float.
- Representation conversion is not value conversion. Registering a
  parameter on a generated controller with a type different from its
  source type (the type ParameterInfo reports, or the catalogue type
  for a built-in) does not fail: VRChat converts the value at runtime
  (VRChat Avatar Platform, Parameter Type Conversion), and the
  generated conditions evaluate the converted value — a Float read
  as Int has rounded, an Int read as Bool has collapsed to
  zero/non-zero. Register the type the build-time set reports; a
  mismatch in generated output is a defect, not a conversion to rely
  on. The null-type default above is value-preserving for Int and
  Bool sources; it is not a licence to register Float where the
  source type is known.
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
