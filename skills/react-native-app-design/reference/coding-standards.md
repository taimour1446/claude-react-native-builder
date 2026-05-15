# Coding Standards — the strict ruleset

This file is the **single source of truth** for code quality in this skill.
`rn-pattern-reviewer` enforces every rule here with **zero tolerance**. Every
other agent must follow it exactly. A rule numbered `[Rxx]` is referenced by the
reviewer in its findings.

Severity: every rule is **BLOCKING** unless explicitly marked `[ADVISORY]`.
A single blocking violation = the whole review FAILS.

---

## 1. Formatting

- **[R01] Indentation is 4 spaces.** No tabs. No 2-space blocks. Every file.
- **[R02] Semicolons required** at the end of every statement.
- **[R03] Single quotes** for strings. Double quotes only to avoid escaping.
- **[R04] One blank line** between logical sections inside a function; two blank
  lines are not used. No blank line at the start of a block.
- **[R05] Trailing commas** in multi-line arrays, objects, and parameter lists.
- **[R06] Max line length ~100 chars** `[ADVISORY]` — break long JSX props and
  long type unions onto multiple lines.

---

## 2. Imports — order and grouping

**[R07]** Imports appear in this exact order, each group separated by one blank
line:

1. React (`react`)
2. React Native core (`react-native`)
3. Third-party packages (Expo modules, navigation, redux, etc.)
4. Local — services, store, hooks, navigation, components
5. Local — types (use `import type { … }`)
6. Local — constants

```typescript
// 1. React
import React, { useState, useEffect } from 'react';
// 2. React Native core
import { View, Text, StyleSheet } from 'react-native';
// 3. Third-party
import { useTranslation } from 'react-i18next';
import { LinearGradient } from 'expo-linear-gradient';
// 4. Local modules
import { useTheme } from '../../theme/ThemeContext';
import { useAppDispatch } from '../../store/hooks';
import { Button } from '../../components/common';
// 5. Local types
import type { RootStackParamList } from '../../navigation/RootNavigator';
// 6. Local constants
import { ASSET_ICONS } from '../../constants';
```

- **[R08]** Type-only imports MUST use `import type`.
- **[R09]** Import shared components from the folder barrel
  (`'../../components/common'`), not the individual file.

---

## 3. Naming

- **[R10]** Components, screens, types, interfaces → `PascalCase`.
  Files for them match: `LoginScreen.tsx`, `CameraCard.tsx`.
- **[R11]** Hooks → `useCamelCase`, file name matches: `useAddCameraForms.ts`.
- **[R12]** Variables and functions → `camelCase`.
- **[R13]** Constants that are fixed values → `UPPER_SNAKE_CASE`
  (`API_BASE_URL`, `ACCESS_TOKEN_KEY`). Constant *objects* used as maps are
  `UPPER_SNAKE_CASE` too (`ASSET_ICONS`, `CAMERA_ICONS`).
- **[R14]** Redux slices → `<feature>Slice.ts`; RTK Query APIs →
  `<feature>Api.ts`; Yup schemas → `<name>Schema.ts`.
- **[R15]** Screen files end in `Screen`. Navigator files end in `Navigator`.
- **[R16]** Booleans read as predicates: `isLoading`, `hasAuthData`,
  `editMode`, `canSubmit`.

---

## 4. TypeScript

- **[R17]** `strict` mode is on. **No implicit or explicit `any`.** If a type is
  genuinely unknown, use `unknown` and narrow it.
- **[R18]** Every component has a typed props interface named `<Component>Props`.
- **[R19]** Every RTK Query endpoint has explicit request and response types,
  declared and exported from the API file.
- **[R20]** Every slice has a typed `State` type and typed `PayloadAction<…>`
  for each reducer.
- **[R21]** Function parameters and non-trivial return values are typed. Trivial
  inferred returns (e.g. a JSX-returning component) may rely on inference.
- **[R22]** Prefer `type` for unions/aliases and `interface` for object/props
  shapes — but be consistent within a file.

---

## 5. Component structure — the 9 sections

**[R23]** Every component file follows this exact section order. Mark each
section with its comment header.

```typescript
// ---- 1. Imports (ordered per R07) ----
import React, { useState, useEffect } from 'react';
import { View, Text, StyleSheet, ViewStyle } from 'react-native';
import { useTranslation } from 'react-i18next';
import { useTheme } from '../../theme/ThemeContext';

// ---- 2. Types ----
/**
 * Props for ExampleCard.
 * @param title  Heading text shown at the top of the card.
 * @param onPress  Optional callback fired when the card is tapped.
 */
interface ExampleCardProps {
    title: string;
    onPress?: () => void;
}

// ---- 3. Component definition ----
/**
 * ExampleCard — a tappable surface that shows a title.
 * Presentational only; all behaviour is passed in via props.
 */
export const ExampleCard: React.FC<ExampleCardProps> = ({ title, onPress }) => {
    // ---- 4. Hooks (theme, i18n, redux, local state, navigation) ----
    const { colors, typography, spacing } = useTheme();
    const { t } = useTranslation();
    const [isPressed, setIsPressed] = useState(false);

    // ---- 5. Effects ----
    useEffect(() => {
        // Reset the pressed flag whenever the title changes so a recycled
        // card never shows a stale pressed state.
        setIsPressed(false);
    }, [title]);

    // ---- 6. Event handlers ----
    /** Marks the card pressed, then delegates to the parent's onPress. */
    const handlePress = (): void => {
        setIsPressed(true);
        onPress?.();
    };

    // ---- 7. Dynamic (theme-dependent) styles ----
    // Theme values change at runtime, so theme-dependent styles cannot live in
    // the static StyleSheet below — they are built here on each render.
    const dynamicStyles = {
        container: {
            backgroundColor: colors.surface,
            padding: spacing.md,
        } as ViewStyle,
    };

    // ---- 8. Render ----
    return (
        <View style={[styles.container, dynamicStyles.container]}>
            <Text style={{ color: colors.text, fontSize: typography.fontSize.md }}>
                {title}
            </Text>
        </View>
    );
};

// ---- 9. Static styles ----
const styles = StyleSheet.create({
    container: {
        borderRadius: 12,
    },
});
```

- **[R24]** Static, theme-independent styles → `StyleSheet.create` at the
  bottom. Theme-dependent styles → the dynamic-styles object in section 7.
- **[R25]** Components are exported as **named exports** (`export const X`).
  Screens are the one exception — they use `export default` because navigators
  import them that way. (See R26.)
- **[R26]** Screen components: `export default function XScreen(props: Props)`.
  Props typed via `NativeStackScreenProps<RootStackParamList, 'X'>` (or the tab
  equivalent).

---

## 6. MANDATORY COMMENTING STANDARD

Comments are not optional. The reviewer FAILS any code that omits a required
comment. The goal: a reader understands *what* the code does and *why* without
running it.

- **[R27] Every file** that contains non-trivial logic begins with a short
  header comment stating the file's purpose. (Barrels and pure re-export files
  are exempt.)
- **[R28] Every function, hook, and component** has a doc comment (`/** … */`)
  above it stating WHAT it does. If its behaviour is not obvious, also state
  WHY it exists / why it works this way.
- **[R29] Every exported function** documents its parameters and return value
  when they are not self-evident from the names and types.
- **[R30] Every complex or non-obvious logic block** has an inline comment
  explaining what is happening and why. "Complex or non-obvious" includes:
  - conditionals whose purpose is not obvious from the expression,
  - data transformations / mappings,
  - workarounds, ordering requirements, race-condition handling,
  - anything a future reader would pause at.
- **[R31]** Comments explain intent, not the obvious. Do NOT write
  `// set loading to true` above `setLoading(true)`. DO write why the ordering,
  the guard, or the workaround is necessary.
- **[R32]** Comments stay in sync with code. A comment that contradicts the code
  is a FAIL.
- **[R33] No commented-out code** left in the file. Delete dead code.
- **[R34] No `TODO` / `FIXME`** left unresolved in delivered code. If something
  is genuinely deferred, it is surfaced to the user, not buried in a comment.

Example of a correctly commented non-obvious block:

```typescript
// expo-video's native module is heavy to initialize and blocks the JS thread.
// We delay the dynamic import by one tick so the surrounding screen paints
// first, then load the player in the background.
setTimeout(() => {
    void loadVideoModule();
}, 0);
```

---

## 7. Styling rules

- **[R35] No hardcoded colors.** Every color comes from `colors` via
  `useTheme()`. A hex/`rgb()`/named CSS color literal in a component is a FAIL.
  Allowed exceptions, and ONLY these:
  - `'transparent'`.
  - A theme color with an appended hex-opacity suffix for a tint, e.g.
    `colors.primary + '10'` — the base must still be a theme color.
  - Semi-transparent black/white overlay scrims (`'rgba(0,0,0,0.5)'`) used as a
    modal/bottom-sheet backdrop — a documented, intentional use. Anything else
    is a FAIL.
- **[R36] No magic spacing numbers.** Padding, margin, and gap use the `spacing`
  scale from the theme. Raw pixel numbers for layout spacing are a FAIL.
  Numbers for `borderRadius`, `width`/`height` of fixed assets, and similar
  non-spacing dimensions are allowed but should be intentional.
- **[R37] No hardcoded font sizes / families.** Use `typography` from the theme.
- **[R38]** Static layout styles → `StyleSheet.create`. Never inline a large
  style object in JSX; small one-off style props are acceptable.
- **[R39]** `lightColors` and `darkColors` must have identical key sets.

---

## 8. State, data, and Redux

- **[R40]** Components read state with `useAppSelector` and dispatch with
  `useAppDispatch` from `src/store/hooks.ts`. Raw `useSelector`/`useDispatch`
  is a FAIL.
- **[R41]** Server data goes through RTK Query (`*Api.ts`), not manual `fetch`
  in components or thunks.
- **[R42]** Every `*Api.ts` uses `createBaseQueryWithAuth()` from `baseQuery.ts`.
- **[R43]** Do NOT handle `401` errors in components — `baseQuery` does it
  globally (logout + navigate). Do NOT add `Authorization` headers manually —
  `baseQuery` adds them.
- **[R44]** Generic API errors (403/404/5xx) are surfaced by `baseQuery` as
  toasts globally. Components handle only `400` field-level errors, mapping them
  onto the form.
- **[R45]** User-facing messages go through the global toast
  (`dispatch(showToast({ … }))`). Never build a local ad-hoc toast/snackbar.
- **[R46]** Slice reducers are pure. Side effects (secure storage writes,
  navigation) are fired from within reducers/thunks as fire-and-forget promises
  with a `.catch`, exactly as the auth slice does — or from RTK Query
  `matchFulfilled` matchers.

---

## 9. Navigation

- **[R47]** Every navigator declares and exports a typed `ParamList`.
- **[R48]** In-component navigation uses the typed `navigation` prop. Navigation
  from outside React (thunks, services, error handlers) uses `navigationRef`.
- **[R49]** Auth-gated routing is driven by the `auth` slice in
  `RootNavigator`, never a local flag.

---

## 10. Forms and validation

- **[R50]** Forms use `react-hook-form` with `yupResolver`.
- **[R51]** Every form field renders through `<Controller>`.
- **[R52]** Validation schemas are factories: `createXSchema(t)` returning a Yup
  object; the file also exports the `XFormData` interface.
- **[R53]** `useForm` is configured with `mode: 'onChange'` for immediate
  feedback unless the task explicitly requires otherwise.
- **[R54]** Non-trivial forms use the two-hook pattern: `use<Feature>Forms`
  (state) + `use<Feature>Handlers` (logic). Screens stay thin.
- **[R55]** Backend field names (snake_case) are mapped to frontend names
  (camelCase) before `setError`, using the field-mapping utility.

---

## 11. Safety, errors, performance

- **[R56]** Async functions that can fail are wrapped in `try/catch`. Failures
  are either surfaced to the user (toast) or logged with `console.warn` /
  `console.error` — never silently swallowed with an empty catch.
- **[R57]** Cleanup-style operations (clearing storage on logout) catch and log
  but do NOT re-throw, so they cannot block the flow.
- **[R58]** Heavy native components (e.g. video) are lazy-loaded; do not import
  them eagerly at module scope on a screen. See `patterns.md`.
- **[R59]** Don't render layout-occupying components before they are functional;
  mount conditionally. See `patterns.md`.
- **[R60]** No secrets in source. API keys / tokens come from env or secure
  storage, never literals.
- **[R60a]** Lists use `FlatList`/`SectionList`, never `.map()` inside a
  `ScrollView` for variable-length data. `keyExtractor` returns a stable id
  (never the array index); `renderItem` is typed `ListRenderItem<T>`; long
  lists set the performance props. See `data-patterns.md` §1.
- **[R60b]** Data screens distinguish the four loading states (initial,
  refetch, pull-refresh, load-more) — see `data-patterns.md` §2. A blank screen
  during loading or empty is a FAIL; use `LoadingIndicator` / `EmptyState`.
- **[R60c]** Dates that cross Redux state or the API are ISO 8601 strings, not
  `Date` objects (Redux state must stay serializable). Date formatting is
  timezone-aware via the `dateFormatters` utilities. See `utils-and-types.md`.
- **[R60d]** CREATE and DELETE mutations carry an idempotency key generated
  with `generateUUID()`. See `data-patterns.md` §11.

---

## 12. Localization (i18n) — NO hardcoded user-facing strings

The whole app is localizable from day one. A literal user-facing string in JSX
or in a thrown/dispatched message is a FAIL.

- **[R61] No hardcoded user-facing text.** Every string a user can SEE — screen
  titles, labels, button titles, placeholders, empty-state text, error messages,
  toast messages, accessibility labels/hints — comes from `t('key')`. A string
  literal passed to `<Text>`, to a `title`/`label`/`placeholder` prop, to
  `showToast({ message })`, or to an `accessibilityLabel` is a FAIL.
  - Allowed literals: keys, test IDs, style values, icon names, non-visible
    technical strings, and dynamic data already coming from the API.
- **[R62] All keys live in `locales/<lang>/common.json`.** When a feature is
  built, every new string is added to the locale file in the SAME change. The
  reviewer FAILS a `t('x')` whose key `x` is missing from the locale JSON.
- **[R63] Dynamic values use interpolation,** never string concatenation:
  `t('welcome', { name })` with `"welcome": "Welcome, {{name}}!"` — NOT
  `t('welcome') + name`.
- **[R64] Key naming:** `camelCase`, grouped by feature prefix where helpful
  (`emailRequired`, `cameraNamePlaceholder`). Keys describe meaning, not the
  English text.
- **[R65]** Validation messages: schema factories receive `t` and call it for
  every message (see R52). No literal validation strings.
- **[R66]** A fallback after `t()` (`t('x') || 'Default'`) is allowed only as a
  safety net; the key must still exist in the locale file.
- **[R67]** Screens and components get `t` from `useTranslation()`. Non-React
  modules (e.g. `baseQuery`) use the `i18next` instance directly.

## 13. Text display and typography

- **[R68]** All visible text is rendered inside a React Native `<Text>` (or a
  shared text-bearing component). Never put a bare string elsewhere.
- **[R69]** Text color comes from `colors` (theme); font size and family come
  from `typography` (theme). No literals — this restates R35/R37 for text.
- **[R70] Line height is computed,** not guessed. Use the `getLineHeight(fontSize, ratio)`
  utility (ratio ~1.3 for headings, ~1.4 for body) so text scales correctly.
- **[R71] Font sizes scale with the OS accessibility setting.** The `typography`
  scale is built with the `scaleFont()` utility — feature code must consume
  `typography.fontSize.*`, never raw numbers, so OS font scaling is respected.
- **[R72]** Multi-line-risk text (titles, labels in tight layouts) sets
  `numberOfLines` + `ellipsizeMode` deliberately rather than overflowing.
- **[R73]** Use the semantic size scale (`xs/sm/md/lg/xl/xxl/xxxl/icon`) — pick
  the size by role (`md` = body, `xl` = screen title, etc.), do not invent sizes.

## 14. Theming

- **[R74]** All color, spacing, and typography access goes through
  `useTheme()`. A component must work in BOTH light and dark mode with no extra
  work — which it does automatically if it only uses theme values.
- **[R75]** New colors are added to BOTH `lightColors` and `darkColors` with the
  SAME key, chosen appropriately for each mode. Adding a key to only one is a
  FAIL (restates R39).
- **[R76]** Color keys are semantic, not literal: `colors.error`,
  `colors.surface`, `colors.placeholderText` — never `colors.red`,
  `colors.grey2`.
- **[R77]** Theme-dependent styles are built in component section 7 (the dynamic
  styles block); static styles go in `StyleSheet.create`. Never read `useTheme()`
  values into a module-scope `StyleSheet` (it would not update on theme change).
- **[R78]** The theme mode is persisted (AsyncStorage) and restored on launch —
  this is wired by the scaffolder; feature code must not bypass it with a local
  mode state.

## 15. Context usage

- **[R79]** A React Context is the right tool ONLY for app-wide, low-frequency,
  non-server state — theme is the canonical example. Server data and complex
  shared state use Redux (see R40/R41). Do not add a Context as an alternative
  to a slice.
- **[R80]** Every context exposes a typed value, is created with a typed default
  of `undefined`, and ships a `useX()` hook that throws if used outside its
  provider — exactly as `ThemeContext` does:
  ```typescript
  const ctx = useContext(ThemeContext);
  if (!ctx) throw new Error('useTheme must be used within ThemeProvider');
  ```
- **[R81]** Providers are mounted in `App.tsx` in the fixed nesting order (see
  `architecture.md`). A new provider is added there, in a deliberate position,
  never deep in the tree.
- **[R82]** Context provider values that are objects should be stable across
  renders where the context is high-frequency; theme is low-frequency so a
  fresh object per render is acceptable. `[ADVISORY]`

## 16. Accessibility

- **[R83]** Every interactive element (`TouchableOpacity`, pressable, custom
  button) has an `accessibilityRole` and an `accessibilityLabel` — and the label
  is localized via `t()` (R61).
- **[R84]** Inputs set `accessibilityLabel` (the field label) and, where useful,
  `accessibilityHint`. Error text uses `accessibilityRole="alert"` with
  `accessibilityLiveRegion="polite"` so screen readers announce it.
- **[R85]** Small touch targets add `hitSlop` so they meet the ~44pt minimum.
- **[R86]** Decorative-only images set `accessibilityElementsHidden` /
  `importantForAccessibility="no"`; meaningful images get a label. `[ADVISORY]`

## 17. Scope discipline

- **[R87]** Make only the change the task asks for. Do not refactor, reformat,
  rename, or "improve" adjacent code.
- **[R88]** Do not add a dependency outside the Locked Stack (see `SKILL.md`)
  without explicit user approval.
- **[R89]** Do not create files outside the 12-folder architecture.

---

## Reviewer verdict format

`rn-pattern-reviewer` returns exactly one of:

- **`PASS`** — every rule satisfied. One line confirming it.
- **`FAIL`** — one finding per violation, each as:

  ```
  FAIL [Rxx] <file>:<line> — <what is wrong> → <what the fix is>
  ```

  Followed by a one-line summary count. No PASS may be issued while any FAIL
  finding stands.

The reviewer must check ALL rule sections 1–17 (rules R01–R89), not just code
formatting — localization (§12), text/typography (§13), theming (§14), context
(§15), and accessibility (§16) are checked with the same zero tolerance.
