# Template — Redux slice

Path: `src/store/slices/<feature>Slice.ts`
After creating: register the reducer in `src/store/index.ts`.

One slice per feature domain. Typed state, typed `PayloadAction`, named
selectors. Reducers mutate draft state (Immer). Each reducer is commented.

```typescript
// <feature>Slice.ts
// Redux slice owning the <feature> domain's client-side state.

// ---- Imports ----
import { createSlice, PayloadAction } from '@reduxjs/toolkit';
import type { RootState } from '../index';
// If syncing from a server call, also import the matching API:
// import { <feature>Api } from '../../services/<feature>Api';

// ---- State type ----
/** Shape of the <feature> slice state. */
type <Feature>State = {
    items: string[];
    selectedId: string | null;
    isLoading: boolean;
};

// ---- Initial state ----
const initialState: <Feature>State = {
    items: [],
    selectedId: null,
    isLoading: false,
};

// ---- Slice ----
/** Slice for the <feature> domain. */
const <feature>Slice = createSlice({
    name: '<feature>',
    initialState,
    reducers: {
        /** Replaces the full item list (e.g. after a manual refresh). */
        set<Feature>Items: (state, action: PayloadAction<string[]>) => {
            state.items = action.payload;
        },
        /** Marks one item as selected; pass null to clear the selection. */
        select<Feature>: (state, action: PayloadAction<string | null>) => {
            state.selectedId = action.payload;
        },
    },
    extraReducers: (builder) => {
        // NOTE: all addCase(...) calls MUST come before any addMatcher(...) calls.
        builder
            // Example: sync this slice when a server list query succeeds.
            .addMatcher(
                <feature>Api.endpoints.get<Feature>List.matchFulfilled,
                (state, { payload }) => {
                    // Mirror the server's item ids into local state so other
                    // parts of the app can read them without re-querying.
                    state.items = payload.data.map((item) => item.id);
                },
            );
    },
});

// ---- Actions ----
export const { set<Feature>Items, select<Feature> } = <feature>Slice.actions;

// ---- Selectors ----
/** Selects all <feature> item ids. */
export const select<Feature>Items = (state: RootState) => state.<feature>.items;
/** Selects the currently selected <feature> id, or null. */
export const selectSelected<Feature>Id = (state: RootState) => state.<feature>.selectedId;

// ---- Reducer ----
export default <feature>Slice.reducer;
```

Register in `src/store/index.ts`:

```typescript
import <feature> from './slices/<feature>Slice';
// in the reducer object:
<feature>,
```

## Variants — see `data-patterns.md`

- **Paginated slice** (`data-patterns.md` §4) — holds `items`, `meta`,
  `hasMore`; `setItems` REPLACES (page 1 / refresh), `appendItems` APPENDS the
  next page de-duplicating by id.
- **Per-tab pagination** (§4) — `tabPagination: Record<TabId, number>` with
  `incrementPage` / `resetTabPage`; `setFilters` resets every tab to page 1.
- **`createAsyncThunk` with `rejectWithValue`** (§5) — for non-RTK async work
  that can fail.
- Selectors are PLAIN functions, not `createSelector`. Entities are stored as
  arrays, not normalized keyed objects.
- Prefer wiring `resetX` to the `logout` action via an `extraReducers`
  `addMatcher` so logout reliably clears this slice.

Checklist:
- State typed `<Feature>State`; `initialState` typed (R20).
- Every reducer has `PayloadAction<…>` and a doc comment (R20, R28).
- `addCase` before `addMatcher` in `extraReducers` (conventions §10).
- Named PLAIN selectors typed with `RootState` (conventions §10).
- Default-export the reducer; named-export actions + selectors.
- Registered in the store.
