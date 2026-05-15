# Architecture — the 12-folder `src/` layout

Every app this skill produces uses the SAME structure. Do not add top-level
folders, do not rename these, do not move responsibilities between them.

```
<project-root>/
├── App.tsx                  Root component: providers + app initialization
├── index.ts                 registerRootComponent(App)
├── app.config.ts            Expo config (dynamic, reads env)
├── app.json                 Static Expo fields (minimal)
├── eas.json                 EAS build / submit profiles
├── babel.config.js          babel-preset-expo + reanimated plugin
├── tsconfig.json            extends expo/tsconfig.base, strict: true
├── .env.example             Documented env vars (committed)
├── .env                     Local env values (gitignored)
├── assets/
│   ├── fonts/               Font files (.ttf)
│   └── icons/               App icon, splash, in-app icons
└── src/
    ├── components/          Reusable UI, grouped by feature domain
    ├── screens/             Full screens, grouped by feature domain
    ├── navigation/          Navigators, param lists, navigationRef
    ├── store/               Redux store, slices, typed hooks
    ├── services/            RTK Query API services + baseQuery + storage
    ├── validation/          Yup schema factories, grouped by feature
    ├── hooks/               Custom hooks (shared + feature-grouped)
    ├── constants/           Static values, grouped by feature, barrel index
    ├── theme/               ThemeContext + colors
    ├── types/               Shared TypeScript types, grouped by feature
    ├── utils/               Pure helper functions
    └── i18n/                i18next setup + locale JSON
```

---

## What goes in each folder

### `src/components/`
Reusable UI building blocks. **Grouped by feature domain**, plus a `common/`
folder for cross-feature primitives.

```
components/
├── common/          Button, TextInput, Dropdown, EmptyState, Toast, FAB …
│   └── index.ts     Barrel — re-exports every component in the folder
├── <feature>/       e.g. auth/, home/, profile/
│   └── index.ts     Barrel
```

- Every feature folder AND `common/` MUST have an `index.ts` barrel that
  re-exports its components.
- A component is "common" only if two or more features use it. Otherwise it
  lives in its feature folder.
- Components are presentational. Business logic belongs in hooks.

### `src/screens/`
Full-screen components rendered by a navigator. Grouped by feature domain, each
group with an `index.ts` barrel.

```
screens/
├── auth/
│   ├── LoginScreen.tsx
│   ├── ForgotPasswordScreen.tsx
│   └── index.ts
├── home/
│   ├── HomeScreen.tsx
│   └── index.ts
```

- One screen per file. File name ends in `Screen.tsx`.
- Screens are thin: they wire hooks + components together. Heavy logic goes to
  `src/hooks/`. Heavy validation goes to `src/validation/`.
- Screens use `SafeAreaView` (from `react-native-safe-area-context`) and, for
  any screen with text inputs, `KeyboardAvoidingView`.

### `src/navigation/`
```
navigation/
├── RootNavigator.tsx        Native stack; switches auth flow vs main app
├── TabNavigator.tsx         Bottom tabs for the authenticated app
├── <X>TopTabsNavigator.tsx  Material top tabs where needed
└── navigationRef.ts         Global ref + navigate()/reset() helpers
```

- Each navigator declares a typed `ParamList` and exports it.
- `RootNavigator` decides authenticated vs unauthenticated by reading the
  `auth` slice — never a local boolean.
- `navigationRef` enables navigation from outside React (thunks, error
  handlers). See `patterns.md`.

### `src/store/`
```
store/
├── index.ts             configureStore: slice reducers + API reducers + middleware
├── hooks.ts             Typed useAppDispatch / useAppSelector
└── slices/
    ├── authSlice.ts
    └── <feature>Slice.ts
```

- One slice per feature domain.
- `index.ts` registers every slice reducer, every API `reducerPath`, and
  concatenates every API `middleware`.
- Components ALWAYS use `useAppDispatch` / `useAppSelector` from `hooks.ts`,
  never the raw react-redux hooks.

### `src/services/`
```
services/
├── baseQuery.ts         Shared RTK Query base query: auth header + global errors
├── <feature>Api.ts      One RTK Query createApi per feature domain
├── secureStorage.ts     expo-secure-store + AsyncStorage wrappers
└── <name>Service.ts     Non-RTK side-effect modules (e.g. notification handlers)
```

- Every `*Api.ts` uses `createBaseQueryWithAuth()` from `baseQuery.ts`.
- Request and response types are declared in the API file and exported.
- Sensitive data (tokens) → expo-secure-store. Non-sensitive prefs → AsyncStorage.

### `src/validation/`
```
validation/
├── <feature>/
│   ├── <name>Schema.ts   Exports createXSchema(t) + the XFormData interface
│   └── index.ts          Barrel
```

- Every schema is a FACTORY: `createXSchema(t)` returns a Yup object. The factory
  takes the i18n `t()` function so messages are translatable.
- Each schema file also exports the matching `FormData` TypeScript interface.

### `src/hooks/`
Custom hooks. Shared hooks live at the top level; feature-specific hooks are
grouped in a subfolder with an `index.ts` barrel.

```
hooks/
├── useSomethingShared.ts
└── <feature>/
    ├── use<Feature>Forms.ts      Form state (react-hook-form setup)
    ├── use<Feature>Handlers.ts   Business logic (submit, transform, navigate)
    └── index.ts
```

- Forms use the **two-hook pattern**: a `Forms` hook for state and a `Handlers`
  hook for logic. See `patterns.md`.

### `src/constants/`
```
constants/
├── api.ts            API base URL + fixed IDs
├── spacing.ts        Spacing scale
├── typography.ts     Font families + sizes
├── icons.ts          Icon asset map
├── ui.ts             UI constants
├── <feature>.ts      Feature-specific constants
└── index.ts          Barrel — re-exports every constants file
```

- `index.ts` re-exports all constant files so imports can come from
  `'../../constants'`.

### `src/theme/`
```
theme/
├── ThemeContext.tsx   ThemeProvider + useTheme(); persists mode; light/dark
└── colors.ts          lightColors + darkColors (identical key sets)
```

- `useTheme()` returns `{ mode, colors, typography, spacing, toggle }`.
- `lightColors` and `darkColors` MUST have identical keys.

### `src/types/`
Shared TypeScript types grouped by feature: `camera.ts`, `profile.ts`, etc.
Types tied to a single API file may live in that API file instead — but anything
used across files belongs here.

### `src/utils/`
Pure, side-effect-free helper functions: formatters, mappers, validators,
small utilities. No React, no Redux, no navigation.

### `src/i18n/`
```
i18n/
├── index.ts                  setupI18n() — initializes i18next
└── locales/<lang>/common.json   Translation strings
```

---

## App initialization (`App.tsx`)

`App.tsx` follows a fixed shape:

1. Side-effect module setup at module scope (e.g. notification handler config).
2. `SplashScreen.preventAutoHideAsync()` at module scope.
3. An inner `AppContent` component so app-wide hooks can run inside `<Provider>`.
4. The root `App` component runs initialization tasks IN PARALLEL with
   `Promise.all` (fonts, i18n, asset preload), then hides the splash screen.
5. The provider nesting order is fixed:

```
GestureHandlerRootView
└── SafeAreaProvider
    └── Provider (redux)
        └── ThemeProvider
            └── StatusBar + AppContent
                └── NavigationContainer (ref=navigationRef)
                    └── RootNavigator + GlobalToast
```

Never change this nesting order — each layer depends on the one above it.

---

## Dependency direction

Allowed import direction (top may import bottom; never the reverse):

```
screens  →  components, hooks, services, store, validation, navigation, theme, constants, types, utils
hooks    →  services, store, navigation, validation, constants, types, utils
components → theme, constants, store (typed hooks), utils, types
services →  store (types only), constants, navigation (ref), utils
store    →  services, navigation (ref), types
```

- `utils/` imports nothing from the app except other utils and types.
- `theme/`, `constants/`, `types/` are leaf layers.
- Circular imports are forbidden. Where Redux and services would form a cycle
  (e.g. `baseQuery` needs the store), use a dynamic `import()` inside the
  function — see `patterns.md`.
