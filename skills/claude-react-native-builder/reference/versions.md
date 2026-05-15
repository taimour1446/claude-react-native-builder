# Versions — policy and known-good baseline

## Version policy

This skill targets **the latest stable Expo SDK at scaffold time**, but never
blindly. The architecture (RTK Query, React Navigation, react-hook-form + Yup,
StyleSheet theming) is stable across SDK versions — only the package versions
move.

The `rn-scaffolder` agent follows this sequence:

1. Scaffold with `npx create-expo-app@latest` — this always pulls the current
   Expo SDK and a matching RN/React version.
2. Read the resulting `package.json` to learn the actual SDK version.
3. Add the architecture's extra dependencies using **`npx expo install <pkg>`**
   (NOT `npm install`). `expo install` picks the version compatible with the
   installed SDK — this is what keeps the dependency set coherent.
4. Run `npx expo-doctor` and resolve anything it flags before declaring the
   scaffold done.
5. If the latest SDK proves unstable (a core library broken, doctor failing in
   a way that cannot be resolved), fall back to the **known-good baseline**
   below and inform the user.

Never hand-pin a version that `expo install` would otherwise choose. Only pin
when `expo install` does not manage a package, and then prefer that package's
latest stable.

## Known-good baseline

This baseline is verified to work together. Use it as the fallback, and as the
reference for what each dependency is for.

| Package | Baseline version | Purpose |
| --- | --- | --- |
| `expo` | `~54.0.x` | Framework |
| `react` | `19.1.0` | React |
| `react-native` | `0.81.x` | RN core |
| `typescript` | `~5.9.x` (dev) | TypeScript |
| `@types/react` | `~19.1.x` (dev) | Types |
| `@reduxjs/toolkit` | `^2.9.x` | State + RTK Query |
| `react-redux` | `^9.2.x` | React bindings for Redux |
| `@react-navigation/native` | `^7.1.x` | Navigation core |
| `@react-navigation/native-stack` | `^7.3.x` | Native stack |
| `@react-navigation/bottom-tabs` | `^7.4.x` | Bottom tabs |
| `@react-navigation/material-top-tabs` | `^7.4.x` | Top tabs |
| `react-native-screens` | `~4.16.x` | Navigation native deps |
| `react-native-safe-area-context` | `~5.6.x` | Safe area insets |
| `react-native-gesture-handler` | `~2.28.x` | Gestures |
| `react-native-reanimated` | `~4.1.x` | Animations |
| `react-native-pager-view` | `^6.9.x` | Top-tab pager |
| `react-native-tab-view` | `^4.2.x` | Tab view |
| `react-hook-form` | `^7.65.x` | Forms |
| `@hookform/resolvers` | `^5.2.x` | Yup resolver bridge |
| `yup` | `^1.7.x` | Validation |
| `i18next` | `^25.6.x` | i18n core |
| `react-i18next` | `^16.1.x` | React bindings for i18n |
| `@react-native-async-storage/async-storage` | `^2.2.x` | Non-secure storage |
| `expo-secure-store` | (SDK-managed) | Secure storage |
| `expo-notifications` | (SDK-managed) | Push notifications |
| `expo-updates` | (SDK-managed) | OTA updates |
| `expo-font` | (SDK-managed) | Custom fonts |
| `expo-localization` | (SDK-managed) | Device locale |
| `expo-splash-screen` | (SDK-managed) | Splash screen control |
| `expo-status-bar` | (SDK-managed) | Status bar |
| `expo-system-ui` | (SDK-managed) | Native UI background |
| `expo-constants` | (SDK-managed) | Runtime config access |
| `expo-linear-gradient` | (SDK-managed) | Gradients |
| `expo-image` | (SDK-managed) | Performant images |
| `@expo/vector-icons` | (SDK-managed) | Icon set |
| `dotenv-flow` | `^4.1.x` | `.env` loading in `app.config.ts` |

Notes:

- "(SDK-managed)" → always install with `npx expo install`; the version is
  dictated by the Expo SDK.
- Optional, install only when the app needs them: `expo-video` (video),
  `expo-crypto` / `expo-device` (device + crypto), `react-native-international-phone-number`
  (phone fields), `@react-native-community/datetimepicker`, `@react-native-picker/picker`,
  `@react-native-community/slider`, `react-native-render-html`.
- Config: `tsconfig.json` extends `expo/tsconfig.base` with `strict: true`.
  `babel.config.js` uses `babel-preset-expo` + the `react-native-reanimated/plugin`
  (the reanimated plugin MUST be last in the plugins array).
- New Architecture (`newArchEnabled: true`) is on. If a chosen optional library
  is not New-Arch compatible, surface it to the user rather than disabling New
  Architecture silently.
