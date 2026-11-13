# agent-skills-vrc-ndmf-plugins

An agent skill for building VRChat avatar plugins on NDMF (Non-Destructive
Modular Framework): instructions, procedures, and references that an AI
coding assistant loads before designing or writing a plugin.

A plugin in this skill's scope is an editor-only component plus a
build-time pass that validates against `ParameterInfo`, generates and
merges animator output, and reports problems through `ErrorReport`.

## Scope

- Unity 2022 LTS, VRChat Avatars SDK (SDK3A), editor-only plugins.
- Facts are anchored to NDMF 1.15.* and Modular Avatar 1.19.*.
- Exactly two dependency cases are covered: (1) NDMF only, and
  (2) NDMF + Modular Avatar.
- Out of scope: VRChat worlds and Udon, and shader work.

## Contents

- `SKILL.md` — the entry point: scope, the NEVER list of prohibitions,
  design decisions, the core workflow, and the table of contents.
- `CHEATSHEET.md` — the plugin skeleton, the asmdef assembly tables,
  and the package.json tables.
- `references/` — one file per subject: framework, animator generation,
  reactive tracking, Modular Avatar integration, parameters, error
  reporting, Expression Parameters and Expressions Menu, manipulation
  techniques, the VRChat avatar platform, VPM packaging, Avatar
  Optimizer compatibility, debugging, anti-patterns, and the procedure
  index (P-01 to P-25).
- `skill.yaml` — the skill manifest.

## Use

Load `SKILL.md` into the coding agent before it designs or writes an
NDMF plugin; the file directs the agent to the reference each task
needs. The layout follows the agent-skills form used by
[agent-skills-vrc-udon](https://github.com/niaka3dayo/agent-skills-vrc-udon).

## License

MIT License. See `LICENSE`.

The skill's knowledge draws on existing open-source work in the VRChat
avatar tooling ecosystem. No code from them is reproduced in this
skill.
