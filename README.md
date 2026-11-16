# agent-skills-vrc-ndmf-plugins

A collection of agent skills for building VRChat avatar plugins on
NDMF (Non-Destructive Modular Framework): instructions, procedures,
and references that an AI coding assistant loads before designing
or writing a plugin, and before writing or extending its tests.

A plugin in these skills' scope is an editor-only component plus a
build-time pass that validates against `ParameterInfo`, generates and
merges animator output, and reports problems through `ErrorReport`.

## Install

With the skills CLI:

```bash
npx skills add catalyst2960x/agent-skills-vrc-ndmf-plugins
```

Pin a release by appending its tag:

```bash
npx skills add catalyst2960x/agent-skills-vrc-ndmf-plugins@v0.4
```

As a Claude Code plugin:

```bash
claude plugin marketplace add catalyst2960x/agent-skills-vrc-ndmf-plugins
claude plugin install agent-skills-vrc-ndmf-plugins@agent-skills-vrc-ndmf-plugins
```

With git:

```bash
git clone https://github.com/catalyst2960x/agent-skills-vrc-ndmf-plugins.git
```

## Skills

### agent-skills-vrc-ndmf-plugins

Building NDMF plugins: passes that generate animator controllers,
parameters and menus, and package the result for VPM.

| Area | Content |
|------|---------|
| **Framework** | Pass phases and ordering, build contexts, extension contexts, the two dependency cases (NDMF only, NDMF + Modular Avatar) |
| **Animator generation** | Controllers, layers, transitions and blend trees generated in code, state behaviours, merging into the avatar's playable layers |
| **Reactive tracking** | Following a GameObject's active state from generated layers through a mirrored proxy parameter |
| **Parameters** | Parameter type resolution, supplying and declaring parameters, expression parameter input with suggestions |
| **Menus** | Building Expressions Menus and menu items through Modular Avatar components |
| **Inspectors and drawers** | IMGUI inspectors, shared presentation across plugins, localized labels |
| **Error reporting** | The severity policy: a plugin never blocks the build; Information, NonFatal, and no Error |
| **Platform** | The VRChat avatar platform: playable layers, VRC Contacts and Material Contacts, Avatar Optimizer compatibility |
| **Packaging** | The VPM package layout and distribution |
| **Templates** | 24 worked samples of the mechanisms the references state as rules |

### agent-skills-vrc-ndmf-unit-tests

Unit testing the plugins the first skill builds: NUnit editor tests
that assert on generated animator controllers, parameters, and
build reports.

| Area | Content |
|------|---------|
| **Test base** | Tracked cleanup, the minimal avatar fixture, the two build entries, error capture, FX inspection |
| **Case catalogue** | 70 cases in 14 categories: base verification, component processing, generated output, parameter supply, error reporting, ordering and interoperation, non-destructiveness, preview, platform variants, mesh and hierarchy, menus, editor operations, pure logic |
| **Procedures** | T-01 to T-10: setting up, selecting and writing cases, verifying a case, regression, running the suite in Unity, triage, and close-out |
| **Failure triage** | Reading a failing case: base first, then the pass and pipeline bisection, then product against dependency |
| **Templates** | 17 worked samples, one per case area, with document-local stand-ins whose contracts are stated |

## Scope

- Unity 2022 LTS, VRChat Avatars SDK (SDK3A), editor-only plugins.
- Facts are anchored to NDMF 1.15.* and Modular Avatar 1.19.*.
- Exactly two dependency cases are covered: (1) NDMF only, and
  (2) NDMF + Modular Avatar.
- Out of scope: VRChat worlds and Udon, and shader work.

## Use

Load `skills/agent-skills-vrc-ndmf-plugins/SKILL.md` into the coding
agent before it designs or writes an NDMF plugin; the file directs
the agent to the reference each task needs. Load the unit-tests
skill before writing or extending a plugin's tests. The layout follows the
agent-skills form used by
[agent-skills-vrc-udon](https://github.com/niaka3dayo/agent-skills-vrc-udon).

## License

MIT License. See `LICENSE`.

The skill's knowledge draws on existing open-source work in the VRChat
avatar tooling ecosystem. No code from them is reproduced in this
skill.
