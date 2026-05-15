# Conventions — every small detail

This file captures the fine-grained conventions of the architecture: the things
that are not "big rules" but make produced code identical to reference code.
Read alongside `coding-standards.md` (the enforced rules) and `patterns.md`
(the structural patterns).

---

## 1. Barrel exports (`index.ts`)

Every `components/<group>/`, every `screens/<group>/`, every
`validation/<group>/`, every feature `hooks/<group>/`, and `constants/` has an
`index.ts` barrel.

- **Components barrel** — named re-exports; also re-export any public type the
  component file exposes:
  ```typescript
  export { Button } from './Button';
  export { Picker, type OptionItem as PickerOptionItem } from './Picker';
  export { EmptyState } from './EmptyState';
  ```
- **Screens barrel** — screens are default exports, so re-export with a name:
  ```typescript
  export { default as HomeScreen } from './HomeScreen';
  ```
- **Constants barrel** (`constants/index.ts`) — wildcard re-export each file:
  ```typescript
  export * from './api';
  export * from './spacing';
  export * from './typography';
  ```
- When a new file is added to a barrelled folder, the barrel is updated in the
  SAME change. A new component not added to its barrel is incomplete.

---

## 2. Export style per file type

| File type | Export style |
| --- | --- |
| Component | `export const ComponentName: React.FC<Props> = …` (named) |
| Component using `forwardRef` | `export const X = React.forwardRef<Ref, Props>(…)` (named) |
| Screen | `export default function XScreen(props: Props)` |
| Navigator | `export default function XNavigator()` + named `export type XParamList` |
| RTK Query API | named `export const xApi` + named hook exports |
| Slice | `export default xSlice.reducer` + named action exports + named selectors |
| Hook | `export const useX = …` (named) |
| Util | named `export const` / `export function` per helper |
| Validation schema | named `export const createXSchema` + named `export interface XFormData` |
| Constants | named `export const X` |

---

## 3. Constants conventions

- Constant *map objects* (icon maps, option lists) are `UPPER_SNAKE_CASE` and
  end with `as const` so TypeScript infers literal types:
  ```typescript
  export const ASSET_ICONS = {
      BACK: require('../../assets/icons/common/ic_back.png'),
      MENU: require('../../assets/icons/common/ic_menu.png'),
  } as const;
  ```
- Derive types FROM the const object rather than declaring them separately:
  ```typescript
  export const COMPONENT_SIZE = { SMALL: 'small', LARGE: 'large' } as const;
  export type ComponentSize = typeof COMPONENT_SIZE[keyof typeof COMPONENT_SIZE];
  ```
- Image assets are referenced through `require()` inside an icon map in
  `constants/icons.ts` — never `require()` inline in a component.
- Icon files on disk follow `ic_<name>.png`, grouped in
  `assets/icons/<group>/`. Icon map keys are `UPPER_SNAKE_CASE`.
- Fixed backend IDs / config values live in `constants/api.ts`.

---

## 4. Theme & color conventions

- `colors.ts` exports `lightColors` and `darkColors`. `darkColors` is typed
  `: typeof lightColors` so the key sets cannot drift:
  ```typescript
  export const lightColors = { primary: '#FC4C00', background: '#FFFFFF', /* … */ };
  export const darkColors: typeof lightColors = { primary: '#FC4C00', background: '#0A0A0A', /* … */ };
  ```
- Color keys are **semantic and grouped with comments**: brand, input state,
  active/gradient, inactive, disabled, etc.
- The theme object exposed by `useTheme()` is exactly:
  `{ mode, colors, typography, spacing, toggle }`.
- `ThemeProvider` persists the mode to `AsyncStorage` under `'themeMode'` and
  restores it on mount; it also calls `SystemUI.setBackgroundColorAsync` so the
  native background matches.
- Default mode is `'light'`.

---

## 5. Typography & text conventions

- `typography` has `fontFamily` (thin/light/regular/medium/bold/black) and
  `fontSize` (xs/sm/md/lg/xl/xxl/xxxl/icon).
- If a font weight has no font file, alias it to the nearest available one and
  comment why (`medium: 'Lato-Regular', // fallback, no Lato-Medium.ttf`).
- Every `fontSize` entry is wrapped in `scaleFont()` so OS accessibility text
  scaling applies. Feature code reads `typography.fontSize.*`, never raw numbers.
- `lineHeight` on any styled text is computed with `getLineHeight(fontSize, ratio)`
  (`fontScaling.ts`) — ratio ≈ 1.3 headings, ≈ 1.4 body.
- Size selection by role: `xs` badges/hints, `sm` labels/secondary/errors,
  `md` body & inputs (default), `lg` emphasis/subheaders, `xl` screen titles,
  `xxl` brand/hero, `xxxl` large display numbers, `icon` FAB/large symbols.
- Icons/decorative text that must NOT grow with accessibility settings use
  `scaleWithoutAccessibility()`.

---

## 6. Spacing conventions

- The scale is fixed: `xs:4, sm:8, md:16, lg:20, xl:24, xxl:32`.
- All layout padding/margin/gap use `spacing.*`. The only raw numbers allowed
  are non-spacing dimensions (a fixed icon `width: 18`, a `borderRadius`).
- Spacing values are NOT run through font scaling by default (they are stable);
  `scaleSpacing()` exists for the rare case spacing must track text — use it
  only deliberately.

---

## 7. Component conventions

- Functional components only. No class components.
- Props are destructured in the parameter list, with defaults assigned there:
  `({ variant = 'active', size = 'medium', fullWidth = false, style, ...props })`.
- A component that wraps a native input/touchable spreads remaining props with
  `{...props}` and extends the native props interface
  (`interface XProps extends TouchableOpacityProps`).
- Components that need a ref to a native element use `React.forwardRef`.
- Variant/size props are string unions (`'small' | 'medium' | 'large'`), and the
  concrete value is resolved with a nested ternary into a local variable.
- Theme-dependent style objects are typed `ViewStyle` / `TextStyle` / `ImageStyle`
  and built in section 7 of the component.
- A handler that forwards to an optional prop uses optional chaining:
  `onPress?.()`, `props.onFocus?.(e)`.

---

## 8. Screen conventions

- `export default function XScreen({ navigation, route }: Props)`.
- `Props` is `NativeStackScreenProps<RootStackParamList, 'X'>` (or the bottom-tab
  / top-tab equivalent).
- Screen root is `<SafeAreaView>` from `react-native-safe-area-context`.
- Any screen with text inputs wraps content in `<KeyboardAvoidingView>` with a
  `Platform.OS === 'ios' ? 'padding' : 'height'` behavior, and usually a
  `<ScrollView>` inside.
- Screens are thin: they call hooks, render components and `<Controller>`
  fields, and pass handlers. Business logic lives in `hooks/`.
- Form screens get state from `useXForms()` and logic from `useXHandlers()`.

---

## 9. Navigation conventions

- `RootNavigator` is a `createNativeStackNavigator` with
  `screenOptions={{ headerShown: false }}` — this app draws its own headers.
- `RootNavigator` renders TWO mutually exclusive groups based on
  `isAuthenticated` derived from the `auth` slice (`!!(user && accessToken)`):
  the authenticated stack and the auth-flow stack.
- `RootStackParamList` is a `type` mapping every route name to its params type
  (`undefined` when none). Routes that take an id: `{ id: string }`.
- `TabNavigator` is a `createBottomTabNavigator`; tab icons come from
  `NAVIGATION_ICONS` with active/inactive variants chosen by `focused`; tab bar
  colors come from the theme; bottom padding/height respects
  `useSafeAreaInsets()`.
- Tab labels and any visible navigation text use `t()`.
- `navigationRef` is used only outside React; always guard `isReady()`.

---

## 10. Redux conventions

- One slice per domain, file `<feature>Slice.ts`, slice `name` is the feature.
- State type is a `type` named `<Feature>State`; `initialState` is typed with it.
- Reducers mutate draft state directly (Immer) — each reducer has a doc comment.
- `PayloadAction<…>` types every reducer's payload.
- Async work that is not a server call uses `createAsyncThunk`; its three
  lifecycle cases are handled in `extraReducers` with `addCase`.
- Server state is synced from RTK Query with
  `addMatcher(api.endpoints.x.matchFulfilled, …)`.
- **Ordering:** in `extraReducers`, all `.addCase(...)` come before all
  `.addMatcher(...)` — RTK enforces this.
- Selectors are named `selectX`, declared in the slice file, typed with
  `(state: RootState)`.
- Side effects fired from a reducer (secure-storage write) are fire-and-forget
  with a `.catch(...)` that logs.
- `store/index.ts` registers, for every API: the reducer under
  `[xApi.reducerPath]` and `.concat(xApi.middleware)`.

---

## 11. Services / API conventions

- One `createApi` per domain in `<feature>Api.ts`, `reducerPath: '<feature>Api'`.
- `baseQuery: createBaseQueryWithAuth()` — always.
- Request/response types are `type` aliases declared above the `createApi` call
  and exported. The response envelope is `{ ok: boolean; data: … }`.
- Endpoints: `builder.query` for GET, `builder.mutation` for POST/PUT/DELETE.
  Each endpoint has a one-line comment with its HTTP method + path.
- Generated hooks are re-exported at the bottom: `export const { useXQuery } = xApi;`.

---

## 12. Error handling conventions

- `utils/errorHelpers.ts` is the standard place for API-error parsing. Reuse:
  - `parseApiErrors(error)` → `Record<field, message> | null` for 400 field
    errors (already maps backend → frontend field names).
  - `setFormErrors(setError, fieldErrors)` → pushes them onto the form.
  - `getGenericErrorMessage(error, fallback)` → extracts a single message.
  - `isApiFieldError(error)` → guard.
- The API error envelope is `{ ok: false, error: { code, message } }` where
  `message` may be a string, string[], or `Record<field, string|string[]>`.
- Backend field names are mapped to frontend names via
  `utils/fieldMappings.ts` → `mapBackendFieldName()` and the
  `BACKEND_TO_FRONTEND_FIELD_MAP` object. New mismatched fields are added there.
- Components handle ONLY 400 field errors. 401 → global logout. 403/404/5xx →
  global toast. (See `patterns.md` pattern 1.)

---

## 13. Forms conventions

- `useForm<XFormData>({ resolver: yupResolver(schema), mode: 'onChange', defaultValues })`.
- The schema is built from a factory and **memoized with `useMemo`** when it
  depends on props (e.g. `editMode`) so it is not recreated each render:
  ```typescript
  const schema = useMemo(() => createXSchema(t, editMode), [t, editMode]);
  ```
- Every field renders inside `<Controller control={…} name="…" render={…} />`.
- On submit failure, parse with `parseApiErrors` then `setFormErrors`.
- Two-hook split for non-trivial forms (`useXForms` + `useXHandlers`).

---

## 14. i18n conventions

- `i18n/index.ts` exports `setupI18n()` (async, called in `App.tsx`) and a
  default `i18n` instance.
- Config: `ns: ['common']`, `defaultNS: 'common'`, `fallbackLng: 'en'`,
  `interpolation: { escapeValue: false }`.
- Resources are typed: `as const satisfies Resource`.
- Strings live in `locales/<lang>/common.json`, flat `camelCase` keys.
- Interpolation uses `{{name}}` placeholders.
- Adding a language = add `locales/<lang>/common.json` with the SAME keys and
  register it in the `resources` object.
- In React: `const { t } = useTranslation();`. Outside React:
  `import i18next from 'i18next'; i18next.t('key');`.
- Validation message keys follow a consistent naming scheme:
  `<field>Required`, `<field>Invalid`, `<field>MinLength`, `<field>MaxLength`,
  `<field>Mismatch` (cross-field), `<field>Weak` (strength). API error keys:
  `apiErrorGeneric`, `apiErrorForbidden`, `apiErrorNotFound`, `apiErrorServer`,
  `apiErrorNetwork`.

---

## 14a. Navigation — extra conventions

- A material-top-tabs navigator (`createMaterialTopTabNavigator`) is used for
  swipeable in-app tab sets. Tabs can be generated DYNAMICALLY from a query
  result; a render-prop `<Tab.Screen>{() => <Content … />}</Tab.Screen>` passes
  props into the tab content. A custom tab bar is supplied via the `tabBar`
  prop. See `patterns.md` §15 for the bottom-tab shape.
- `navigation.reset({ index: 0, routes: [{ name: 'Tabs' }] })` is used to drop
  the back stack — e.g. after a destructive action or a 404 — not only for the
  auth flow.
- Navigation can be conditional on data state, e.g. routing to a different
  screen depending on an entity's status.

---

## 15. Comment style conventions

- File header: a short `//` or `/** */` block stating the file's purpose.
- Functions / hooks / components: `/** … */` JSDoc above the declaration.
- Exported helpers document params/return with `@param` / `@returns` when not
  obvious (see `fontScaling.ts`, `errorHelpers.ts` for the reference style).
- Inline `//` comments above non-obvious blocks explain intent (the WHY).
- Section markers in components: `// ---- N. Section ----` (the 9 sections).
- No commented-out code, no unresolved `TODO`/`FIXME` in delivered files.

---

## 16. EAS / build conventions

- `eas.json` defines `development`, `preview`, `production`, and
  `production-aab` (extends `production`, `buildType: app-bundle`) profiles.
- Each profile sets `channel` and an `env` block with `EXPO_PUBLIC_*` vars.
- `appVersionSource: 'remote'`, `production.autoIncrement: true`.
- `package.json` scripts wrap EAS commands:
  `build:preview:*`, `build:prod:*`, `update:preview`, `update:prod`,
  `build:status`, `credentials:*`.
- OTA updates are published per `eas.json` channel via `eas update --branch`.
- `app.config.ts` is dynamic: it loads `.env` only when NOT in EAS/CI
  (`if (!process.env.EAS_BUILD && !process.env.CI) require('dotenv-flow/config')`),
  and exposes runtime config through `expo.extra`.
- Env vars: public ones are `EXPO_PUBLIC_*`; `.env.example` is committed and
  documents them; `.env` is gitignored.
- CI: GitHub Actions workflows publish EAS Updates — one per channel
  (`eas-update-preview.yml`, `eas-update-production.yml`). Triggered on
  `pull_request`; steps: checkout, Node 20 (`actions/setup-node`),
  `expo/expo-github-action@v8` with the `EXPO_TOKEN` secret, `npm ci`,
  `eas update --auto --branch <channel>`, then comment on the PR.
- `app.config.ts` concrete fields to set per project: `name`, `slug`, `scheme`,
  `owner`, `version`, `orientation: 'portrait'`, `userInterfaceStyle: 'light'`,
  `icon`, `newArchEnabled: true`, `splash`, `updates.url`,
  `runtimeVersion: { policy: 'appVersion' }`, iOS
  `bundleIdentifier` + `infoPlist.ITSAppUsesNonExemptEncryption: false` +
  `supportsTablet: false`, Android `package` + `googleServicesFile` +
  `adaptiveIcon` + `edgeToEdgeEnabled: true` +
  `predictiveBackGestureEnabled: false` + `permissions`, the `plugins` array
  (with config blocks for `expo-splash-screen` and `expo-notifications`), and
  `extra` (API base URL + `eas.projectId`).
- `.gitignore` MUST include: `node_modules/`, `.expo/`, `dist/`, `web-build/`,
  `expo-env.d.ts`, `*.tsbuildinfo`, `.env*` (with a `!.env.example` exception),
  signing files (`*.jks`, `*.p8`, `*.p12`, `*.key`, `*.mobileprovision`),
  `.DS_Store`, and the generated `/ios` and `/android` folders.

---

## 17. Git conventions

- **Never set the commit author to "Claude".** Use the repository's normal
  author identity.
- Commit messages use imperative mood: "Add camera timeline", not "Added…".
- Feature branches: `feature/<short-description>`.

---

## 18. Animation conventions

- Component animations use React Native's built-in **`Animated`** API
  (`Animated.timing`, `Animated.spring`, `Animated.parallel`,
  `Animated.sequence`, `Animated.loop`) — NOT `react-native-reanimated`
  directly. Reanimated is present only as a transitive dependency of animation
  libraries (sliders, etc.); do not author reanimated worklets in feature code
  unless a task specifically calls for it.
- Set `useNativeDriver: true` for transform/opacity animations (driver-eligible
  properties) so they run off the JS thread.
- A toast-style entrance uses `Animated.spring` (springy settle); an exit uses
  `Animated.timing` (~300ms). A combined slide+fade uses `Animated.parallel`.
- Keep animation durations/configs as named constants, not magic numbers
  scattered in JSX — e.g. a `constants/<feature>.ts` value or a local `const`.

---

## 19. Platform-specific code

- Branch on `Platform.OS` (or `Platform.select`) only where iOS and Android
  genuinely differ. Common, expected branches:
  - `KeyboardAvoidingView` `behavior` — `'padding'` on iOS, `'height'` on
    Android.
  - A picker — a modal bottom sheet on iOS vs an inline native picker on
    Android.
  - `expo-notifications` Android notification channels (Android only).
  - An OTA update screen shown on Android only.
- Keep the branch as narrow as possible — branch a single prop or block, not a
  whole component, when that suffices.
