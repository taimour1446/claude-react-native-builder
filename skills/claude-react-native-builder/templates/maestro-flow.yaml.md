# Template — Maestro E2E flow

Path: `<project-root>/.maestro/<feature>.yaml`

One flow per feature. The flow launches the app, navigates to the feature,
exercises it, and asserts the feature's success condition. Read
`reference/testing.md` for the conventions.

```yaml
# .maestro/<feature>.yaml
# E2E flow for the <feature> feature.
# Verifies: <one line — what working behaviour this flow proves>.

appId: <the app's bundle identifier, e.g. com.company.appname>
---
# --- Launch ---
# clearState gives a clean start; drop it for flows that depend on prior state.
- launchApp:
    clearState: true

# --- Reach the feature ---
# Assert the app opened on the expected screen before doing anything else.
- assertVisible: "<text or id proving the start screen rendered>"

# Navigate to the feature under test.
- tapOn: "<nav target — tab label, button text, or id>"
- assertVisible: "<text proving the feature screen rendered>"

# --- Exercise the feature ---
# Drive the feature's main path: taps, text input, scrolls.
- tapOn: "<a field label>"
- inputText: "<a valid test value>"
- tapOn: "<the submit / action control>"

# --- Assert the success condition ---
# The flow must assert the feature actually WORKED — a resulting screen,
# a confirming message, or a state change. Not just "no tap errored".
- assertVisible: "<the success indicator>"

# --- Screenshot for human review ---
# Saved as an artifact; a human (or Claude via Read) checks the visuals.
- takeScreenshot: "<feature>-success"
```

## Notes

- **Matchers:** prefer text and accessibility-label matchers — the skill
  already requires localized `accessibilityLabel`s (R83), which makes flows
  resilient. Match by `id` where visible text would be brittle.
- **Assert the outcome, not the action.** A flow that only taps proves nothing;
  it must `assertVisible` the feature's success state.
- **Negative paths** are worth a separate flow when a feature has meaningful
  failure handling (e.g. an invalid-login flow asserting an error message).
- **`clearState`:** use it for auth and first-run flows; omit it when the test
  depends on data created by an earlier step.
- **Screenshots are artifacts, not assertions** — name them `<feature>-<step>`.

Checklist:
- File at `.maestro/<feature>.yaml`, one flow per feature.
- `appId` matches the app's bundle identifier.
- Starts with `launchApp` + an `assertVisible` of the start screen.
- Asserts the feature's success condition explicitly.
- Captures at least one screenshot for human visual review.
- Uses text / id / accessibility-label matchers.
