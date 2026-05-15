# Data Patterns — lists, pagination, RTK Query, slices

Advanced data-layer patterns extracted from the reference architecture. Read
this in addition to `patterns.md` whenever building lists, paginated data, or
RTK Query endpoints beyond a trivial query.

---

## 1. FlatList configuration

A list rendered with `FlatList` is configured as follows. The renderItem
callback is typed with `ListRenderItem<T>`.

```typescript
import { FlatList, type ListRenderItem } from 'react-native';

/** Renders one row. Typed with ListRenderItem so item/index are inferred. */
const renderItem: ListRenderItem<Item> = ({ item, index }) => (
    <ItemRow data={item} isLast={index === items.length - 1} />
);

<FlatList
    data={items}
    renderItem={renderItem}
    keyExtractor={(item) => item.id}        // stable id key — never the index
    contentContainerStyle={{ paddingHorizontal: spacing.md, paddingVertical: spacing.md }}
    showsVerticalScrollIndicator={false}
    // --- performance props for long lists ---
    removeClippedSubviews={true}
    maxToRenderPerBatch={10}
    windowSize={10}
    initialNumToRender={10}
    // --- non-happy states ---
    ListEmptyComponent={renderEmpty}        // returns null while loading
    ListFooterComponent={isLoadingMore ? <LoadingIndicator /> : null}
    // --- pull to refresh ---
    refreshControl={
        <RefreshControl refreshing={isPullRefreshing} onRefresh={handleRefresh} />
    }
    // --- infinite scroll ---
    onEndReached={handleLoadMore}
    onEndReachedThreshold={0.5}
    scrollEnabled={items.length > 0}
/>
```

- `keyExtractor` always returns a stable id, never the array index.
- Performance props are added for any list that can grow long.
- A `ListEmptyComponent` renderer returns `null` while loading so the empty
  message and the loader never show at once.
- When a filter changes, scroll the list to the top:
  `flatListRef.current?.scrollToOffset({ offset: 0, animated: false })`.

## 2. Multi-state loading

A data screen distinguishes FOUR loading states — do not collapse them:

| State | Meaning | Where the indicator shows |
| --- | --- | --- |
| `isLoading` | First load, no data yet | Centered loader replacing the list |
| `isRefetching` | Re-query after a filter change | Loader above the list |
| `isPullRefreshing` | User pulled to refresh | `RefreshControl` spinner |
| `isLoadingMore` | Next page loading (infinite scroll) | `ListFooterComponent` |

Render rule: `ListEmptyComponent` returns `null` when `isLoading` is true.

## 3. Screen-level pagination (infinite scroll)

```typescript
// Page state lives in the screen (or a slice for cross-screen lists).
const [page, setPage] = useState(1);
const [isLoadingMore, setIsLoadingMore] = useState(false);

const { data, isLoading } = useGetItemsQuery(
    { page, limit: PAGE_LIMIT },
    { skip: !isAuthenticated },
);

// Sync into the slice: page 1 REPLACES, later pages APPEND.
useEffect(() => {
    if (data?.data && data?.meta) {
        if (page === 1) {
            dispatch(setItems({ items: data.data, meta: data.meta }));
        } else {
            dispatch(appendItems({ items: data.data, meta: data.meta }));
        }
        setIsLoadingMore(false);
    }
}, [data, page, dispatch]);

/** Advances to the next page if more pages exist and none is in flight. */
const handleLoadMore = useCallback(() => {
    if (!isLoadingMore && hasMore && !isLoading) {
        setIsLoadingMore(true);
        setPage((prev) => prev + 1);
    }
}, [isLoadingMore, hasMore, isLoading]);
```

## 4. Paginated slice

A slice backing a paginated list:

```typescript
type ItemsState = {
    items: Item[];
    meta: PaginationMeta | null;   // { total, page, pageCount, limit }
    hasMore: boolean;
};

reducers: {
    /** Replaces the list — used for page 1 / refresh. */
    setItems: (state, action: PayloadAction<{ items: Item[]; meta: PaginationMeta }>) => {
        state.items = action.payload.items;
        state.meta = action.payload.meta;
        state.hasMore = action.payload.meta.page < action.payload.meta.pageCount;
    },
    /** Appends the next page, de-duplicating by id so a re-fetched page
     *  cannot insert the same item twice. */
    appendItems: (state, action: PayloadAction<{ items: Item[]; meta: PaginationMeta }>) => {
        const existingIds = new Set(state.items.map((i) => i.id));
        const fresh = action.payload.items.filter((i) => !existingIds.has(i.id));
        state.items = [...state.items, ...fresh];
        state.meta = action.payload.meta;
        state.hasMore = action.payload.meta.page < action.payload.meta.pageCount;
    },
}
```

### Per-tab pagination

When one screen has several tabs each with its own page counter:

```typescript
type State = {
    tabPagination: Record<TabId, number>;   // page number per tab
};
reducers: {
    /** Bumps the page counter for one tab. */
    incrementPage: (state, action: PayloadAction<TabId>) => {
        state.tabPagination[action.payload] += 1;
    },
    /** Resets one tab's page back to 1. */
    resetTabPage: (state, action: PayloadAction<TabId>) => {
        state.tabPagination[action.payload] = 1;
    },
    /** Changing filters resets every tab's page to 1. */
    setFilters: (state, action: PayloadAction<Partial<Filters>>) => {
        state.filters = { ...state.filters, ...action.payload };
        (Object.keys(state.tabPagination) as TabId[]).forEach((id) => {
            state.tabPagination[id] = 1;
        });
    },
}
```

## 5. Slice conventions (additions to patterns.md §3)

- **Selectors are plain functions**, not `createSelector`. A by-id selector
  takes the id as a second arg:
  ```typescript
  export const selectItemById = (state: RootState, id: string) =>
      state.items.items.find((i) => i.id === id);
  ```
- **Entities are stored as arrays**, not normalized keyed objects
  (`{ [id]: entity }`).
- `PayloadAction` payloads vary: a value (`PayloadAction<string>`), an object,
  `PayloadAction<Partial<Filters>>`, or a keyed update
  `PayloadAction<{ key: keyof Filters; value: FilterValue }>`.
- **A reducer may reset related fields.** When one piece of state changing
  invalidates another, reset both in the same reducer — e.g. selecting a new
  list item resets the scroll/playback position:
  ```typescript
  selectClip: (state, action: PayloadAction<number>) => {
      state.selectedClipIndex = action.payload;
      // Selecting a different clip invalidates the old position/play state.
      state.currentTime = 0;
      state.isPlaying = false;
  },
  ```
- **Bounded lists.** A list that grows unboundedly (e.g. received
  notifications) is capped in its reducer so memory stays bounded:
  ```typescript
  addNotification: (state, action: PayloadAction<StoredNotification>) => {
      // Keep only the most recent 50 — prevents unbounded growth.
      state.notifications = [action.payload, ...state.notifications].slice(0, 50);
  },
  ```
- `createAsyncThunk` for non-RTK async work uses `rejectWithValue`:
  ```typescript
  export const doThing = createAsyncThunk(
      'feature/doThing',
      async (arg: Arg, { rejectWithValue }) => {
          try {
              return await someAsyncCall(arg);
          } catch (error) {
              // rejectWithValue routes the message to the .rejected case.
              return rejectWithValue((error as Error).message);
          }
      },
  );
  ```
- A slice exposes a `resetX` action to clear its state on logout. Note: in the
  reference these resets are dispatched **manually**, not automatically — when
  building new apps, prefer wiring resets to the `logout` action via
  `extraReducers` `addMatcher` so logout reliably clears every slice.

## 6. RTK Query — cache tags

Every query/mutation participates in cache invalidation via `tagTypes`.

```typescript
tagTypes: ['Item'],
endpoints: (builder) => ({
    // List query provides a LIST tag plus one tag per item.
    getItems: builder.query<ItemsResponse, void>({
        query: () => '/items',
        providesTags: (result) =>
            result
                ? [
                    ...result.data.map((i) => ({ type: 'Item' as const, id: i.id })),
                    { type: 'Item' as const, id: 'LIST' },
                  ]
                : [{ type: 'Item' as const, id: 'LIST' }],
    }),
    // Single query provides that item's tag.
    getItemById: builder.query<ItemResponse, string>({
        query: (id) => `/items/${id}`,
        providesTags: (_r, _e, id) => [{ type: 'Item', id }],
    }),
    // Create invalidates the LIST so it refetches.
    createItem: builder.mutation<ItemResponse, CreateItemRequest>({
        query: (body) => ({ url: '/items', method: 'POST', body }),
        invalidatesTags: [{ type: 'Item', id: 'LIST' }],
    }),
    // Update invalidates BOTH the item and the list.
    updateItem: builder.mutation<ItemResponse, UpdateItemRequest>({
        query: ({ id, ...body }) => ({ url: `/items/${id}`, method: 'PUT', body }),
        invalidatesTags: (_r, _e, { id }) => [{ type: 'Item', id }, { type: 'Item', id: 'LIST' }],
    }),
}),
```

## 7. RTK Query — transformResponse

To unwrap the `{ ok, data }` envelope so the hook returns `data` directly:

```typescript
getProfile: builder.query<UserProfile, string>({
    query: (id) => `/users/${id}`,
    // Unwrap the envelope — consumers get UserProfile, not { ok, data }.
    transformResponse: (response: ApiEnvelope<UserProfile>) => response.data,
}),
```

## 8. RTK Query — infinite-scroll endpoint (merge)

For an endpoint whose pages accumulate in the RTK Query cache itself:

```typescript
getFeed: builder.query<FeedResponse, FeedArgs>({
    query: (args) => `/feed${buildQueryString(args)}`,
    // Cache key ignores page/limit so all pages share one cache entry.
    serializeQueryArgs: ({ queryArgs }) => {
        const { page, limit, ...rest } = queryArgs;
        return rest;
    },
    // Append later pages; page 1 replaces.
    merge: (currentCache, newItems, { arg }) => {
        if (arg.page === 1) {
            return newItems;
        }
        currentCache.data.push(...newItems.data);
    },
    // Refetch when the args (other than page) actually change.
    forceRefetch: ({ currentArg, previousArg }) =>
        JSON.stringify(currentArg) !== JSON.stringify(previousArg),
}),
```

## 9. RTK Query — onQueryStarted

For conditional cache invalidation after a mutation succeeds:

```typescript
updateProfile: builder.mutation<void, UpdateProfileRequest>({
    query: (body) => ({ url: '/profile', method: 'PATCH', body }),
    async onQueryStarted(_arg, { dispatch, queryFulfilled }) {
        try {
            await queryFulfilled;
            // Invalidate only after a confirmed success.
            dispatch(profileApi.util.invalidateTags(['Profile']));
        } catch {
            // On failure, leave the cache untouched.
        }
    },
}),
```

## 10. Query argument typing

- Single id → a primitive: `builder.query<Res, string>`.
- No args → `void`: `builder.query<Res, void>`.
- Multiple args / filters → an object type, declared and exported.
- Optional filters → a `T | void` union: `builder.query<Res, Filters | void>`.

## 11. Idempotency keys

CREATE and DELETE mutations carry an idempotency key so a retried request
cannot double-apply. The key is generated with `generateUUID()` (see
`utils-and-types.md`) and sent in the body under `'idempotency-key'`:

```typescript
deleteItem: builder.mutation<void, { id: string; idempotencyKey: string }>({
    query: ({ id, idempotencyKey }) => ({
        url: `/items/${id}`,
        method: 'DELETE',
        body: { 'idempotency-key': idempotencyKey },
    }),
}),
```

## 12. Conditional query skipping

Skip a query when its inputs are not ready — two equivalent forms:

```typescript
// Form A — the skip option:
useGetItemsQuery(undefined, { skip: !isAuthenticated });

// Form B — skipToken (preferred when the ARG itself may be missing):
import { skipToken } from '@reduxjs/toolkit/query';
useGetProfileQuery(userId ?? skipToken);
```

Use `skipToken` when the argument itself might be absent; use the `skip` option
when a separate condition gates the query.
