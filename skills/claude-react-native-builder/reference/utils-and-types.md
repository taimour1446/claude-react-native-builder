# Utils and Types

Covers the `src/utils/` and `src/types/` layers, which `architecture.md`
describes structurally but does not detail.

---

## Part A — `src/types/`

Shared TypeScript types, grouped by feature domain (`camera.ts`, `incident.ts`,
`profile.ts`, `speaker.ts`, …).

### Conventions

- **One file per domain.** Types used across files live here; a type used only
  inside one API file may stay in that API file.
- **Nested interface chains** model hierarchical API data — e.g. an `Incident`
  has an `IncidentCamera`, an `IncidentObject | null`, an `IncidentArchive | null`.
  Each level is its own named `interface`.
- **API response envelopes** are typed:
  ```typescript
  /** Standard list response with pagination metadata. */
  export interface ItemsResponse {
      ok: boolean;
      data: Item[];
      meta: PaginationMeta;
  }
  /** Standard pagination metadata. */
  export interface PaginationMeta {
      total: number;
      page: number;
      pageCount: number;
      limit: number;
  }
  ```
- **Discriminated unions** for request variants that differ by a type field:
  ```typescript
  interface CreateSpeakerBase {
      idempotencyKey: string;
      name: string;
      cameraIds: string[];
  }
  /** IP speakers require a url. */
  export interface CreateIPSpeakerRequest extends CreateSpeakerBase {
      type: 'IP';
      url: string;
  }
  /** SIP speakers have no url. */
  export interface CreateSIPSpeakerRequest extends CreateSpeakerBase {
      type: 'SIP';
  }
  /** The discriminant `type` lets TS narrow which shape applies. */
  export type CreateSpeakerRequest = CreateIPSpeakerRequest | CreateSIPSpeakerRequest;
  ```
- **Union types** for small fixed string sets: `type CameraTab = 'pnp' | 'onvif' | 'generic'`.
  For value sets also needed at runtime, derive the union from an `as const`
  object instead (see `conventions.md` §3).
- **Optionality:** use `?` for a field that may be absent; use `| null` for a
  field the API explicitly returns as null. Do not mix the two meanings.
- **Naming:** `interface`/`type` are `PascalCase`. Request types end in
  `Request`, responses in `Response`, form shapes in `FormData`,
  filters in `Filters`.
- A "validated form type" can be derived from a schema rather than written by
  hand:
  ```typescript
  export type ValidatedLoginData = yup.InferType<ReturnType<typeof createLoginSchema>>;
  ```

---

## Part B — `src/utils/`

Pure, side-effect-free helpers. No React, no Redux, no navigation. Every
exported helper has a doc comment with `@param`/`@returns` when not obvious.

The standard utility modules and what they contain:

### `fontScaling.ts` — text scaling
- `scaleFont(size, options?)` — builds the `typography.fontSize` scale; combines
  the OS font-scale (clamped between `minScale` and `maxScale`) with a moderate
  screen-width factor. Base reference device 390×844.
- `getLineHeight(fontSize, multiplier = 1.4)` — the only sanctioned way to set
  `lineHeight`.
- `scaleWithoutAccessibility(size)` — width scaling only; for icon/decorative
  text that must not grow with the OS accessibility setting.
- `getFontScale()`, `getPixelRatio()`, `isLargeTextEnabled()`,
  `scaleSpacing()`, `responsiveFontSize()`.

### `dateFormatters.ts` — timezone-aware date/time
- **Rule: all dates crossing Redux or the API are ISO 8601 strings**, never
  `Date` objects (Redux state must stay serializable).
- Formatting is **timezone-aware via `Intl.DateTimeFormat`**, falling back to
  the device timezone when none is supplied.
- Typical functions: `getDefaultDateRange()`, `getStartOfDayInTimezone(date, tz?)`,
  `getEndOfDayInTimezone(date, tz?)`, `formatTime(date, tz?)` → `HH:MM:SS`,
  `formatArchiveDate(date, tz?)` → `"Tue 30 Sep - 12:45 AM"`.

### `errorHelpers.ts` — API error parsing
Already covered in `patterns.md` §16. Functions: `parseApiErrors`,
`setFormErrors`, `getGenericErrorMessage`, `isApiFieldError`.

### `fieldMappings.ts` — backend↔frontend field names
Covered in `patterns.md` §17: `BACKEND_TO_FRONTEND_FIELD_MAP` +
`mapBackendFieldName()`.

### `uuid.ts` — id generation
```typescript
import * as Crypto from 'expo-crypto';
/** Generates an RFC-4122 v4 UUID — used for mutation idempotency keys. */
export const generateUUID = (): string => Crypto.randomUUID();
```

### `imageUtils.ts` — image helpers
```typescript
/**
 * Appends a time-bucketed query param so a live thumbnail refreshes at most
 * once per `refreshInterval` ms while still being disk-cacheable.
 * @returns the cache-busted url, or undefined if no url was given.
 */
export const getCacheBustedUrl = (
    url: string | undefined,
    refreshInterval = 60000,
): string | undefined => {
    if (!url) {
        return undefined;
    }
    const bucket = Math.floor(Date.now() / refreshInterval);
    return `${url}${url.includes('?') ? '&' : '?'}_t=${bucket}`;
};
```

### `fileSize.ts` — file size helpers
`getFileSizeFromUrl(url)` (HEAD → content-length, fallback to blob size),
`formatFileSize(bytes, decimals = 2)` → `"1.5 MB"`.

### Data mappers — `<feature>DataMapper.ts`
For features with an edit mode, a mapper module converts an API entity into the
shape a form expects:
```typescript
/**
 * Maps an API Camera entity to the P&P form's default values.
 * Note: password fields are intentionally left blank — credentials are never
 * pre-filled into an edit form.
 */
export const mapCameraToPnPForm = (camera: Camera): PnPFormData => ({
    registrationCode: camera.registrationCode ?? '',
    cameraName: camera.name ?? '',
    username: camera.username ?? '',
    password: '',           // never pre-fill credentials
    confirmPassword: '',
    description: camera.description ?? '',
});
```

### `removeEmptyFields` — pre-submit cleanup
A generic helper that strips `null`, `undefined`, `''`, and `NaN` from an
object before it is sent as a mutation body:
```typescript
/** Returns a copy of `obj` with null / undefined / empty-string / NaN values
 *  removed — so optional fields are omitted from API requests entirely. */
export const removeEmptyFields = <T extends Record<string, unknown>>(obj: T): Partial<T> => {
    const result: Partial<T> = {};
    (Object.keys(obj) as (keyof T)[]).forEach((key) => {
        const value = obj[key];
        const isEmpty =
            value === null ||
            value === undefined ||
            value === '' ||
            (typeof value === 'number' && Number.isNaN(value));
        if (!isEmpty) {
            result[key] = value;
        }
    });
    return result;
};
```

### Rules for utils
- Pure functions only — no React/Redux/navigation imports.
- Each exported helper has a doc comment; document `@param`/`@returns` when not
  obvious from names and types.
- No `any`; use generics (`<T>`) for shape-preserving helpers.
- Group helpers by purpose into the modules above; do not scatter one-off
  helpers across feature files.
