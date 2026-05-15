# Template — Constants file

Path: `src/constants/<feature>.ts`
After creating: add it to the barrel `src/constants/index.ts`.

Holds fixed values for a feature: option lists, fixed IDs, enums-as-objects,
icon maps. Constant map objects are `UPPER_SNAKE_CASE` and end with `as const`
so TypeScript infers literal types. Types are DERIVED from the const objects.

```typescript
// <feature>.ts
// Fixed constants for the <feature> domain.

/**
 * Selectable status values for a <feature> item.
 * `as const` makes each value a literal type, not just `string`.
 */
export const <FEATURE>_STATUS = {
    ACTIVE: 'active',
    INACTIVE: 'inactive',
    PENDING: 'pending',
} as const;

/** Union of valid <feature> status values, derived from the object above. */
export type <Feature>Status = typeof <FEATURE>_STATUS[keyof typeof <FEATURE>_STATUS];

/**
 * Option list for a <feature> picker. Labels are i18n KEYS, not display
 * text — the component resolves them with t() at render time.
 */
export const <FEATURE>_OPTIONS = [
    { value: <FEATURE>_STATUS.ACTIVE, labelKey: '<feature>StatusActive' },
    { value: <FEATURE>_STATUS.INACTIVE, labelKey: '<feature>StatusInactive' },
] as const;

/** Fixed backend identifiers for <feature> (mirrors server seed data). */
export const <FEATURE>_IDS = {
    DEFAULT: '00000000-0000-0000-0000-000000000000',
} as const;
```

## Image / icon map

Icon maps belong in `src/constants/icons.ts`. Keys are `UPPER_SNAKE_CASE`;
values are `require()` calls; the object ends with `as const`:

```typescript
/** In-app icons for the <feature> screens. */
export const <FEATURE>_ICONS = {
    EXAMPLE: require('../../assets/icons/<feature>/ic_example.png'),
} as const;
```

Asset files on disk: `assets/icons/<feature>/ic_<name>.png`.

## Update the barrel

`src/constants/index.ts`:

```typescript
export * from './<feature>';
```

This lets feature code import from `'../../constants'` rather than the file.

Rules:
- Constant maps: `UPPER_SNAKE_CASE` + `as const` (conventions §3, R13).
- Derive types from the const object — do NOT declare a separate enum/type
  by hand (conventions §3).
- Labels for user-facing options are i18n KEYS, never display strings (R61).
- Never `require()` an image inline in a component — reference it through an
  icon map here (conventions §3).
- Add the file to `constants/index.ts` (conventions §1).
- No business logic in constants files — values only.
