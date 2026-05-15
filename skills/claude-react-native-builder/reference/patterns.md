# Signature Patterns

These are the fixed patterns of this architecture. Reuse them verbatim — never
invent an alternative. Every snippet here is the canonical form; copy its shape.

---

## 1. Global base query (auth + global errors)

`src/services/baseQuery.ts` exports `createBaseQueryWithAuth()`. Every `*Api.ts`
uses it. It does three things so individual code never has to:

1. Attaches the `Bearer` token from the `auth` slice to every request.
2. On `401` (except auth endpoints), dispatches `logout()` and resets navigation
   to `Login`.
3. On `403/404/5xx/network` errors, dispatches a global error toast.

```typescript
// src/services/baseQuery.ts
import { fetchBaseQuery } from '@reduxjs/toolkit/query/react';
import type { BaseQueryFn, FetchArgs, FetchBaseQueryError } from '@reduxjs/toolkit/query/react';
import type { RootState } from '../store';
import { API_BASE_URL } from '../constants';
import i18next from 'i18next';

/**
 * Builds an RTK Query base query that:
 *  - injects the auth token on every request,
 *  - logs the user out + redirects on a 401 (auth failure),
 *  - shows a global toast for other server/network errors.
 * Individual API services and components therefore never handle auth or
 * generic errors themselves.
 */
export const createBaseQueryWithAuth = (): BaseQueryFn<
    string | FetchArgs,
    unknown,
    FetchBaseQueryError
> => {
    const baseQuery = fetchBaseQuery({
        baseUrl: API_BASE_URL,
        prepareHeaders: (headers, { getState }) => {
            // Pull the token from the auth slice and attach it. If there is no
            // token (logged out), the header is simply omitted.
            const accessToken = (getState() as RootState).auth.accessToken;
            if (accessToken) {
                headers.set('authorization', `Bearer ${accessToken}`);
            }
            headers.set('content-type', 'application/json');
            return headers;
        },
    });

    return async (args, api, extraOptions) => {
        const result = await baseQuery(args, api, extraOptions);

        // --- Global 401 handling ---
        // Auth endpoints must handle their own 401 (e.g. "wrong password"),
        // so they are excluded from the global logout behaviour.
        if (result.error && result.error.status === 401) {
            const authEndpointsToSkip = ['/auth/login', '/auth/forgot-password'];
            const isAuthRequest = authEndpointsToSkip.some((endpoint) =>
                (typeof args === 'string' && args.includes(endpoint)) ||
                (typeof args === 'object' && 'url' in args && args.url.includes(endpoint)),
            );
            if (!isAuthRequest) {
                // Dynamic import breaks the store <-> baseQuery circular dependency.
                const { store } = await import('../store');
                const { logout } = await import('../store/slices/authSlice');
                store.dispatch(logout());
                const { navigationRef } = await import('../navigation/navigationRef');
                if (navigationRef.isReady()) {
                    navigationRef.reset({ index: 0, routes: [{ name: 'Login' }] });
                }
            }
        }

        // --- Global non-401, non-400 error toasts ---
        // 400 is left to screens (field-level form errors). Everything else
        // surfaces as a localized toast.
        if (result.error && result.error.status !== 401 && result.error.status !== 400) {
            const { store } = await import('../store');
            const { showToast } = await import('../store/slices/toastSlice');
            store.dispatch(showToast({
                message: i18next.t('apiErrorGeneric'),
                type: 'error',
                duration: 5000,
            }));
        }

        return result;
    };
};
```

**Rule recap:** never add auth headers manually; never catch `401` in a
component; only handle `400` field errors locally.

---

## 2. RTK Query API service

One `createApi` per feature. Types declared and exported in the same file.

```typescript
// src/services/<feature>Api.ts
import { createApi } from '@reduxjs/toolkit/query/react';
import { createBaseQueryWithAuth } from './baseQuery';

/** Request body for the login endpoint. */
export type LoginRequest = { email: string; password: string };

/** Successful login response envelope returned by the backend. */
export type LoginResponse = {
    ok: boolean;
    data: { accessToken: string; user: { id: string; email: string } };
};

/**
 * Authentication API service. All endpoints share the global base query, so
 * the token is attached and global errors are handled automatically.
 */
export const authApi = createApi({
    reducerPath: 'authApi',
    baseQuery: createBaseQueryWithAuth(),
    endpoints: (builder) => ({
        // POST /auth/login — exchanges credentials for a token + user.
        login: builder.mutation<LoginResponse, LoginRequest>({
            query: (body) => ({ url: '/auth/login', method: 'POST', body }),
        }),
    }),
});

// RTK Query auto-generates a hook per endpoint; export it for screens to use.
export const { useLoginMutation } = authApi;
```

Every new API service must also be registered in `src/store/index.ts` (reducer
+ middleware).

---

## 3. Redux slice

```typescript
// src/store/slices/<feature>Slice.ts
import { createSlice, PayloadAction } from '@reduxjs/toolkit';

/** Shape of this feature's state. */
type FeatureState = {
    items: string[];
    isLoading: boolean;
};

const initialState: FeatureState = {
    items: [],
    isLoading: false,
};

/** Slice owning the <feature> domain state. */
const featureSlice = createSlice({
    name: 'feature',
    initialState,
    reducers: {
        /** Replaces the item list with a new set. */
        setItems: (state, action: PayloadAction<string[]>) => {
            state.items = action.payload;
        },
    },
});

export const { setItems } = featureSlice.actions;
export default featureSlice.reducer;
```

For server-driven state, sync the slice from RTK Query results with
`extraReducers` + `addMatcher(api.endpoints.x.matchFulfilled, …)` — see how the
auth slice reacts to `login.matchFulfilled`. **Ordering rule:** all `addCase`
calls come before any `addMatcher` calls.

---

## 4. Typed Redux hooks

```typescript
// src/store/hooks.ts
import { TypedUseSelectorHook, useDispatch, useSelector } from 'react-redux';
import type { RootState, AppDispatch } from './index';

/** Dispatch hook typed with the app's dispatch type. */
export const useAppDispatch = () => useDispatch<AppDispatch>();
/** Selector hook typed with the app's root state. */
export const useAppSelector: TypedUseSelectorHook<RootState> = useSelector;
```

Always use these — never raw `useDispatch` / `useSelector`.

---

## 5. Navigation ref (navigate from outside React)

```typescript
// src/navigation/navigationRef.ts
import { createNavigationContainerRef } from '@react-navigation/native';
import type { RootStackParamList } from './RootNavigator';

/**
 * Global navigation reference. Lets non-component code (Redux thunks, API
 * error handlers, services) trigger navigation, since they cannot use the
 * useNavigation hook.
 */
export const navigationRef = createNavigationContainerRef<RootStackParamList>();
```

In components, prefer the typed `navigation` prop. Use `navigationRef` ONLY
outside React. Always guard with `navigationRef.isReady()`.

---

## 6. Global toast

One `<GlobalToast />` is mounted once in `App.tsx`. Show a toast from anywhere:

```typescript
import { useAppDispatch } from '../store/hooks';
import { showToast } from '../store/slices/toastSlice';
import { useTranslation } from 'react-i18next';

const dispatch = useAppDispatch();
const { t } = useTranslation();

// Message is localized — never a raw string.
dispatch(showToast({
    message: t('cameraAddedSuccess'),
    type: 'success',          // 'success' | 'error' | 'info' | 'warning'
    duration: 5000,
}));
```

Never build a local toast/snackbar component.

---

## 7. Theme usage

```typescript
import { useTheme } from '../theme/ThemeContext';

const { colors, typography, spacing, mode, toggle } = useTheme();
```

- Colors → `colors.*` (semantic keys). Spacing → `spacing.*`. Type →
  `typography.fontFamily.*` / `typography.fontSize.*`.
- Theme-dependent styles are built inside the component (section 7), NOT in the
  module-scope `StyleSheet`.
- The `useTheme` hook throws outside `ThemeProvider` — keep that guard.

---

## 8. Localized text

```typescript
import { useTranslation } from 'react-i18next';

const { t } = useTranslation();

// Static text
<Text style={{ color: colors.text }}>{t('home')}</Text>

// Interpolated text — locale JSON: "welcome": "Welcome, {{name}}!"
<Text style={{ color: colors.text }}>{t('welcome', { name: user.name })}</Text>
```

Every new key is added to `src/i18n/locales/<lang>/common.json` in the same
change. Non-React modules use the `i18next` instance directly:
`import i18next from 'i18next'; i18next.t('apiErrorGeneric');`.

---

## 9. Two-hook form pattern

Non-trivial forms split into a state hook and a logic hook so the screen stays
thin.

```typescript
// src/hooks/<feature>/use<Feature>Forms.ts
/**
 * Owns the react-hook-form instance for the <feature> form: resolver,
 * default values, and mode. Returns the form object for the screen/handlers.
 */
export const useFeatureForms = () => {
    const { t } = useTranslation();
    const form = useForm<FeatureFormData>({
        resolver: yupResolver(createFeatureSchema(t)),
        mode: 'onChange',                       // immediate validation feedback
        defaultValues: { name: '' },
    });
    return { form };
};
```

```typescript
// src/hooks/<feature>/use<Feature>Handlers.ts
/**
 * Owns the business logic for the <feature> form: submitting to the API,
 * transforming data, mapping backend errors onto fields, toasts, navigation.
 */
export const useFeatureHandlers = (form: UseFormReturn<FeatureFormData>) => {
    const dispatch = useAppDispatch();
    const { t } = useTranslation();
    const [createItem, { isLoading }] = useCreateItemMutation();

    /** Submits the form; surfaces success via toast, field errors via setError. */
    const handleSubmit = form.handleSubmit(async (values) => {
        try {
            await createItem(values).unwrap();
            dispatch(showToast({ message: t('itemCreated'), type: 'success' }));
        } catch (error) {
            // 400 field errors are mapped here; other errors are handled
            // globally by baseQuery (see pattern 1).
            applyBackendFieldErrors(error, form, t);
        }
    });

    return { handleSubmit, isLoading };
};
```

The screen just calls both hooks and renders `<Controller>` fields.

---

## 10. Validation schema factory

```typescript
// src/validation/<feature>/<name>Schema.ts
import * as yup from 'yup';

/** Data shape of the <name> form. */
export interface NameFormData {
    email: string;
}

/**
 * Builds the Yup schema for the <name> form. Takes the i18n `t` function so
 * every validation message is localized.
 */
export const createNameSchema = (t: (key: string) => string) => {
    return yup.object().shape({
        email: yup
            .string()
            .trim()
            .lowercase()
            .required(t('emailRequired'))
            .email(t('emailInvalid')),
    });
};
```

---

## 11. Secure vs. non-secure storage

- Tokens / user identity → `expo-secure-store` via `src/services/secureStorage.ts`.
- Non-sensitive prefs (theme mode, last email) → `AsyncStorage`.
- Storage wrapper functions catch errors; cleanup functions (logout) catch and
  log but never re-throw.

---

## 12. Performance patterns

- **Lazy-load heavy native components.** Components like `expo-video`'s
  `VideoView` block the JS thread on init. Wrap them in a component that, after
  mount, waits ~100ms via `setTimeout` and then dynamically `import()`s the
  heavy module, showing a `LoadingIndicator` until it resolves:
  ```typescript
  useEffect(() => {
      // Delay one beat so the surrounding screen paints before the heavy
      // native module is pulled in and initialized.
      const timer = setTimeout(() => {
          import('expo-video').then((mod) => setVideoView(() => mod.VideoView));
      }, 100);
      return () => clearTimeout(timer);
  }, []);
  ```
- **Conditional mounting.** Don't render a component that occupies layout space
  before it is functional — mount it only when ready, rather than rendering it
  disabled.
- **Last-item margin.** In lists, pass `isLast` to the row and drop its bottom
  margin so there is no dead space before a tab bar.
- **Conditional `flexGrow`.** Apply `flexGrow: 1` to a `ScrollView`'s content
  only for empty/loading centering states, not always.
- **Parallel init.** App startup tasks (fonts, i18n, asset preload) run together
  with `Promise.all`, not sequentially.

---

## 13. Breaking circular imports

When module A and module B would import each other (classically the Redux store
and `baseQuery`), do NOT import at module scope. Use a dynamic `import()` inside
the function that needs it — see pattern 1. This is the ONLY sanctioned way to
resolve such a cycle.

---

## 14. Auth-gated root navigator

`RootNavigator` renders two mutually exclusive screen groups, chosen from the
`auth` slice. There is no separate "is logged in" boolean and no manual
navigation between the two worlds — changing the slice changes the tree.

```typescript
/** Root native-stack navigator. Switches between the auth flow and the
 *  authenticated app based purely on Redux auth state. */
export default function RootNavigator() {
    const { user, accessToken, isLoading } = useAppSelector((s) => s.auth);

    // Authenticated only when BOTH a user and a token exist.
    const isAuthenticated = !!(user && accessToken);

    // While stored auth is being loaded, render nothing (splash stays up).
    if (isLoading) {
        return null;
    }

    return (
        <Stack.Navigator screenOptions={{ headerShown: false }}>
            {isAuthenticated ? (
                <>
                    <Stack.Screen name="Tabs" component={TabNavigator} />
                    {/* …authenticated-only screens… */}
                </>
            ) : (
                <>
                    <Stack.Screen name="Login" component={LoginScreen} />
                    {/* …auth-flow screens… */}
                </>
            )}
        </Stack.Navigator>
    );
}
```

`screenOptions={{ headerShown: false }}` is fixed — the app draws its own
headers as components.

---

## 15. Bottom tab navigator

```typescript
/** Bottom tab navigator for the authenticated app. Tab icons use active /
 *  inactive image variants; colors and safe-area insets come from the theme. */
export default function TabNavigator() {
    const { t } = useTranslation();
    const { colors } = useTheme();
    const insets = useSafeAreaInsets();

    return (
        <Tab.Navigator
            screenOptions={({ route }) => ({
                headerShown: false,
                // Pick the active or inactive icon for the current route.
                tabBarIcon: ({ focused }) => (
                    <Image
                        source={pickIcon(route.name, focused)}
                        style={{ width: 18, height: 18 }}
                        resizeMode="contain"
                    />
                ),
                tabBarActiveTintColor: colors.text,
                tabBarInactiveTintColor: colors.inactiveText,
                tabBarStyle: {
                    backgroundColor: colors.background,
                    borderTopColor: colors.border,
                    // Respect the device's bottom safe-area inset.
                    paddingBottom: Math.max(insets.bottom, 8),
                    height: 60 + Math.max(insets.bottom, 0),
                },
            })}
        >
            {/* Tab labels are localized via t(). */}
            <Tab.Screen name="Home" component={HomeScreen}
                options={{ tabBarLabel: t('home') }} />
        </Tab.Navigator>
    );
}
```

---

## 16. API error parsing (`utils/errorHelpers.ts`)

The standard 400 / field-error flow. Reuse these helpers — do not re-parse
errors ad hoc in screens.

```typescript
// In a form submit handler's catch block:
import { parseApiErrors, setFormErrors, getGenericErrorMessage } from '../utils/errorHelpers';

try {
    await createItem(values).unwrap();
} catch (error) {
    // parseApiErrors returns per-field messages (backend names already mapped
    // to frontend names) for a 400 field-validation error, or null otherwise.
    const fieldErrors = parseApiErrors(error);
    if (fieldErrors) {
        setFormErrors(form.setError, fieldErrors);   // push onto the form
    } else {
        // Not a field error — show a single localized message.
        setErrorMessage(getGenericErrorMessage(error, t('loginFailed')));
    }
}
```

The backend error envelope is `{ ok: false, error: { code, message } }` where
`message` can be a `string`, `string[]`, or `Record<field, string|string[]>`.
`errorHelpers.ts` normalizes all three shapes.

---

## 17. Backend ↔ frontend field mapping

`utils/fieldMappings.ts` holds `BACKEND_TO_FRONTEND_FIELD_MAP` and
`mapBackendFieldName()`. When the backend's field name differs from the form's
field name, add an entry so 400 errors land on the right `<Controller>`:

```typescript
export const BACKEND_TO_FRONTEND_FIELD_MAP: Record<string, string> = {
    'connectionMode': 'connectionType',
    'external_id': 'externalId',
};
```

`parseApiErrors` calls `mapBackendFieldName()` automatically — you only maintain
the map.

---

## 18. Font / text scaling (`utils/fontScaling.ts`)

Text must scale with the OS accessibility setting AND adapt to screen size.
Never put raw font numbers in code — they flow through these utilities:

- `scaleFont(size)` — builds every entry of the `typography.fontSize` scale;
  combines OS font scale (clamped) with a moderate screen-width factor.
- `getLineHeight(fontSize, ratio)` — the only way to set `lineHeight`.
- `scaleWithoutAccessibility(size)` — for icon/decorative text that must NOT
  grow with accessibility settings.
- `isLargeTextEnabled()` — for conditional layout when large text is on.

Feature code consumes `typography.*` from the theme; it does not call
`scaleFont` directly (the typography constant already did).

---

## 19. Empty / loading / error UI states

Lists and data screens always handle three non-happy states explicitly:

- **Loading** → a `LoadingIndicator`, with the `ScrollView` content centered via
  conditional `flexGrow` (pattern 12).
- **Empty** → the shared `EmptyState` component (localized message + optional
  action button).
- **Error** → handled globally by `baseQuery` toasts (403/404/5xx) or, for 400,
  on the form.

Never leave a blank screen for loading or empty.
