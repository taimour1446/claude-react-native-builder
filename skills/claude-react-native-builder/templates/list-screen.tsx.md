# Template — List screen

Path: `src/screens/<group>/<Name>Screen.tsx`

A data-list screen: fetches via RTK Query, syncs into a slice, renders a
`FlatList`, handles all four loading states, and supports infinite scroll.
Read `data-patterns.md` (§1–§4) alongside this.

```typescript
// <Name>Screen.tsx
// List screen for <feature> — fetches, paginates, and renders <feature> items.

// ---- 1. Imports ----
import React, { useState, useEffect, useCallback } from 'react';
import { FlatList, View, StyleSheet, RefreshControl, type ListRenderItem } from 'react-native';
import { SafeAreaView } from 'react-native-safe-area-context';
import { useFocusEffect } from '@react-navigation/native';
import { useTranslation } from 'react-i18next';
import type { NativeStackScreenProps } from '@react-navigation/native-stack';
import { useTheme } from '../../theme/ThemeContext';
import { useAppDispatch, useAppSelector } from '../../store/hooks';
import { LoadingIndicator, EmptyState } from '../../components/common';
import { useGet<Feature>ListQuery } from '../../services/<feature>Api';
import { set<Feature>Items } from '../../store/slices/<feature>Slice';
import type { RootStackParamList } from '../../navigation/RootNavigator';

// ---- 2. Types ----
type Props = NativeStackScreenProps<RootStackParamList, '<Name>'>;

// ---- 3. Component ----
/** <Name>Screen — shows the list of <feature> items. */
export default function <Name>Screen({ navigation }: Props) {
    // ---- 4. Hooks ----
    const { colors } = useTheme();
    const { t } = useTranslation();
    const dispatch = useAppDispatch();
    const isAuthenticated = useAppSelector((s) => !!s.auth.accessToken);
    const items = useAppSelector((s) => s.<feature>.items);

    // Pull-to-refresh is tracked separately from the initial query load.
    const [isPullRefreshing, setIsPullRefreshing] = useState(false);

    // The query is skipped until the user is authenticated.
    const { data, isLoading, refetch } = useGet<Feature>ListQuery(undefined, {
        skip: !isAuthenticated,
    });

    // ---- 5. Effects ----
    // Mirror fetched data into the slice so other screens can read it.
    useEffect(() => {
        if (data?.data) {
            dispatch(set<Feature>Items(data.data));
        }
    }, [data, dispatch]);

    // Refetch whenever the screen regains focus (e.g. back from a detail).
    useFocusEffect(
        useCallback(() => {
            if (isAuthenticated) {
                refetch();
            }
        }, [isAuthenticated, refetch]),
    );

    // ---- 6. Event handlers ----
    /** Handles pull-to-refresh: refetches, then clears the refreshing flag. */
    const handleRefresh = useCallback(async (): Promise<void> => {
        setIsPullRefreshing(true);
        try {
            await refetch();
        } finally {
            setIsPullRefreshing(false);
        }
    }, [refetch]);

    /** Navigates to the detail screen for the tapped item. */
    const handleItemPress = useCallback(
        (id: string): void => {
            navigation.navigate('<Name>Detail', { id });
        },
        [navigation],
    );

    // ---- 7. Renderers ----
    /** Renders one list row. Typed with ListRenderItem for inference. */
    const renderItem: ListRenderItem<<Feature>Item> = ({ item }) => (
        <<Feature>Card data={item} onPress={() => handleItemPress(item.id)} />
    );

    /** Empty-state renderer — returns null while the first load is running so
     *  the loader and the empty message never appear together. */
    const renderEmpty = (): React.ReactElement | null => {
        if (isLoading) {
            return null;
        }
        return <EmptyState message={t('<feature>EmptyMessage')} />;
    };

    // ---- 8. Render ----
    return (
        <SafeAreaView style={[styles.container, { backgroundColor: colors.background }]}>
            {/* Initial-load indicator replaces the list until data arrives. */}
            {isLoading && items.length === 0 ? (
                <LoadingIndicator />
            ) : (
                <FlatList
                    data={items}
                    renderItem={renderItem}
                    keyExtractor={(item) => item.id}
                    showsVerticalScrollIndicator={false}
                    removeClippedSubviews
                    maxToRenderPerBatch={10}
                    windowSize={10}
                    initialNumToRender={10}
                    ListEmptyComponent={renderEmpty}
                    refreshControl={
                        <RefreshControl refreshing={isPullRefreshing} onRefresh={handleRefresh} />
                    }
                />
            )}
        </SafeAreaView>
    );
}

// ---- 9. Static styles ----
const styles = StyleSheet.create({
    container: {
        flex: 1,
    },
});
```

For infinite scroll, add `page` state, `appendItems`, `handleLoadMore`,
`onEndReached`, and `ListFooterComponent` per `data-patterns.md` §3.

Checklist:
- `renderItem` typed `ListRenderItem<T>`; `keyExtractor` returns a stable id.
- Query skipped via `{ skip: !isAuthenticated }`; data synced to the slice.
- `useFocusEffect(useCallback(...))` for refetch on focus.
- All four loading states distinguished (data-patterns §2).
- `ListEmptyComponent` returns null during initial load.
- Performance props set on the FlatList.
