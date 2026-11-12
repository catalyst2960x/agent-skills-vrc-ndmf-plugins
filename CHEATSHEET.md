# NDMF Plugin Cheatsheet

Quick reference. This file is the canonical home of three things: the
plugin skeleton, the assembly tables, and the package.json tables.
Everything else here is a pointer to the reference that owns the fact —
facts are maintained in one place only. The reasoning and the failure
modes are in [SKILL.md](SKILL.md); signatures are those of the
versions listed there. For task-first lookup, grep
`references/procedure-index.md` — one entry per build task, with the
steps and the traps.

## Plugin skeleton

```csharp
using nadena.dev.ndmf;
using UnityEngine;

[assembly: ExportsPlugin(typeof(MyPlugin))]

public class MyPlugin : Plugin<MyPlugin>
{
    public override string QualifiedName => "my-package-name";
    public override string DisplayName => "My Plugin";

    protected override void Configure()
    {
        InPhase(BuildPhase.Generating)
            .BeforePlugin("nadena.dev.modular-avatar")
            .BeforePlugin("com.anatawa12.avatar-optimizer")
            .Run("MyPass", Pass);
    }

    void Pass(BuildContext ctx)
    {
        var components = ctx.AvatarRootObject
            .GetComponentsInChildren<MyComponent>(true);
    }
}
```

Inside the pass, in order: sanitize the found components; validate
their parameters against ParameterInfo; generate; merge; then remove
the component instances — every instance found, including ones that
failed validation (P-19).

## Phases in order

Phase order and each phase's intended use: Framework (Phases and
ordering). Generating output for Modular Avatar goes in Generating,
before MA.

## Assemblies: asmdef and assembly-info

The asmdefs carry the Editor/player boundary, the assembly identity that
InternalsVisibleTo and versionDefines address, and the package's explicit
reference graph; P-01 in the Procedure Index states the consequences of
skipping them. One asmdef per assembly. The minimum split is Runtime / Editor; split
further only when a part needs its own references or visibility (for
example parameter providers in their own Editor assembly, or a separate
assembly for shared collections/utilities). Every asmdef also gets a
.meta, like every other file.

| Assembly | References |
| --- | --- |
| Runtime | VRC.SDKBase, VRC.SDK3A |
| Editor | Runtime, nadena.dev.ndmf, nadena.dev.ndmf.vrchat, nadena.dev.modular-avatar.core (case 2 only), VRC.SDKBase, VRC.SDK3A |

asmdef fields that matter:

| Field | Setting |
| --- | --- |
| name | Stable and unique; other asmdefs reference this name. Do not rename after release |
| includePlatforms | Editor assemblies: ["Editor"]. Runtime assemblies: empty (all platforms) |
| autoReferenced | false for Editor/plugin assemblies, so player builds never pull them in |
| rootNamespace | The product's root namespace, so new files start in it |
| versionDefines | Define a symbol when another package of a given version range is installed (NDMF's own asmdef defines its Modular Avatar integration symbol this way). This is how optional-integration code compiles in dependency case 1 yet uses case-2 APIs when present: gate the code with the symbol, never with a hard reference |

assembly-info (a file named AssemblyInfo.cs or assembly-info.cs inside
the assembly's folder) holds the assembly-level attributes that have no
other home:

- `[assembly: InternalsVisibleTo("...")]` — grants the product's own test
  assembly, and any sibling assembly of the same product, access to
  internal types. NDMF grants its unit-test assembly and its VRChat
  assembly this way. Without the grant, tests and sibling assemblies see
  only public API, and internal helpers get promoted to public just to be
  reachable — the visibility smell this file exists to prevent. The name
  in the grant must match the target assembly's asmdef name exactly.
- `[assembly: ExportsPlugin(typeof(...))]` is also an assembly attribute:
  it lives in the plugin's file or in assembly-info — pick one place and
  keep every assembly attribute there.

Tests get their own Editor asmdef referencing the product assemblies
(with the test-assemblies option enabled for that asmdef), and each
tested assembly grants InternalsVisibleTo to the test assembly's name.

Component: MonoBehaviour + IEditorOnly. Editor-only code in a Runtime type:
`#if UNITY_EDITOR`.

## package.json dependencies — the two cases

Case 1, NDMF only:

```json
"vpmDependencies": {
  "nadena.dev.ndmf": ">=1.8.0 <2.0.0-a"
}
```

Case 2, NDMF + Modular Avatar:

```json
"vpmDependencies": {
  "nadena.dev.ndmf": ">=1.8.0 <2.0.0-a",
  "nadena.dev.modular-avatar": ">=1.9.10 <2.0.0-a"
}
```

No other dependency appears in either case. Declare exactly the case the
design chose.

## Pass checklist

Run each item through its procedure; the steps and traps live there.

- [ ] P-04 — components collected (inactive included), arrays sanitized
- [ ] P-05 — parameters indexed and validated; P-07 in place first when
  the plugin itself generates the parameters being validated
- [ ] P-06 — every defect reported through ErrorReport at the policy
  severity; no Debug.Log
- [ ] P-08 / P-09 — controller parameters registered; layer and states
  built to the pattern
- [ ] P-11 (case 2) — merge component added before any source removal;
  only successful components removed
- [ ] Framework (BuildContext) — replaced objects registered with the
  ObjectRegistry when a pass clones or replaces

## Pointers to the references that own the facts

- Emitting MA components (case 2) — what a complete emission sets:
  Modular Avatar Integration (Emitting Modular Avatar components).
- SDK state behaviour attachment semantics: Animator Generation
  (Attaching SDK state behaviours).
- Output routes, including the destructive route (editor tooling only,
  never in a build pass): Animator Generation (Output routes).
- Menus and parameters — budget, collision rules, provider requirement,
  menu limits, installer and hierarchy routes: Expression Parameters and Expressions Menu.
- Property bindings for generated clips: Manipulation Techniques.
- Preview — the IRenderFilter contract and the decision order: Modular
  Avatar Integration (Preview); procedure P-15.
- Audit and delivery: the maintained checklist is P-17 (Procedure
  Index), including the dependency-closure check and the handover lists.
