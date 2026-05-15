# Template — Component

Path: `src/components/<group>/<ComponentName>.tsx`
After creating, add it to `src/components/<group>/index.ts`.

Follow the 9-section structure (R23). Every section header is required. Every
function and the component itself carry a doc comment. Theme values only — no
hardcoded colors / sizes / spacing. All visible text via `t()`.

```typescript
// <ComponentName>.tsx
// <One line: what this component is and where it is used.>

// ---- 1. Imports (ordered: React, RN, third-party, local, types, constants) ----
import React, { useState } from 'react';
import { View, Text, StyleSheet, TouchableOpacity, ViewStyle, TextStyle } from 'react-native';
import { useTranslation } from 'react-i18next';
import { useTheme } from '../../theme/ThemeContext';

// ---- 2. Types ----
/**
 * Props for <ComponentName>.
 * @param title  Visible heading text (already localized by the caller, or a key).
 * @param onPress  Optional handler fired when the component is tapped.
 */
interface <ComponentName>Props {
    title: string;
    onPress?: () => void;
}

// ---- 3. Component definition ----
/**
 * <ComponentName> — <what it renders and its single responsibility>.
 * Presentational only; all behaviour is provided via props.
 */
export const <ComponentName>: React.FC<<ComponentName>Props> = ({ title, onPress }) => {
    // ---- 4. Hooks (theme, i18n, redux, local state, navigation) ----
    const { colors, typography, spacing } = useTheme();
    const { t } = useTranslation();
    const [isActive, setIsActive] = useState(false);

    // ---- 5. Effects ----
    // (none — add useEffect blocks here if needed, each with a WHY comment)

    // ---- 6. Event handlers ----
    /** Toggles the active flag, then delegates to the parent handler. */
    const handlePress = (): void => {
        setIsActive((prev) => !prev);
        onPress?.();
    };

    // ---- 7. Dynamic (theme-dependent) styles ----
    // Built per render because theme values can change at runtime and so
    // cannot live in the static StyleSheet below.
    const dynamicStyles: { container: ViewStyle; title: TextStyle } = {
        container: {
            backgroundColor: isActive ? colors.surface : colors.background,
            padding: spacing.md,
        },
        title: {
            color: colors.text,
            fontFamily: typography.fontFamily.regular,
            fontSize: typography.fontSize.md,
        },
    };

    // ---- 8. Render ----
    return (
        <TouchableOpacity
            style={[styles.container, dynamicStyles.container]}
            onPress={handlePress}
            accessibilityRole="button"
            accessibilityLabel={t('<componentNameLabelKey>')}
        >
            <Text style={dynamicStyles.title} numberOfLines={1} ellipsizeMode="tail">
                {title}
            </Text>
        </TouchableOpacity>
    );
};

// ---- 9. Static styles ----
const styles = StyleSheet.create({
    container: {
        borderRadius: 12,
        alignItems: 'center',
        justifyContent: 'center',
    },
});
```

## Common component patterns

- **Images.** Use `expo-image`'s `Image` for network/remote images
  (`contentFit`, `cachePolicy="disk"`, `onLoad`/`onError`); use React Native's
  `Image` for bundled icons (`resizeMode="contain"`, `fadeDuration={0}`,
  `tintColor` for recoloring). Bundled icons come from a `constants/icons.ts`
  map — never `require()` inline.
- **List rows.** A component used as a `FlatList` row whose props include
  object references (an entity) is wrapped in `React.memo`, with a custom
  comparator that returns `true` to SKIP a re-render:
  ```typescript
  export const ItemRow = React.memo(
      ({ item, onPress }: ItemRowProps) => { /* … */ },
      (prev, next) =>
          prev.item.id === next.item.id &&
          prev.item.updatedAt === next.item.updatedAt &&
          prev.onPress === next.onPress,
  );
  ```
- **Memoization.** `useMemo` for expensive per-render formatting; `useCallback`
  for handlers passed to children or used in effect deps.
- **Touchables.** `TouchableOpacity` with `activeOpacity={0.7}` for normal taps,
  `0.8` for prominent buttons; small icon targets add
  `hitSlop={{ top: 10, bottom: 10, left: 10, right: 10 }}`. Use `Pressable` for
  modal-overlay dismissal.
- **Timers.** A countdown/interval is stored in `useRef`, cleared in the
  `useEffect` cleanup, and the ref set to `null` after clearing.
- **Gradients.** An active/gradient state uses `expo-linear-gradient`'s
  `LinearGradient` with `StyleSheet.absoluteFillObject` + a matching
  `borderRadius`, with the text rendered on top.
- **Card with overlay.** A media card uses `borderRadius` + `overflow: 'hidden'`
  on the container, absolutely-positioned image / gradient / content layers,
  and `shadow*` (iOS) + `elevation` (Android) for elevation.
- **Conditional rendering.** Return `null` to hide; never `display: 'none'`.
- **Sub-components & helpers.** Pure helper functions are declared before the
  JSX. A small sub-component used only by this file may live in the same file,
  defined above the main component.

For a list-rendering component see also `data-patterns.md` §1.

Checklist before handing to the reviewer:
- Imports grouped & ordered (R07).
- All 9 section headers present (R23).
- Doc comment on the component and every function (R28).
- No hardcoded color/size/spacing — theme only, R35 exceptions only (R35–R37).
- Style objects typed `ViewStyle`/`TextStyle`/`ImageStyle`.
- `expo-image` for network images, RN `Image` for icons.
- List-row components memoized with a custom comparator.
- Interactive elements have `accessibilityRole` + localized `accessibilityLabel`.
- Every visible string and `accessibilityLabel` via `t()` (R61, R83).
- Added to the folder barrel `index.ts` (conventions §1).
