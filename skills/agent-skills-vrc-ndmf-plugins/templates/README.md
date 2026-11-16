# Templates

Worked sample code for mechanisms the references state as rules.
Each file is self-contained: the situation it applies to, then the
code, tagged at the top of each block with where the code runs (a
runtime component, an Editor pass or generator, an inspector, a
shader). The code was written for these samples from the
mechanism descriptions and the implementations they record,
trimmed to the mechanism; it is not compiled or verified in
Unity in this form. Where a mechanism has been verified in
Unity — in an adapted form, or in a product that uses it —
the verification covers the mechanism, not the printed
sample.

| File | Content |
| --- | --- |
| [aap-feedback-smoothing.md](aap-feedback-smoothing.md) | — exponential and linear smoothing by feeding the output parameter back into its own blend tree. |
| [aap-logic-gates.md](aap-logic-gates.md) | AND, OR, NOT, and an arbitrary two-input truth table, as nested 1D blend trees over constant parameter clips. |
| [aap-parameter-arithmetic.md](aap-parameter-arithmetic.md) | Arithmetic over animator parameters from constant parameter clips, 1D remap trees, and Direct sums. |
| [animating-contact-placement.md](animating-contact-placement.md) | Clips toggling a contact object's activation and moving a Receiver's placement. |
| [animator-material-property-driving.md](animator-material-property-driving.md) | Clips driving a material Float, a Color per component, and a texture offset, plus the per-renderer instanced-property form. |
| [material-contacts-animator-parameters.md](material-contacts-animator-parameters.md) | Receiver configurations writing a Bool (Constant) and a Float (Proximity) to Animator parameters, with the controller-side declarations. |
| [material-contacts-shader-buffers.md](material-contacts-shader-buffers.md) | A Receiver configured for Write to Shaders, the shader-side buffer declaration and reads, and a complete ForwardBase Material Contacts shader. |
| [component-member-registry.md](component-member-registry.md) | A registry binding a reflection handle, the serialized binding name, an optional label, and an optional nested path per animatable component member. |
| [toggle-transition-clips.md](toggle-transition-clips.md) | A Toggle layer with four clips and exit-time transition states, per-item flip offsets, and the empty-transition handling. |
| [value-curve-families.md](value-curve-families.md) | Curve construction per value type in two families (discrete step, continuous ramp), the return direction mirroring the timing. |
| [menu-installation-decomposition.md](menu-installation-decomposition.md) | Installing a generated menu by decomposition: a Menu Item, a submenu over child Menu Items, or a Menu Group, by the menu's shape, with the installer that makes Modular Avatar visit them. |
| [display-only-slider.md](display-only-slider.md) | A Radial Puppet wheel used as a parameter meter: the displayed name declared as a local-only expression parameter, a continuous writer (the deriving platform component, or a copy layer) that makes menu writes not persist, and the installer that gets the item installed. |
| [choose-any-state-options.md](choose-any-state-options.md) | A Choose layer with one constant clip per option and AnyState equality transitions that never transition to self. |
| [deterministic-layer-names.md](deterministic-layer-names.md) | Disambiguating generated layer names in collection order (`Name`, `Name (2)`, …) so repeated builds produce the same controller. |
| [expression-parameter-guarantee.md](expression-parameter-guarantee.md) | Guaranteeing an Expression Parameters definition by issuing Modular Avatar Parameters on the avatar root. |
| [generated-parameter-entries.md](generated-parameter-entries.md) | Expanding a component's base name into its generated parameters, including the VRCRaycast prefix (`_Hit` / `_Ratio` / `_Distance`). |
| [isanimatorenabled-return-transition.md](isanimatorenabled-return-transition.md) | A settle transition on `IsAnimatorEnabled` IfNot so a layer returns to idle on the last frame before the animator is disabled. |
| [layer-weight-writeback.md](layer-weight-writeback.md) | Setting a generated layer's weight by writing the layers array back into the controller. |
| [nondestructive-parameter-assignment.md](nondestructive-parameter-assignment.md) | Recording a parameter name for another component as data, applied by a Resolving pass to the build copy only, into a field that is still empty. |
| [object-keyed-tracking-registry.md](object-keyed-tracking-registry.md) | Tracking a GameObject's active state through a proxy parameter: an object-keyed registry, and a late pass that resolves paths from the objects at copy time. |
| [parameter-driver-arithmetic.md](parameter-driver-arithmetic.md) | Arithmetic on Expression Parameters with the Avatar Parameter Driver instead of blend trees: Set as constant assignment, Add for constants, Copy with Convert Range as multiplication and division by a constant and as affine forms, and Random; executed once on state entry, with the two-parameter operations stated as having no driver form. |
| [parameter-type-resolution.md](parameter-type-resolution.md) | Resolving a condition parameter's type: the build-time parameter set, then the built-in catalogue, then Float; the name is never validated. |
| [prioritized-branch-state-machine.md](prioritized-branch-state-machine.md) | Branch priority built from entry-transition registration order and accumulated exit conditions. |
| [severity-assignment.md](severity-assignment.md) | The report helpers behind the severity policy: Information for an unset input, NonFatal for a set but invalid value and for contention, no Error cases. |
