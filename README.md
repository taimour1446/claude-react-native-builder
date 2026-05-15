# react-native-app-design

A Claude Code plugin for building and extending **production-grade React Native
(Expo) mobile apps** that strictly follow one fixed architecture and one strict
coding standard.

It ships **one skill** and **four specialized agents**, wired together with a
double-gated review loop so generated code is consistent, fully commented, and
indistinguishable from hand-written reference code.

## What it does

- **Scaffolds** a new Expo app — the full 12-folder `src/` architecture, theme,
  Redux store, RTK Query, navigation, auth shell, i18n, EAS config — ready for
  features.
- **Extends** an existing app — adds screens, API services, Redux slices,
  validation schemas, forms, hooks, and components, all following the standard.
- **Reviews** code against a strict, numbered ruleset (R01–R89).
- **Builds & deploys** with EAS.

## The locked stack

Opinionated and fixed — the skill never asks you to pick libraries:

| Concern | Choice |
| --- | --- |
| Framework | Expo (managed) + React Native |
| Language | TypeScript, `strict` |
| State + data | Redux Toolkit + RTK Query |
| Navigation | React Navigation (native-stack + bottom-tabs + material-top-tabs) |
| Forms | react-hook-form + Yup (schema factories) |
| Styling | `StyleSheet` + a context-based theme |
| i18n | i18next + react-i18next |
| Build / deploy | EAS Build + Update + Submit |

## The agents

| Agent | Role |
| --- | --- |
| `rn-scaffolder` | Bootstraps a new app |
| `rn-feature-builder` | Builds features — gated by the reviewer on both ends |
| `rn-pattern-reviewer` | Strict standards gate (read-only) |
| `rn-build-deployer` | EAS builds, OTA updates, store submission |

## The double review gate

`rn-feature-builder` cannot write code until `rn-pattern-reviewer` has approved
its **plan**, and cannot finish until the reviewer has approved its **code**.
Failures loop back until they pass — block-and-loop-until-pass.

## Installation

This repo is a Claude Code plugin. Install it via the Claude Code plugin system
by pointing at this repository, then the skill and agents become available in
any session. Trigger it with `/react-native-app-design`, or just ask Claude to
build or extend a React Native app.

## Structure

```
.claude-plugin/plugin.json     Plugin manifest
skills/react-native-app-design/
  SKILL.md                     Entry point — triggers, stack, orchestration
  reference/                   Architecture, coding standards, patterns, …
  templates/                   Skeletons for each file type
agents/                        The four rn-* agents
```

## License

MIT — see [LICENSE](LICENSE).
