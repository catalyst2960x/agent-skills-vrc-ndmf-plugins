# Avatar Optimizer Compatibility Reference

What a plugin must do so that Avatar Optimizer (AAO,
`com.anatawa12.avatar-optimizer`) does not warn about, or mishandle,
its components in a production avatar build. Facts are those of
AAO's official developer documentation, version 1.9.22; the document
is named under Source at the end. AAO's assumptions may change, so
re-verify against the same document when AAO major versions move.

Contents:

- Why AAO warns about unknown components
- Route 1: remove the component before AAO runs (the default)
- Route 2: register the component with AAO's API
- Route 3: Asset Description (do-nothing components only)

## Why AAO warns about unknown components

- AAO runs near the end of the build (NDMF Optimizing phase) and
  garbage-collects components and objects: it keeps what is used
  and removes what is not. To do that safely it must know every
  component present on the avatar at that point.
- A component AAO does not know triggers an unknown-component
  warning. AAO assumes an unknown component may become active at
  runtime and depends on every component it references; when that
  assumption is wrong, the build result can be wrong as well.
- A build-time-only component that is still on the avatar when AAO
  processes it is therefore a defect in the plugin, not in AAO.

## Route 1: remove the component before AAO runs (the default)

For a component that only works at build or edit time, process the
avatar and remove the component before AAO, using DestroyImmediate.
AAO's documentation calls this the best option in most cases, and
AAO uses it internally for most of its own components.

- With NDMF: remove the component in a phase before Optimizing, or,
  when removing it inside the Optimizing phase, declare
  `BeforePlugin("com.anatawa12.avatar-optimizer")`. AAO's developer
  documentation says to declare the BeforePlugin constraint even
  when the default callback order already places the pass before
  AAO.
- Without NDMF: remove it in an IVRCSDKPreprocessAvatarCallback
  whose callbackOrder is below -1025 (the NDMF optimize hook runs
  at -1025; the SDK's own RemoveAvatarEditorOnly runs after it,
  later than AAO's processing).
- Remove EVERY instance the pass found, including instances that
  failed validation and instances on a path where nothing was
  generated. An early return placed before the removal loop leaks
  exactly those instances to AAO.

## Route 2: register the component with AAO's API

For a component that must remain on the avatar after AAO runs
(runtime components, or components a later pass still needs):

- Add an asmdef reference to
  `com.anatawa12.avatar-optimizer.api.editor`. To avoid a hard
  dependency on AAO, gate the code with a version define for the
  range [1.6, 2.0) — the API did not exist before 1.6.0 and may
  break compatibility at 2.0.0; use a tighter lower bound (for
  example [1.7, 2.0)) when a newer API is used.
- Define a `ComponentInformation<T>` subclass marked
  `[ComponentInformation(typeof(YourComponent))]`, inside
  `#if AVATAR_OPTIMIZER && UNITY_EDITOR`.
- Implement CollectMutations (the properties the component may
  change) and CollectDependency (the component's build-time and
  runtime dependencies), following each method's xmldoc.

## Route 3: Asset Description (do-nothing components only)

- Since AAO 1.7.0, a component that does nothing at runtime and
  nothing at build time (a pure data holder) can instead be
  registered through an Asset Description as safe to remove.
- This route is NOT recommended for a component whose tool performs
  any build-time or runtime processing: if the registration or the
  execution order is wrong, AAO may delete the component before
  the tool runs and the tool silently stops working.

Source: Avatar Optimizer developer documentation ("Make your
components compatible with AAO").

## See Also

- `references/framework.md` — phases and pass ordering.
- `references/procedure-index.md` — P-19 runs this reference as a
  procedure.
