# Testing — running the app and verifying features

This file defines how the skill runs an app and verifies that features work.
`rn-runner` is the agent that executes everything here.

## What can and cannot be verified — read this first

Be honest about the limits. An agent driving a build has **no eyes on the
simulator screen**. It perceives the app only through process output.

| Aspect | Verifiable automatically? | How |
| --- | --- | --- |
| TypeScript / type errors | ✅ Yes | `tsc --noEmit` output |
| Build success / failure | ✅ Yes | `expo run:ios` / `run:android` exit code |
| Runtime crashes | ✅ Yes | Metro + device console logs |
| `console.error` / warnings | ✅ Yes | log stream |
| A feature flow working end-to-end | ✅ Yes | a Maestro flow's pass/fail |
| Visual correctness (layout, color, spacing, overlap) | ❌ NO | the agent cannot see pixels |
| UX "feel" | ❌ NO | — |

**Therefore:** the skill verifies the app **builds, launches, does not crash,
and that defined feature flows pass**. It does NOT verify appearance.
`rn-runner` captures **screenshots** during Maestro flows and saves them so a
human (or Claude, via the Read tool on the image) can review the visuals.
`rn-runner` must always state plainly that visual review is the user's job.

## Platform reality

- **iOS Simulator is macOS-only.** On Linux/Windows, only the Android emulator
  is available. `rn-runner` detects the OS and reports honestly which targets
  it can run.
- A simulator/emulator must be installed (Xcode for iOS, Android Studio for
  Android). If none is available, `rn-runner` says so rather than failing
  obscurely.

## Running the app

```bash
# iOS (macOS only) — builds and launches on the Simulator
npx expo run:ios

# Android — builds and launches on a running emulator / device
npx expo run:android

# Start the Metro bundler (for log capture / a dev client)
npx expo start
```

`rn-runner` runs the build, captures stdout/stderr, and scans the log stream
for: build failures, red-box runtime errors, `console.error`, unhandled promise
rejections. Any of these → a FAIL with the relevant log excerpt.

## End-to-end testing — Maestro

The skill uses **Maestro** for E2E feature tests. Maestro flows are simple YAML
files that tap, type, scroll, and assert against the running app.

### Why Maestro

- YAML flows — readable, diffable, no test framework boilerplate.
- Tolerant of minor UI change (matches by text / id / accessibility label).
- Runs against the same simulator/emulator the app already launches on.
- Built-in `takeScreenshot` for the human-review bridge.

### Where flows live

```
<project-root>/
  .maestro/
    <feature>.yaml        One flow per feature
    flows/                Optional: shared sub-flows
```

### Flow structure

Each flow starts with the app id and a launch, then drives the feature and
asserts. See `templates/maestro-flow.yaml.md` for the full skeleton.

```yaml
appId: com.company.appname
---
- launchApp:
    clearState: true
- assertVisible: "Login"           # the app reached the expected screen
- tapOn: "Email"
- inputText: "test@example.com"
- tapOn: "Sign in"
- assertVisible: "Home"            # the feature's success condition
- takeScreenshot: "login-success"  # saved for human visual review
```

### Running flows

```bash
maestro test .maestro/<feature>.yaml
# or every flow:
maestro test .maestro/
```

Maestro reports structured per-step pass/fail; `rn-runner` parses that and
reports which flows passed, which failed, and at which step.

## How testing fits the build workflow

When `rn-feature-builder` builds a feature, it ALSO writes a Maestro flow for
that feature under `.maestro/`. After the feature's code passes the
`rn-pattern-reviewer` post-check, the orchestrator delegates to **`rn-runner`**
to:

1. Launch the app (build + simulator/emulator).
2. Confirm it runs with no crash / no console errors.
3. Run the feature's Maestro flow.
4. Capture screenshots.
5. Report: build status, run status, flow pass/fail, screenshot paths, and an
   explicit note that visual review is the user's.

A feature is only "done" when its code passed review AND `rn-runner` reports
the build runs and the flow passes — OR the user explicitly accepts a known
limitation (e.g. no simulator available on this machine).

## Test conventions

- One Maestro flow per feature, named `<feature>.yaml`.
- A flow always starts with `launchApp` and asserts it reached the expected
  screen before exercising the feature.
- A flow asserts the feature's **success condition** (a screen, a piece of
  text, a state) — not just that taps did not error.
- `clearState: true` on `launchApp` for tests that need a clean slate (e.g.
  auth flows); omit it for tests that depend on prior state.
- Use text / accessibility-label matchers (which the skill already mandates,
  R83) — they make flows resilient and are why the accessibility rules matter.
- Flows reference the same i18n strings the UI shows; if the locale changes,
  flows match by id where text would be brittle.
- Screenshots are named meaningfully (`<feature>-<step>`); they are artifacts
  for human review, not assertions.
