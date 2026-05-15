---
name: claude-react-native-builder
description: >-
  Build and extend production-grade React Native (Expo) mobile apps that strictly
  follow a fixed architecture and coding standard. Use whenever the user asks to
  create a new mobile app, scaffold a React Native / Expo project, add a screen,
  API service, Redux slice, form, or component to a mobile app, or build / deploy
  a mobile app with EAS. Enforces a 12-folder src/ architecture, Redux Toolkit +
  RTK Query, React Navigation, react-hook-form + Yup, a context-based theme, and a
  mandatory commenting standard via a double-gated review loop.
---

# React Native App Design

This skill builds and extends **production-grade React Native (Expo) apps** that
follow ONE fixed architecture and ONE strict coding standard. It does not invent
new patterns and it does not ask the user to choose a stack — the stack is locked
(see "Locked Stack" below). Every file produced must be indistinguishable from
hand-written reference code that already follows these rules.

The skill is **self-contained**: every rule needed to build, extend, review, and
deploy an app lives inside this skill's files. Nothing depends on any external
template repository.

---

## When this skill applies

Use this skill when the user wants to:

- **Scaffold** a new React Native / Expo app from scratch.
- **Extend** an existing app of this architecture — add a screen, API service,
  Redux slice, validation schema, form, hook, or component.
- **Review** mobile code against the standard.
- **Build / deploy** the app with EAS (builds, OTA updates, store submission).

If the user's project is NOT a React Native / Expo app, this skill does not apply.

---

## Locked Stack — never deviate

These choices are FINAL. Never substitute a library, never ask the user to pick.

| Concern | Locked choice |
| --- | --- |
| Framework | Expo (managed workflow) + React Native |
| Language | TypeScript, `strict: true` |
| State + data | Redux Toolkit + RTK Query (NOT Zustand, NOT plain Context for data) |
| Navigation | React Navigation (native-stack + bottom-tabs + material-top-tabs). NOT Expo Router |
| Forms | react-hook-form + `@hookform/resolvers` |
| Validation | Yup — schema factory functions that accept the i18n `t()` function |
| Styling | React Native `StyleSheet` + a context-based theme. NOT NativeWind, NOT styled-components |
| i18n | i18next + react-i18next |
| Secure storage | expo-secure-store (tokens) + AsyncStorage (non-sensitive prefs) |
| Notifications | expo-notifications |
| OTA updates | expo-updates |
| Build / deploy | EAS Build + EAS Update + EAS Submit |
| E2E testing | Maestro (YAML flows) |
| Indentation | 4 spaces, everywhere |

For exact version policy (latest-first with a known-good fallback), read
`reference/versions.md`.

---

## The reference files — read before acting

This skill's detail is split across files so only what is needed gets loaded.
Read the relevant file(s) before doing the matching work:

| File | Read it when you need… |
| --- | --- |
| `reference/architecture.md` | The 12-folder `src/` layout and what belongs in each folder |
| `reference/coding-standards.md` | THE strict ruleset (R01–R89) — the reviewer's exact rubric. Read for ANY code work |
| `reference/conventions.md` | Fine-grained conventions — barrels, exports, constants, theme, typography, i18n, errors, EAS, git. Read for ANY code work |
| `reference/patterns.md` | Signature patterns — baseQuery, navigationRef, GlobalToast, two-hook forms, navigators, error helpers, performance |
| `reference/data-patterns.md` | FlatList config, pagination, paginated slices, RTK Query cache tags / transformResponse / merge / idempotency. Read for any list or data work |
| `reference/lifecycle-hooks.md` | Non-form custom hooks — notifications, OTA updates, polling, focus-refetch, device data |
| `reference/utils-and-types.md` | The `src/utils/` catalog and the `src/types/` conventions |
| `reference/testing.md` | Running the app on a simulator/emulator and Maestro E2E flows. Read for any run/test work |
| `reference/versions.md` | Version policy and the known-good baseline |
| `reference/scaffold-checklist.md` | The exact ordered steps + file manifest for a new app |
| `templates/*.md` | The skeleton for the file type being created (component, screen, list-screen, api-service, slice, validation-schema, form-hooks, constants, types, lifecycle-hook, maestro-flow) |

`reference/coding-standards.md` and `reference/conventions.md` are mandatory
reading for every code-producing task. `coding-standards.md` is the single source
of truth the reviewer enforces; `conventions.md` carries the fine detail.
For list/data work also read `data-patterns.md`; for non-form hooks read
`lifecycle-hooks.md`; for utils/types work read `utils-and-types.md`.

---

## The agents

This skill delegates real work to five specialized subagents. The skill itself is
the **orchestrator** (the planner): it interprets the request, picks the agent,
and enforces the review loop. Delegate via the Agent tool.

| Agent | Role | Use it to… |
| --- | --- | --- |
| `rn-scaffolder` | Executor | Bootstrap a brand-new app: create the Expo project, build the 12-folder `src/`, wire `App.tsx`, the store, `baseQuery`, the theme, navigation, the auth shell, configure `app.config.ts` / `eas.json` / i18n |
| `rn-feature-builder` | Executor | Build or change a feature — a vertical slice spanning screen + API service + slice + validation + form-hooks + components, plus a Maestro flow for it |
| `rn-pattern-reviewer` | Reviewer (read-only) | Validate a PLAN or written CODE against `reference/coding-standards.md`. Emits PASS or FAIL with specific findings |
| `rn-runner` | Verifier | Build and launch the app on a simulator/emulator, capture logs, detect crashes, run Maestro E2E flows, capture screenshots, report pass/fail |
| `rn-build-deployer` | Lifecycle | Run EAS — builds, OTA updates, credentials, store submission |

---

## Orchestration

Decide what the user wants, then follow the matching flow.

### Flow A — Scaffold a new app

1. Ask the user for the required inputs — these CANNOT be guessed:
   - App display name
   - Slug (kebab-case)
   - **iOS bundle identifier** and **Android package name** — ask for BOTH,
     separately. They are usually the same reverse-DNS string; suggest one
     value as the default for both but let the user set each independently.
   - Target directory
   - **Expo account** — ask whether to link an Expo account now (for EAS) or
     defer it. The skill cannot create an account (signup is interactive); if
     the user wants one, point them to run `eas login` / sign up at expo.dev.
   See `reference/scaffold-checklist.md` → "Inputs required from the user".
2. Read `reference/scaffold-checklist.md` and `reference/versions.md`.
3. Delegate to **`rn-scaffolder`** with the app name, slug, iOS bundle
   identifier, Android package name, target directory, and the Expo-account
   choice.
4. When it returns, delegate to **`rn-pattern-reviewer`** to audit the generated
   shell against `reference/coding-standards.md`.
5. If the reviewer returns FAIL, send the findings back to `rn-scaffolder` to fix,
   then re-review. Repeat until PASS (see "The review loop").
6. Report the result: project location, how to run it, what was wired.

### Flow B — Build or change a feature (the double gate)

This is the core flow. The reviewer gates the work on **both** ends.

1. Read `reference/coding-standards.md`, `reference/architecture.md`,
   `reference/patterns.md`, and the relevant `templates/*.md`.
2. Delegate to **`rn-feature-builder`**. Instruct it to FIRST produce a **plan**:
   the list of files it will create or change, and the key code snippets — NOT
   the final code yet.
3. **PRE-CHECK.** Pass that plan to **`rn-pattern-reviewer`** with the question:
   "Does this plan follow the standards?" The reviewer checks structure, file
   placement, pattern usage, and the comment plan.
   - FAIL → return findings to `rn-feature-builder`, get a revised plan, re-check.
   - PASS → continue.
4. `rn-feature-builder` writes the actual code, with full comments per the
   mandatory commenting standard, AND a Maestro E2E flow for the feature under
   `.maestro/`.
5. **POST-CHECK.** Pass the written code to **`rn-pattern-reviewer`** with the
   question: "Does this code follow the standards?"
   - FAIL → return findings to `rn-feature-builder` to fix, then re-check.
   - PASS → continue.
6. **RUN & TEST.** Delegate to **`rn-runner`**: build and launch the app, confirm
   it runs with no crash/console errors, run the feature's Maestro flow, capture
   screenshots.
   - If the build fails, the app crashes, or the flow fails → return the report
     to `rn-feature-builder` to fix the feature, then re-run from the POST-CHECK.
   - If no simulator/emulator is available, `rn-runner` says so; the feature is
     reported as code-verified-only and the user decides.
7. Report what was built, confirm both review gates passed, and give the
   `rn-runner` result (build/launch/flow + screenshot paths). State plainly that
   visual review is the user's.

`rn-feature-builder` must NOT write final code until the reviewer has PASSED its
plan. The feature is not "done" until the reviewer has PASSED its code AND
`rn-runner` reports the app runs and the flow passes (or the user accepts a
no-simulator limitation).

### Flow C — Review existing code

Delegate straight to **`rn-pattern-reviewer`** with the files to audit. Report its
findings. If the user wants the findings fixed, run them through
`rn-feature-builder` and then the post-check.

### Flow D — Build / deploy

Delegate to **`rn-build-deployer`**. It handles EAS builds, OTA updates,
credentials, and submission guidance.

### Flow E — Run or test the app

Delegate to **`rn-runner`** to build and launch the app on a simulator/emulator,
detect crashes, and run Maestro E2E flows. Use for "run the app", "test the
login feature", "does it build". `rn-runner` verifies behaviour, not appearance
— it reports screenshot paths for the user's visual review. See
`reference/testing.md`.

---

## The review loop — block and loop until pass

The reviewer is a hard gate. The standard is **block-and-loop-until-pass**:

- A FAIL always sends the work back to the executor agent with the specific
  findings attached.
- The executor fixes ONLY what the findings call out — no unrelated changes.
- The work is re-reviewed.
- This repeats until the reviewer returns PASS.
- Nothing is reported to the user as "done" until a PASS is achieved on the
  final code.

There is no try limit — loop until PASS. If the same finding recurs three times
unchanged, stop and surface it to the user with both the reviewer's finding and
the executor's attempts, so the user can decide. (This is the only escape hatch,
and it is an escalation, not an acceptance.)

---

## Hard rules for every code task

These are summarized here; the full, authoritative rules are in
`reference/coding-standards.md`. Pass them to every agent.

1. **Strict standards.** Follow `reference/coding-standards.md` exactly. The
   reviewer enforces it with zero tolerance.
2. **Mandatory comments.** Every file, every function, and every non-obvious or
   complex logic block must be commented — explaining *what* it does and *why*.
   The reviewer FAILS code that lacks required comments. See the commenting
   section of `reference/coding-standards.md`.
3. **No new patterns.** Reuse the signature patterns in `reference/patterns.md`.
   Do not invent alternatives.
4. **Surgical changes.** Touch only what the task requires. Do not refactor or
   "clean up" adjacent code.
5. **No hardcoded styling.** Colors, spacing, and typography come from the theme
   and constants — never literal values in components.
6. **TypeScript strict.** No implicit `any`. Type every prop, request, response,
   and slice state.
7. **Locked stack only.** Never introduce a library outside the Locked Stack
   table without explicit user approval.
