# agent-skills-vrc-ndmf-plugins

An agent skill for building VRChat avatar plugins on NDMF (Non-Destructive
Modular Framework): instructions, procedures, and references that an AI
coding assistant loads before designing or writing a plugin.

A plugin in this skill's scope is an editor-only component plus a
build-time pass that validates against `ParameterInfo`, generates and
merges animator output, and reports problems through `ErrorReport`.

## Install

With the skills CLI:

```bash
npx skills add catalyst2960x/agent-skills-vrc-ndmf-plugins
```

Pin a release by appending its tag:

```bash
npx skills add catalyst2960x/agent-skills-vrc-ndmf-plugins@v0.2
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

## Scope

- Unity 2022 LTS, VRChat Avatars SDK (SDK3A), editor-only plugins.
- Facts are anchored to NDMF 1.15.* and Modular Avatar 1.19.*.
- Exactly two dependency cases are covered: (1) NDMF only, and
  (2) NDMF + Modular Avatar.
- Out of scope: VRChat worlds and Udon, and shader work.

## Contents

- `skills/agent-skills-vrc-ndmf-plugins/SKILL.md` — the entry point:
  scope, the NEVER list of prohibitions, design decisions, the core
  workflow, and the table of contents.
- `skills/agent-skills-vrc-ndmf-plugins/CHEATSHEET.md` — the plugin
  skeleton, the asmdef assembly tables, and the package.json tables.
- `skills/agent-skills-vrc-ndmf-plugins/references/` — one file per
  subject: framework, animator generation, reactive tracking, Modular Avatar integration, parameters, expression
  parameter input, inspectors and drawers, error reporting, Expression Parameters and Expressions Menu, manipulation
  techniques, the VRChat avatar platform, VPM packaging, Avatar
  Optimizer compatibility, debugging, anti-patterns, and the procedure
  index (P-01 to P-25).
- `skills/agent-skills-vrc-ndmf-plugins/skill.yaml` — the skill
  manifest.
- `.claude-plugin/` — the Claude Code plugin and marketplace
  manifests.

## Use

Load `skills/agent-skills-vrc-ndmf-plugins/SKILL.md` into the coding
agent before it designs or writes an NDMF plugin; the file directs
the agent to the reference each task needs. The layout follows the
agent-skills form used by
[agent-skills-vrc-udon](https://github.com/niaka3dayo/agent-skills-vrc-udon).

## License

MIT License. See `LICENSE`.

The skill's knowledge draws on existing open-source work in the VRChat
avatar tooling ecosystem. No code from them is reproduced in this
skill.
