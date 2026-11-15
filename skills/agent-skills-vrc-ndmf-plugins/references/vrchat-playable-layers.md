# VRChat Playable Layers

The playable layers of an Avatars 3.0 avatar and the Avatar
Masks that gate them: what each layer is for, how VRChat
composites the layers into one Animator, and what a mask
allows a layer to animate. Layer facts are those of the
official documentation (Playable Layers,
https://creators.vrchat.com/avatars/playable-layers). Mask
semantics are Unity's Avatar Mask behaviour (Unity Manual,
Avatar Mask); VRChat's application of masks is stated
where it adds to that behaviour.

## The playable layers

- Humanoid avatars have five playable layers, applied in
  this order: Base, Additive, Gesture, Action, FX.
  Generic avatars have three: Base, Action, FX. Later
  layers take precedence over earlier ones for the same
  property.
- Only AnimatorControllers are supported in playable
  layer slots, and the same controller must never be
  assigned to more than one slot.
- Base, Additive, Gesture and Action are for transform
  animation only: on those layers, non-transform
  animation (materials, blendshapes, GameObject and
  component toggles) is not copied to the mirror clone
  and does not survive. Everything that is not a humanoid
  transform or muscle animation belongs in FX. This is
  why generated effect output targets FX (Animator
  Generation, Merging).
- Additive is always additive blending. Its first
  layer's Avatar Mask is ignored.
- Action is blended to weight 0 by default: blend it up
  with the Playable Layer Control behaviour before the
  action, and back to 0 afterwards.
- Three further slots sit beside the playable layers:
  T-Pose, IK Pose, and Sitting. Their facts are with the
  topics that use them (VRChat Avatar Platform: IK,
  Stations).

## Avatar Masks

- A layer's Avatar Mask gates what that layer's
  animation may write. The Humanoid part of the mask
  enables or disables individual muscles. The Transform
  part enables or disables individual transforms; a
  transform the mask does not list counts as enabled. An
  enabled transform may be animated in position,
  rotation, and scale, and may take a material swap on
  its first material slot.
- A layer that carries any mask at all cannot animate a
  material swap on any slot beyond the first, and cannot
  animate the root transform of the object the Animator
  sits on. Blendshapes and other component properties
  are not affected by masks.
- When VRChat composites the playable layers into one
  Animator, each controller's first-layer Avatar Mask is
  applied to that whole controller, combined with each
  layer's own mask: a muscle or transform disabled in
  either mask is disabled. A hands-only mask of the kind
  the Gesture samples carry therefore voids an FX
  controller's transform and GameObject animation if it
  ends up on FX's first layer; generated FX output
  leaves the first layer's mask alone.
- The first FX layer's mask defaults to disabling all
  humanoid muscles while enabling GameObject animation:
  an empty first-layer mask is replaced at avatar init
  by a created default mask with that shape. Animating
  transforms in FX is not recommended; a transform
  animated in Gesture must be disabled in the FX mask
  for the Gesture animation to show through.

## Write Defaults

- Within one controller, all states must share one
  setting — a mix makes the whole controller's behaviour
  unstable. VRChat's position is that the setting be
  uniform across the avatar, and the default playable
  layers are built with it off; a product that needs it
  on must expect every playable layer replaced. One
  exception: a layer whose single state holds a Direct
  Blend Tree. That technique depends on Write Defaults
  being on, so such a layer is set on regardless of the
  avatar-wide setting, and it does not destabilize the
  surrounding off layers.
