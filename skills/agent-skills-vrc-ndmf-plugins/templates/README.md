# Templates

Worked sample code for mechanisms the references state as rules.
Each file is self-contained: the situation it applies to, then the
code, tagged at the top of each block with where the code runs (a
runtime component, an Editor pass or generator, an inspector, a
shader). The code was written for these samples from the
mechanism descriptions and the implementations they record,
trimmed to the mechanism; it is not compiled or verified in
Unity in this form.

- `aap-feedback-smoothing.md` — exponential and linear
  smoothing by feeding the output parameter back into its own
  blend tree.

- `aap-logic-gates.md` — AND, OR, NOT, and an arbitrary
  two-input truth table, as nested 1D blend trees over constant
  parameter clips.

- `aap-parameter-arithmetic.md` — arithmetic over animator
  parameters from constant parameter clips, 1D remap trees, and
  Direct sums.

- `animating-contact-placement.md` — clips toggling a contact
  object's activation and moving a Receiver's placement.

- `animator-material-property-driving.md` — clips driving a
  material Float, a Color per component, and a texture offset,
  plus the per-renderer instanced-property form.

- `material-contacts-animator-parameters.md` — Receiver
  configurations writing a Bool (Constant) and a Float
  (Proximity) to Animator parameters, with the controller-side
  declarations.

- `material-contacts-shader-buffers.md` — a Receiver configured
  for Write to Shaders, the shader-side buffer declaration and
  reads, and a complete ForwardBase Material Contacts shader.

- `component-member-registry.md` — a registry binding a
  reflection handle, the serialized binding name, an optional
  label, and an optional nested path per animatable component
  member.

- `toggle-transition-clips.md` — a Toggle layer with four clips
  and exit-time transition states, per-item flip offsets, and
  the empty-transition handling.

- `value-curve-families.md` — curve construction per value type
  in two families (discrete step, continuous ramp), the return
  direction mirroring the timing.

- `menu-installation-decomposition.md` — installing a generated
  menu without an installer: a Menu Item, a submenu over child
  Menu Items, or a Menu Group, by the menu's shape.

- `choose-any-state-options.md` — a Choose layer with one
  constant clip per option and AnyState equality transitions
  that never transition to self.

- `deterministic-layer-names.md` — disambiguating generated
  layer names in collection order (`Name`, `Name (2)`, …) so
  repeated builds produce the same controller.

- `expression-parameter-guarantee.md` — guaranteeing an
  Expression Parameters definition by issuing Modular Avatar
  Parameters on the avatar root.

- `generated-parameter-entries.md` — expanding a component's
  base name into its generated parameters, including the
  VRCRaycast prefix (`_Hit` / `_Ratio` / `_Distance`).

- `isanimatorenabled-return-transition.md` — a settle
  transition on `IsAnimatorEnabled` IfNot so a layer returns to
  idle on the last frame before the animator is disabled.

- `layer-weight-writeback.md` — setting a generated layer's
  weight by writing the layers array back into the controller.

- `nondestructive-parameter-assignment.md` — recording a
  parameter name for another component as data, applied by a
  Resolving pass to the build copy only, into a field that is
  still empty.

- `object-keyed-tracking-registry.md` — tracking a GameObject's
  active state through a proxy parameter: an object-keyed
  registry, and a late pass that resolves paths from the objects
  at copy time.

- `parameter-driver-arithmetic.md` — arithmetic on Expression
  Parameters with the Avatar Parameter Driver instead of blend
  trees: Set as constant assignment, Add for constants, Copy
  with Convert Range as multiplication and division by a
  constant and as affine forms, and Random; executed once on
  state entry, with the two-parameter operations stated as
  having no driver form.

- `parameter-type-resolution.md` — resolving a condition
  parameter's type: the build-time parameter set, then the
  built-in catalogue, then Float; the name is never validated.

- `prioritized-branch-state-machine.md` — branch priority built
  from entry-transition registration order and accumulated exit
  conditions.

- `severity-assignment.md` — the report helpers behind the
  severity policy: Information for an unset input, NonFatal for
  a set but invalid value and for contention, no Error cases.
