# Template — RTK Query API service

Path: `src/services/<feature>Api.ts`
After creating: register in `src/store/index.ts` (reducer + middleware).

One `createApi` per feature domain. All request/response types declared and
exported here. Always uses `createBaseQueryWithAuth()`.

```typescript
// <feature>Api.ts
// RTK Query service for the <feature> domain. All endpoints share the global
// base query, so the auth token and global error handling are automatic.

// ---- Imports ----
import { createApi } from '@reduxjs/toolkit/query/react';
import { createBaseQueryWithAuth } from './baseQuery';

// ---- Request / response types ----
/** Standard backend response envelope: ok flag + typed data payload. */
type ApiEnvelope<T> = { ok: boolean; data: T };

/** A single <feature> item as returned by the backend. */
export type <Feature>Item = {
    id: string;
    name: string;
};

/** Body for creating a <feature> item. */
export type Create<Feature>Request = {
    name: string;
};

/** Response for the list endpoint. */
export type <Feature>ListResponse = ApiEnvelope<<Feature>Item[]>;

/** Response for the create endpoint. */
export type Create<Feature>Response = ApiEnvelope<<Feature>Item>;

// ---- API service ----
/**
 * <feature>Api — CRUD endpoints for the <feature> domain.
 * Register this service's reducer and middleware in src/store/index.ts.
 */
export const <feature>Api = createApi({
    reducerPath: '<feature>Api',
    baseQuery: createBaseQueryWithAuth(),
    // tagTypes enable cache invalidation between queries and mutations.
    tagTypes: ['<Feature>'],
    endpoints: (builder) => ({
        // GET /<feature> — fetches all <feature> items for the user.
        get<Feature>List: builder.query<<Feature>ListResponse, void>({
            query: () => '/<feature>',
            providesTags: ['<Feature>'],
        }),
        // POST /<feature> — creates a new <feature> item.
        create<Feature>: builder.mutation<Create<Feature>Response, Create<Feature>Request>({
            query: (body) => ({ url: '/<feature>', method: 'POST', body }),
            // Invalidate the list so it refetches after a successful create.
            invalidatesTags: ['<Feature>'],
        }),
    }),
});

// ---- Generated hooks ----
// RTK Query auto-generates one hook per endpoint; export them for screens/hooks.
export const {
    useGet<Feature>ListQuery,
    useCreate<Feature>Mutation,
} = <feature>Api;
```

Register in `src/store/index.ts`:

```typescript
import { <feature>Api } from '../services/<feature>Api';
// reducer:
[<feature>Api.reducerPath]: <feature>Api.reducer,
// middleware:
.concat(<feature>Api.middleware)
```

## Advanced — see `data-patterns.md`

This template shows a basic service. For the full data-layer patterns read
`reference/data-patterns.md`:

- **Cache tags** (§6) — `providesTags` with BOTH a `{ id: 'LIST' }` tag and a
  per-item tag; `invalidatesTags` on mutations (update invalidates the item AND
  the list).
- **transformResponse** (§7) — unwrap the `{ ok, data }` envelope so consumers
  receive `data` directly.
- **Infinite-scroll endpoints** (§8) — `serializeQueryArgs` + `merge` +
  `forceRefetch` so pages accumulate in one cache entry.
- **onQueryStarted** (§9) — conditional cache invalidation after success.
- **Idempotency keys** (§11) — CREATE/DELETE mutations send an
  `'idempotency-key'` in the body.
- **Query arg typing** (§10) — primitive for an id, `void` for none, an object
  for filters, `T | void` for optional filters.

The response envelope is usually `{ ok, data }`; a paginated one adds `meta`.

Checklist:
- `reducerPath` matches the file name (R14, conventions §11).
- `createBaseQueryWithAuth()` used — no manual headers, no manual 401 (R42, R43).
- Every request/response type declared and exported, no `any` (R19).
- `tagTypes` declared; queries `providesTags`, mutations `invalidatesTags`.
- Each endpoint has a `// METHOD /path` comment.
- CREATE/DELETE mutations carry an idempotency key.
- Hooks exported at the bottom.
- Registered in the store (reducer + middleware).
