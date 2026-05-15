# Scaffold Checklist — building a new app

The `rn-scaffolder` agent follows this checklist IN ORDER. Every step must be
done; the result is a runnable app with the full 12-folder architecture wired,
an auth flow shell, theme, store, i18n, and EAS config — ready for features.

The scaffolded shell is NOT feature code. It is the skeleton that every
`rn-feature-builder` run extends.

---

## Inputs required from the user

These MUST be collected from the user before scaffolding starts — they cannot
be guessed or defaulted. The orchestrator asks for them explicitly:

- **App display name** — the user-visible name, e.g. "Camory AI". Used for
  `expo.name` and the home-screen label.
- **Slug** — kebab-case project slug, e.g. `camory-ai`. Used for `expo.slug`
  and the project directory if no directory is given.
- **iOS bundle identifier** — reverse-DNS, e.g. `com.company.appname`. Used for
  iOS `bundleIdentifier`.
- **Android package name** — reverse-DNS, e.g. `com.company.appname`. Used for
  Android `package`.
- **Target directory** — where to create the project.
- **Expo account choice** — ask the user ONE of:
  - **Link now** — they have (or will create) an Expo account and want EAS set
    up during scaffolding. If they want a NEW account, point them to run
    `eas login` themselves (signup is interactive and account-bound — the skill
    cannot create an account for them) or sign up at expo.dev first.
  - **Defer** — scaffold now, link the Expo account later. The skill leaves
    clearly-marked placeholders and prints the exact follow-up steps.

**Ask for the iOS identifier and the Android package separately.** They are
commonly the same reverse-DNS string, but they are two distinct fields in
`app.config.ts` and the user may want them to differ. When asking, offer the
same value as the suggested default for both, but let the user override either
one. Do not collapse them into a single "bundle id" question.

A `scheme` (deep-link URL scheme) is derived from the slug (e.g. slug
`camory-ai` → scheme `camoryai`) unless the user specifies one.

---

## Step 1 — Create the Expo project

1. `npx create-expo-app@latest <dir> --template blank-typescript`
2. `cd <dir>`, read `package.json`, note the Expo SDK version.
3. Run `npx expo install` for every architecture dependency (see
   `versions.md` — use `expo install`, not `npm install`).
4. Confirm `tsconfig.json` extends `expo/tsconfig.base` and has `strict: true`.
5. Set `babel.config.js` to `babel-preset-expo` + `react-native-reanimated/plugin`
   (reanimated plugin LAST).

## Step 2 — Root config files

- `app.config.ts` — dynamic config: load `dotenv-flow` only when not in EAS/CI;
  set `name`, `slug`, `scheme`, `owner`, icons, splash, `newArchEnabled: true`,
  iOS `bundleIdentifier`, Android `package` + permissions, `plugins` array
  (`expo-secure-store`, `expo-font`, `expo-localization`, `expo-splash-screen`,
  `expo-notifications`, plus `expo-video` if used), and `extra` (API base URL +
  EAS `projectId`).
- `app.json` — minimal static fields only.
- `eas.json` — `development` / `preview` / `production` / `production-aab`
  profiles, each with `channel` + `env`; `appVersionSource: 'remote'`,
  `production.autoIncrement: true`.
- `.env.example` (committed) and `.env` (gitignored) — document
  `EXPO_PUBLIC_API_BASE_URL`.
- `index.ts` — `registerRootComponent(App)`.

## Step 2a — Expo account & EAS setup

This step depends on the user's "Expo account choice" input.

The skill CANNOT create an Expo account — signup is interactive and
account-bound. It CAN, with the user's account, create the EAS project and
wire the resulting identifiers in.

### If the user chose "link now"

1. Check whether the user is already logged in: `npx eas whoami`.
2. If not logged in, tell the user to run `eas login` themselves (it prompts
   for credentials — the skill must not handle them). If they need a new
   account, direct them to sign up at expo.dev or via `eas login` first, then
   continue.
3. Once logged in, create the EAS project: `npx eas init`. This creates the
   project on Expo's servers and writes/updates the `projectId`.
4. Wire the real values into config:
   - `expo.extra.eas.projectId` ← the project id from `eas init`.
   - `expo.owner` ← the Expo account/organization slug.
   - `expo.updates.url` ← `https://u.expo.dev/<projectId>`.
   - `expo.runtimeVersion` ← `{ policy: 'appVersion' }`.
5. Confirm `eas.json` profiles reference the correct channels.
6. Tell the user that EAS Build / Update / Submit are now usable, and that
   store submission still needs paid Apple ($99/yr) and Google ($25) accounts —
   `rn-build-deployer` covers that.

### If the user chose "defer"

1. Do NOT run `eas login` or `eas init`.
2. In `app.config.ts`, leave `expo.extra.eas.projectId`, `expo.owner`, and
   `expo.updates.url` as clearly-marked placeholders, e.g.:
   ```typescript
   // TODO(expo): run `eas login` then `eas init`, then replace this placeholder
   // with the real projectId and set owner / updates.url accordingly.
   eas: { projectId: 'REPLACE_WITH_EAS_PROJECT_ID' },
   ```
   This is the ONE sanctioned placeholder-TODO exception to the no-`TODO` rule
   (coding-standards R34) — it is surfaced to the user, not buried.
3. In the report and the project `CLAUDE.md`, print the exact follow-up:
   `eas login` → `eas init` → fill in `projectId` / `owner` / `updates.url`.

## Step 3 — Create the 12-folder `src/` tree

Create all 12 folders (see `architecture.md`):
`components/`, `screens/`, `navigation/`, `store/`, `services/`,
`validation/`, `hooks/`, `constants/`, `theme/`, `types/`, `utils/`, `i18n/`.

## Step 4 — Theme layer

- `theme/colors.ts` — `lightColors` + `darkColors` (identical keys, semantic
  names, grouped with comments).
- `theme/ThemeContext.tsx` — `ThemeProvider` + `useTheme()`; persists mode to
  AsyncStorage (`'themeMode'`), default `'light'`, syncs `SystemUI` background;
  `useTheme` throws outside the provider.

## Step 5 — Constants layer

- `constants/spacing.ts` — `xs:4, sm:8, md:16, lg:20, xl:24, xxl:32`.
- `constants/typography.ts` — `fontFamily` + `fontSize` (every size via
  `scaleFont`).
- `constants/api.ts` — `API_BASE_URL` from `expo-constants` `extra`.
- `constants/icons.ts` — icon `require()` maps (`ASSET_ICONS`, `NAVIGATION_ICONS`).
- `constants/ui.ts` — UI constants.
- `constants/index.ts` — barrel re-exporting all of the above.

## Step 6 — Utils layer

- `utils/fontScaling.ts` — `scaleFont`, `getLineHeight`,
  `scaleWithoutAccessibility`, `isLargeTextEnabled`, etc.
- `utils/errorHelpers.ts` — `parseApiErrors`, `setFormErrors`,
  `getGenericErrorMessage`, `isApiFieldError`.
- `utils/fieldMappings.ts` — `BACKEND_TO_FRONTEND_FIELD_MAP` + `mapBackendFieldName`.

## Step 7 — i18n layer

- `i18n/locales/en/common.json` — initial keys (app name, auth strings, tab
  labels, generic API error keys: `apiErrorGeneric`, `apiErrorForbidden`,
  `apiErrorNotFound`, `apiErrorServer`, `apiErrorNetwork`).
- `i18n/index.ts` — `setupI18n()` + default instance.

## Step 8 — Store layer

- `store/slices/authSlice.ts` — auth state, `setCredentials` / `logout`,
  `loadStoredAuth` thunk, `login.matchFulfilled` matcher.
- `store/slices/toastSlice.ts` — toast state, `showToast` / `hideToast`,
  selectors.
- `store/index.ts` — `configureStore` with the slices + API reducers +
  middleware; dispatches `loadStoredAuth()`; exports `RootState` / `AppDispatch`.
- `store/hooks.ts` — typed `useAppDispatch` / `useAppSelector`.

## Step 9 — Services layer

- `services/baseQuery.ts` — `createBaseQueryWithAuth()`: auth header; global 401
  logout (skipping auth endpoints `/auth/login`, `/auth/forgot-password`,
  `/auth/verify-otp`, `/auth/reset-password`); per-status global error toasts
  for 403 / 404 / 5xx / `FETCH_ERROR` / `TIMEOUT_ERROR`, each with its own i18n
  key. See `patterns.md` §1.
- `services/secureStorage.ts` — `storeAuthData` / `getAuthData` / `clearAuthData`
  / `hasAuthData` + last-email helpers. SecureStore for tokens, AsyncStorage for
  non-sensitive prefs.
- `services/authApi.ts` — `authApi` with `login` (and the auth endpoints the app
  needs) + exported hooks + request/response types.
- If the app uses push notifications, `services/notificationService.ts` — a
  non-RTK `<name>Service.ts` module: `configureNotificationHandler()`,
  `registerForPushNotificationsAsync()` (with a `Device.isDevice` check and the
  Android notification channels), local-notification + badge helpers.

## Step 10 — Navigation layer

- `navigation/navigationRef.ts` — global ref + `navigate` / `reset` helpers.
- `navigation/RootNavigator.tsx` — native stack; `RootStackParamList`;
  auth-gated two-group rendering from the `auth` slice;
  `screenOptions={{ headerShown: false }}`.
- `navigation/TabNavigator.tsx` — bottom tabs; `TabParamList`; themed tab bar;
  safe-area insets; localized labels.

## Step 11 — Common components

A starter set in `components/common/` with a barrel `index.ts`:
`Button`, `TextInput`, `PasswordInput`, `LoadingIndicator`, `EmptyState`,
`GlobalToast` (Redux-driven), `FixedToast`. Each follows the 9-section structure
and the comment standard.

## Step 12 — Auth screens

`screens/auth/` with a barrel: `LoginScreen` (working, wired to `authApi` +
`useForm` + Yup), plus placeholders for `ForgotPasswordScreen`,
`VerifyOTPScreen`, `ResetPasswordScreen`, `PasswordResetSuccessScreen` if the
app needs password recovery. A first authenticated screen `screens/home/HomeScreen.tsx`.
Add `validation/auth/loginSchema.ts` (+ barrel).

## Step 13 — Root `App.tsx`

Wire `App.tsx` exactly per `architecture.md`:
side-effect setup → `SplashScreen.preventAutoHideAsync()` → `AppContent` inner
component → parallel init (`Promise.all`: fonts, i18n, asset preload) → hide
splash → fixed provider nesting
(`GestureHandlerRootView > SafeAreaProvider > Provider > ThemeProvider >
StatusBar + AppContent > NavigationContainer(ref) > RootNavigator + GlobalToast`).

## Step 14 — Assets

- `assets/fonts/` — the chosen font family files; load them in `App.tsx`.
- `assets/icons/` — app icon, adaptive icon, splash, favicon, plus any in-app
  icons referenced by `constants/icons.ts`.

## Step 14a — `.gitignore` and CI

- Write `.gitignore` including `node_modules/`, `.expo/`, `dist/`,
  `web-build/`, `expo-env.d.ts`, `*.tsbuildinfo`, `.env*` with a
  `!.env.example` exception, signing files (`*.jks`, `*.p8`, `*.p12`, `*.key`,
  `*.mobileprovision`), `.DS_Store`, and the generated `/ios` and `/android`
  folders.
- Add GitHub Actions workflows under `.github/workflows/` —
  `eas-update-preview.yml` and `eas-update-production.yml` — each triggered on
  `pull_request`, running checkout → Node 20 → `expo/expo-github-action@v8`
  (with the `EXPO_TOKEN` secret) → `npm ci` → `eas update --auto --branch
  <channel>` → PR comment. (See `conventions.md` §16.)

## Step 15 — Project `CLAUDE.md`

Write a `CLAUDE.md` at the project root recording: the stack + versions, the
12-folder architecture, the signature patterns, the commands, and the rule that
this project is built with the `react-native-app-design` skill — so future
sessions stay consistent.

## Step 16 — Verify

- `npx tsc --noEmit` — zero type errors.
- `npx expo-doctor` — resolve every issue.
- `npx expo start` — confirm the app boots to the login screen.
- Hand the project to `rn-pattern-reviewer` for a full standards audit before
  reporting the scaffold complete.

---

## Definition of done

- The app runs and shows the login screen.
- All 12 folders exist and are wired.
- `tsc --noEmit` and `expo-doctor` are clean.
- `rn-pattern-reviewer` returns PASS on the scaffolded shell.
- A project `CLAUDE.md` exists.
