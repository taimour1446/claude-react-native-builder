# Template — Types file

Path: `src/types/<feature>.ts`

Shared TypeScript types for one feature domain. Read `utils-and-types.md`
Part A for the full conventions.

```typescript
// <feature>.ts
// Shared TypeScript types for the <feature> domain.

// ---- Domain entity ----
/** A <feature> item as returned by the API. */
export interface <Feature> {
    id: string;
    createdAt: string;          // ISO 8601 string
    updatedAt: string;          // ISO 8601 string
    name: string;
    description?: string;       // `?` — may be absent from the payload
    parentId: string | null;    // `| null` — API explicitly returns null
    nested: <Feature>Detail;    // nested interface chain for hierarchical data
}

/** Nested detail object embedded in a <Feature>. */
export interface <Feature>Detail {
    code: string;
    enabled: boolean;
}

// ---- Pagination ----
/** Standard pagination metadata returned with list endpoints. */
export interface PaginationMeta {
    total: number;
    page: number;
    pageCount: number;
    limit: number;
}

// ---- API response envelopes ----
/** List response: ok flag + data array + pagination meta. */
export interface <Feature>ListResponse {
    ok: boolean;
    data: <Feature>[];
    meta: PaginationMeta;
}

/** Single-item response. */
export interface <Feature>Response {
    ok: boolean;
    data: <Feature>;
}

// ---- Request types ----
/** Body for creating a <feature>. */
export interface Create<Feature>Request {
    idempotencyKey: string;     // for mutation idempotency (data-patterns §11)
    name: string;
    description?: string;
}

// ---- Discriminated union (when a request has variants) ----
interface <Feature>RequestBase {
    idempotencyKey: string;
    name: string;
}
/** Variant A — discriminated by `kind`. */
export interface <Feature>RequestA extends <Feature>RequestBase {
    kind: 'A';
    url: string;
}
/** Variant B. */
export interface <Feature>RequestB extends <Feature>RequestBase {
    kind: 'B';
}
/** The `kind` field lets TypeScript narrow which variant applies. */
export type Create<Feature>VariantRequest = <Feature>RequestA | <Feature>RequestB;

// ---- Filters ----
/** Query filters for the <feature> list. */
export interface <Feature>Filters {
    page?: number;
    limit?: number;
    search?: string;
}

// ---- Small fixed string sets ----
/** Use a union for a small fixed set used only at the type level. For a set
 *  also needed at runtime, derive it from an `as const` object instead
 *  (see conventions.md §3). */
export type <Feature>Tab = 'overview' | 'detail' | 'history';
```

Checklist:
- One file per domain; types reused across files live here.
- `interface` for object shapes, `type` for unions/aliases.
- Dates that cross Redux/API are ISO 8601 `string`, not `Date`.
- `?` for absent, `| null` for explicitly-null — never conflated.
- Request types end `Request`, responses `Response`, form shapes `FormData`,
  filters `Filters`.
- Discriminated unions for request variants that differ by a type field.
