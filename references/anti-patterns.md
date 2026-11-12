# Anti-Patterns: Unity Habits That Break Under NDMF / Modular Avatar

Rules for code written by a Unity developer who does not yet know NDMF or
Modular Avatar (MA). Each entry names the habit the coder brings from plain
Unity work, what it breaks in an avatar build, and the rule that replaces
it. Use this file when reviewing or fixing such code; the NEVER list in
SKILL.md states the prohibitions, this file teaches the pattern behind
them so the same mistake is recognizable in a new shape.

Grounding: every rule maps to a NEVER entry, a procedure (P-nn), or the
reference section named with it, whose facts live in the other
references. Patterns 4 and 9 in particular are
drawn from real plugin code that failed in exactly these ways; they are
the most likely first mistakes, not theoretical ones.

## 1. Reaching for a runtime script
Habit: put the logic in a MonoBehaviour — Update, OnEnable, a coroutine.
Breaks: an uploaded avatar runs no user code at all; nothing observes
anything at runtime.
Rule: express every effect as build-time generation — animator parameter
conditions, MA reactive components, or a static transformation. If a
design seems to need runtime observation, the design is wrong.
(NEVER #6; SKILL.md, Trigger model)

## 2. Reading the avatar's assets as the truth
Habit: open the VRCExpressionParameters asset, the descriptor, or the FX
controller to learn what parameters and layers exist.
Breaks: parameters are renamed, added and synced by other plugins during
the build; the asset on disk is not the build-time truth, so validation
passes against names that will not exist.
Rule: ask ParameterInfo (GetParametersForObject, keyed by EffectiveName);
ask the build context, not the assets. (NEVER #2; P-05)

## 3. Editing the author's controller in place
Habit: load the existing FX AnimatorController asset and add layers to
it, the way an editor tool edits a project asset.
Breaks: it mutates the author's asset and races Modular Avatar's own
cloning and merging; in a build pass it also breaks the guarantee that
the build works on a clone and the author's assets survive.
Rule: generate a fresh controller and merge it (case 2), or clone and
manage the controller yourself (case 1). Destructive editing belongs to
editor tooling the author runs deliberately, never to a distributed
plugin's build pass. (NEVER #3, #9; P-09, P-11)

## 4. Calling a helper from a library the package does not depend on
Habit: a well-known library has exactly the conversion or utility
needed, so call it — it is right there in the reference projects.
Breaks: the call compiles only where that library happens to be
present — typically in the reference projects open on the author's own
machine — and fails in any project that does not install it. The
failure is not visible in the call itself: its signature genuinely
matches the reference it was taken from. What differs is the project
it compiles in.
Rule: every called method and type must resolve inside the package's own
assemblies plus its declared dependencies — the dependency-closure
check. If a conversion is needed and no dependency provides it, write
the few lines yourself. (P-08 step 3; P-17 step 2; SKILL.md step 7)

## 5. Taking SDK names from documentation or memory
Habit: type the field or enum name as the documentation shows it, or as
remembered.
Breaks: documentation shows display names; the C# identifier differs,
and the error only appears when the user imports the package.
Rule: verify every SDK identifier against code that compiles against the
SDK, and mark anything unverified as unverified. (NEVER #8; P-10 step 3)

## 6. Ordering passes by feel
Habit: register the pass in whatever phase seems late enough, or add the
merge component whenever the controller is ready.
Breaks: Modular Avatar consumes merge components in its own phase; a
merge component added after that is silently dropped and the generated
layers never reach the avatar, with no error.
Rule: generation for Modular Avatar runs
InPhase(BuildPhase.Generating).BeforePlugin("nadena.dev.modular-avatar").
(NEVER #4; P-03, P-11)

## 7. Cleaning up first
Habit: DestroyImmediate the source components at the start of the pass
"to avoid duplicates", then generate.
Breaks: a component destroyed before its validation runs is never
validated and never reported; the build proceeds as if it had not
been there.
Rule: process every found instance first (validate, report,
generate), then DestroyImmediate every found instance, success or
failure. Removal acts on the build clone; the author's original
avatar keeps its components either way. (NEVER #5; P-11 step 3, P-19)

## 8. Debug.Log for validation
Habit: log the problem and continue, as in a runtime tool.
Breaks: the build reports success with wrong output; the author never
sees the failure tied to the component that caused it.
Rule: report through ErrorReport with the component as context object,
at the severity the error policy assigns. (NEVER #1; P-06)

## 9. Trusting serialized data
Habit: iterate a serialized array directly and switch on a serialized
enum without a guard.
Breaks: data written by older versions or by script carries null arrays
and null elements into the pass, and out-of-range enum values throw
mid-generation. Both occur in real plugin code: a condition reversal on
an out-of-range mode value throws, and unsanitized arrays do the same.
Rule: sanitize once at the top of the pass (null array = empty, null
elements dropped, enum-driven branches guarded) and use only the
sanitized data afterwards. (NEVER #7; P-04)

## 10. Finding build-time objects by name or path
Habit: GameObject.Find, transform.Find, or a stored path string to
locate targets when generating.
Breaks: earlier passes replace objects (reactive generation, merging,
cloning); a name or path found before that no longer denotes the object
in the build, or denotes a different one.
Rule: resolve targets through references captured on the build-time
clone and through the ObjectRegistry when earlier passes may have
replaced them. (P-14, Requires)

## 11. Skipping asmdefs because it compiles at home
Habit: drop scripts into folders and let the predefined assemblies sort
it out.
Breaks: in the predefined assemblies every script sees every other
script, so undeclared references compile at home and fail in a user's
project; the Editor/player split also stops being a guarantee, so
editor-only code can leak toward the player build the user uploads.
Rule: named Runtime and Editor asmdefs, Editor asmdef with
includePlatforms = Editor and autoReferenced = false, references listed
explicitly. (P-01, "Why the asmdefs are load-bearing")

## 12. Declaring parameters without the shared budget
Habit: add synced parameters as the feature list grows, with product
names as they come.
Breaks: the avatar has one 256-bit sync budget shared by everything on
it, and parameter names are a compatibility surface across products;
collisions and budget overflow surface as build errors on the user's
avatar, not in the plugin's test scene.
Rule: check the budget against the real avatar's usage (P-05), prefix
product parameters, decide names once and keep them stable, mark
localOnly when remote players do not need the value. (P-12)

## 13. Driving one property through two routes
Habit: add a Reactive Component for a property and also generate a clip
that animates it, "to be safe".
Breaks: the two routes fight over the same binding; the result depends
on evaluation order and breaks when either side changes.
Rule: one property, one route — choose per property and record the
choice. (P-14 step 4)

## 14. No preview story
Habit: check the effect in Play mode, as with a plain Unity tool.
Breaks: the build-time avatar differs from the live scene (generation,
merging and reactive evaluation happen in the build), so Play mode
shows something the upload will not match, and the author gets no
editor feedback before building.
Rule: decide the preview route before building: inherit Modular
Avatar's preview by using a Reactive Component, write an NDMF
IRenderFilter, or state explicitly that there is no preview
(legitimate for non-visual output such as audio). (P-15)

## 15. Using a type name that matches the namespace's last segment
Habit: declare the namespace the asmdef's rootNamespace suggests
(`Foo.Editor`) and write `class MyEditor : Editor`.
Breaks: inside that namespace, `Editor` resolves to the namespace
itself, not to UnityEditor.Editor, and the package fails to compile
at the user's first import (CS0118). The same resolution failure
occurs at every other type-position use of the name, such as
`typeof(Editor)`.
Rule: fully qualify any type whose name matches one of the file's
namespace segments (UnityEditor.Editor is the common case), and
audit the whole assembly for bare uses. (P-17)

## 16. Assuming the overload that reads naturally exists
Habit: call an editor API in the shape that feels obvious, such as
`EditorGUILayout.PropertyField(prop, "Label")` with a string label.
Breaks: that overload does not exist; the string binds to a
params GUILayoutOption[] overload instead, and the import fails
(CS1503) at every call site that used the shape.
Rule: verify editor-GUI call signatures against the real UnityEditor
API before writing them; a PropertyField label is a GUIContent.
The audit in P-17 includes the call shapes a file repeats.
## 17. Converting only the live value of something animation also drives

A converter that rewrites renderer materials and component fields
and ships without touching animation curves. Why it's wrong: toggles
and outfit swaps animate exactly those references; the build looks
correct until the first such animation plays, then the original
value returns. What instead: run every reference — live and animated
— through one de-duplicating mapping. (Manipulation Techniques,
Converting shared assets in a pass.)
