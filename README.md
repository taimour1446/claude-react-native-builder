# claude-react-native-builder

> 🤖 A **Claude Code plugin** — an AI agent system that scaffolds and extends
> production-grade **React Native (Expo)** mobile apps on a fixed, opinionated
> architecture.

This is not a starter template and not a CLI. It is a [Claude
Code](https://claude.com/claude-code) plugin: it ships **one skill** and **four
specialized AI agents** that work together, under a self-reviewing build loop,
to produce React Native code that is consistent, fully commented, and
indistinguishable from hand-written reference code.

---

## What it does

- **Scaffolds** a new Expo app — the full 12-folder `src/` architecture, theme,
  Redux store, RTK Query, navigation, auth shell, i18n, and EAS config — ready
  for features.
- **Extends** an existing app — adds screens, API services, Redux slices,
  validation schemas, forms, hooks, and components, each following the standard.
- **Reviews** code against a strict, numbered ruleset (R01–R89).
- **Builds & deploys** with EAS Build, EAS Update, and EAS Submit.

## The locked stack

Opinionated and fixed — the plugin never asks you to pick libraries:

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

## The four agents

| Agent | Role |
| --- | --- |
| `rn-scaffolder` | Bootstraps a new app |
| `rn-feature-builder` | Builds features — gated by the reviewer on both ends |
| `rn-pattern-reviewer` | Strict standards gate (read-only) |
| `rn-build-deployer` | EAS builds, OTA updates, store submission |

## The double review gate

`rn-feature-builder` cannot write code until `rn-pattern-reviewer` has approved
its **plan**, and cannot finish until the reviewer has approved its **code**.
Failures loop back until they pass — *block-and-loop-until-pass*.

## Installation

This repository is both a Claude Code **plugin** and its own **marketplace** —
install it in two commands inside Claude Code:

```
/plugin marketplace add taimour1446/claude-react-native-builder
/plugin install react-native-app-design@claude-react-native-builder
```

- The first command registers this repo as a marketplace
  (it reads `.claude-plugin/marketplace.json`).
- The second installs the plugin from it. The `@claude-react-native-builder`
  suffix is the marketplace name — it tells Claude Code which source to install
  from.

To update later, run `/plugin marketplace update claude-react-native-builder`.

Once installed, trigger it with `/react-native-app-design`, or simply ask
Claude to build or extend a React Native app.

## How to use it

| You want to… | Do this |
| --- | --- |
| Start a new app | `/react-native-app-design`, or "build me a new React Native app" |
| Add a feature | "add a profile screen" / "add a notifications API service" |
| Review code | "review this mobile code against the standard" |
| Build / deploy | "build a preview with EAS" / "publish an OTA update" |

## Repository structure

```
.claude-plugin/plugin.json          Plugin manifest
skills/react-native-app-design/
  SKILL.md                          Entry point — triggers, stack, orchestration
  reference/                        Architecture, coding standards, patterns, …
  templates/                        Skeletons for each file type
agents/                             The four rn-* agents
```

> The plugin/skill slug remains `react-native-app-design` (it is the command
> name and is referenced internally); the repository is named
> `claude-react-native-builder` for clarity.

## License

MIT — see [LICENSE](LICENSE).
