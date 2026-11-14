# Debugging Reference

How to find where a build goes wrong. NDMF's and Modular Avatar's
documentation name the tools below but give no triage procedure;
this file is the procedure-shaped summary; tool facts are those of
the pinned NDMF version (SKILL.md, Sources). Diagnosis happens in the reader's Unity Editor;
anything that needs it stays unconfirmed until run there (P-20).

## Inspect a finished manual build

`AvatarProcessor.ManualProcessAvatar` runs the full pipeline and
leaves the generated assets on disk; `AvatarProcessor.ProcessAvatar`
runs the same pipeline against a temporary container that is
deleted afterwards. For "the output is wrong, show me the output",
run the manual build and inspect the persisted controllers, clips
and the avatar clone. The ObjectRegistry (Framework, BuildContext)
traces a generated object back to the author's original when an
error or an unexpected replacement needs provenance.

## Step through the pipeline: the single-step debugger

NDMF 1.14.1 and later ship a single-step build debugger (Tools >
NDM Framework > Debug Tools > Single-step build). It decomposes
the build into one step per pass execution and per extension-context
activation, and runs them against a clone of the target avatar, so
the intermediate hierarchy and assets can be inspected after each
step. Use it when the question is WHEN in the pipeline a defect
appears — before or after another plugin's pass, or while a
particular extension context is active — rather than what the
final output is.

Two behaviours to know before relying on it:

- Step Back is not an undo. Passes are not guaranteed reversible,
  so stepping back discards the clone and replays every step from
  the start; a pass with side effects outside the avatar (file
  writes, shared state) runs those effects again.
- Stop and Keep Clone leaves the clone in the scene at the step it
  reached, for inspection after the session ends. Closing the
  window discards the session; the window offers to resume if the
  pass list still maps the recorded step uniquely.

## Ordering problems

When a pass behaves differently in a VRChat build or in Play mode
than in a plain NDMF build, check the split VRCSDK pipeline first
(Framework, VRChat build context): another preprocessor in the
chain (VRCFury is the common case) may run between the Transforming
and Optimizing phases, and Play mode goes through the same chain.

## Route the output of a diagnosis

While diagnosing a build, the two output channels have different
jobs. The build report is a set of claims about the author's
configuration: every entry spends the reader's attention, and a
report that lists things that are not problems stops being read,
so it carries only the defects the product actually claims (Error
Reporting). Diagnostic detail — traces of what a pass saw and
did, and records that claim nothing about the configuration —
goes to the Unity console log instead. This routing is a
diagnosis-time practice for development, not a reporting policy
for the shipped product; it is also the boundary of the Debug.Log
prohibition (SKILL.md NEVER #1), which forbids the log as the
channel for reporting a build problem to the author, not as a
diagnostic trace while developing.
