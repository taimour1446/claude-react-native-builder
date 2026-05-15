# Template — Screen

Path: `src/screens/<group>/<Name>Screen.tsx`
After creating: add to `src/screens/<group>/index.ts`, register the route in the
relevant navigator's `ParamList` + `<Stack.Screen>` / `<Tab.Screen>`.

Screens are **thin**: hooks + components + `<Controller>` fields. Heavy logic
belongs in `src/hooks/`. Screens use `export default`.

```typescript
// <Name>Screen.tsx
// <One line: what this screen shows and its place in the flow.>

// ---- 1. Imports ----
import React from 'react';
import { View, Text, StyleSheet, KeyboardAvoidingView, Platform, ScrollView } from 'react-native';
import { SafeAreaView } from 'react-native-safe-area-context';
import { useTranslation } from 'react-i18next';
import type { NativeStackScreenProps } from '@react-navigation/native-stack';
import { useTheme } from '../../theme/ThemeContext';
import { Button } from '../../components/common';
import type { RootStackParamList } from '../../navigation/RootNavigator';

// ---- 2. Types ----
/** Navigation + route props for <Name>Screen. */
type Props = NativeStackScreenProps<RootStackParamList, '<Name>'>;

// ---- 3. Component definition ----
/**
 * <Name>Screen — <what the user does on this screen>.
 * Composes hooks and components; contains no business logic itself.
 */
export default function <Name>Screen({ navigation, route }: Props) {
    // ---- 4. Hooks ----
    const { colors, spacing } = useTheme();
    const { t } = useTranslation();
    // For a form screen, pull state + logic from the two feature hooks:
    // const { form } = use<Feature>Forms();
    // const { handleSubmit, isLoading } = use<Feature>Handlers(form);

    // ---- 5. Effects ----
    // (none — add here if needed, each with a WHY comment)

    // ---- 6. Event handlers ----
    /** Navigates back to the previous screen. */
    const handleBack = (): void => {
        navigation.goBack();
    };

    // ---- 7. Dynamic styles ----
    const dynamicStyles = {
        container: { backgroundColor: colors.background },
        content: { padding: spacing.lg },
    };

    // ---- 8. Render ----
    return (
        // `edges` controls which safe-area sides are insetted. Omit it for all
        // sides; pass ['top'] when a bottom sheet / tab content must reach the
        // bottom edge; ['top','left','right'] when only the bottom is free.
        <SafeAreaView style={[styles.safeArea, dynamicStyles.container]}>
            {/* KeyboardAvoidingView is required on any screen with text inputs. */}
            <KeyboardAvoidingView
                style={styles.flex}
                behavior={Platform.OS === 'ios' ? 'padding' : 'height'}
                // Offset accounts for a header above the inputs; tune per screen.
                keyboardVerticalOffset={Platform.OS === 'ios' ? 0 : 20}
            >
                <ScrollView
                    contentContainerStyle={dynamicStyles.content}
                    // Lets a tap on a button register without first dismissing
                    // the keyboard — important on form screens.
                    keyboardShouldPersistTaps="handled"
                >
                    <Text style={{ color: colors.text }}>{t('<screenTitleKey>')}</Text>
                    <Button title={t('back')} onPress={handleBack} />
                </ScrollView>
            </KeyboardAvoidingView>
        </SafeAreaView>
    );
}

// ---- 9. Static styles ----
const styles = StyleSheet.create({
    safeArea: {
        flex: 1,
    },
    flex: {
        flex: 1,
    },
});
```

## Form submit button gating

On a form screen, the submit `Button` is gated on form state, and its variant
mirrors the gate so it visually reflects whether it can be pressed:

```typescript
const { formState: { isValid, isDirty } } = form;

<Button
    title={isSubmitting ? t('submitting') : t('submit')}
    onPress={handleSubmit}
    disabled={!isValid || !isDirty || isSubmitting}
    variant={isValid && isDirty && !isSubmitting ? 'active' : 'inactive'}
/>
```

## Other screen patterns

- **Destructive actions** use `Alert.alert` with a `cancel` option and a
  `destructive`-styled confirm; the confirm handler is async. Set an
  `isDeleted` flag before the call (and `{ skip: isDeleted }` on the screen's
  query) so a now-deleted entity is not re-fetched; roll the flag back on error.
- **In-screen tabs** use `useState` for the active tab + a `switch` in a
  `renderTabContent()` helper; a child tab can call an `onSuccess` callback to
  switch the active tab.
- **Headers** are per-screen custom components (the navigators set
  `headerShown: false`) — not the React Navigation header.

Checklist:
- `export default function` named `<Name>Screen` (R26).
- Props typed via `NativeStackScreenProps` / tab equivalent.
- Root is `SafeAreaView` with the correct `edges`; inputs →
  `KeyboardAvoidingView` with `keyboardVerticalOffset` (conventions §8).
- Form ScrollViews set `keyboardShouldPersistTaps="handled"`.
- Submit button gated on `isValid`/`isDirty`/loading, variant mirrors it.
- All visible text via `t()` (R61).
- Route added to the navigator `ParamList` + screen registered.
- Added to the screens barrel.
- Loading / empty / error states handled if it shows data (patterns §19).
