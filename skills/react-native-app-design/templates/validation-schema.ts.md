# Template — Validation schema

Path: `src/validation/<feature>/<name>Schema.ts`
After creating: add to `src/validation/<feature>/index.ts`.

Every schema is a FACTORY that takes the i18n `t` function so messages are
localized. The file also exports the matching `FormData` interface.

```typescript
// <name>Schema.ts
// Yup validation schema for the <name> form.

// ---- Imports ----
import * as yup from 'yup';

// ---- Form data type ----
/** Field shape of the <name> form. */
export interface <Name>FormData {
    email: string;
    password: string;
}

// ---- Schema factory ----
/**
 * Builds the Yup schema for the <name> form.
 * @param t  i18n translation function — every validation message is localized
 *           through it, so no message strings are hardcoded here.
 * @returns  A Yup object schema for use with yupResolver.
 */
export const create<Name>Schema = (t: (key: string) => string) => {
    return yup.object().shape({
        email: yup
            .string()
            .trim()
            .lowercase()
            .required(t('emailRequired'))
            .email(t('emailInvalid')),

        password: yup
            .string()
            .required(t('passwordRequired'))
            // Minimum length is a security requirement, not just UX.
            .min(8, t('passwordMinLength')),
    });
};
```

If the schema depends on a prop (e.g. `editMode`), accept it as a parameter and
the consuming hook memoizes the result:

```typescript
export const create<Name>Schema = (t: (key: string) => string, editMode: boolean) => {
    return yup.object().shape({
        // In edit mode the password field is optional; otherwise required.
        password: editMode
            ? yup.string().min(8, t('passwordMinLength'))
            : yup.string().required(t('passwordRequired')).min(8, t('passwordMinLength')),
    });
};
```

Every message key used here must exist in `src/i18n/locales/<lang>/common.json`.

## Advanced patterns

### Conditional optionality in edit mode

A field that is required on create but optional on edit uses `.test()` so an
empty value passes in edit mode. Nest the ternary when a sub-flag applies:

```typescript
// Required normally; optional in edit mode unless the password is being changed.
password: editMode
    ? isPasswordBeingChanged
        ? yup.string().required(t('passwordRequired')).min(8, t('passwordMinLength'))
        : yup.string()                                  // fully optional
    : yup.string().required(t('passwordRequired')).min(8, t('passwordMinLength')),

// A field optional only in edit mode, validated only when non-empty:
username: editMode
    ? yup.string().test('min-length', t('usernameMinLength'), (value) => {
        // An empty value is allowed in edit mode; a present value must be valid.
        if (!value) {
            return true;
        }
        return value.length >= 3;
      })
    : yup.string().required(t('usernameRequired')).min(3, t('usernameMinLength')),
```

### Cross-field validation

Use `.test()` with `this.parent` to compare sibling fields (use a regular
`function`, not an arrow, so `this` binds):

```typescript
confirmPassword: yup
    .string()
    .required(t('confirmPasswordRequired'))
    .test('passwords-match', t('confirmPasswordMismatch'), function (value) {
        // this.parent gives access to the other fields in the same object.
        return value === this.parent.newPassword;
    }),
```

### Custom test — password strength

```typescript
newPassword: yup
    .string()
    .required(t('newPasswordRequired'))
    .min(8, t('passwordMinLength'))
    .test('password-strength', t('passwordWeak'), (value) => {
        // Require all four character classes for a strong password.
        if (!value) {
            return false;
        }
        return (
            /[A-Z]/.test(value) &&
            /[a-z]/.test(value) &&
            /[0-9]/.test(value) &&
            /[!@#$%^&*(),.?":{}|<>]/.test(value)
        );
    }),
```

### Transform — empty to default

```typescript
// An empty port input becomes the default 80 rather than NaN.
httpPort: yup
    .number()
    .transform((value, original) => (original === '' || original === null ? 80 : value))
    .default(80)
    .integer()
    .min(1, t('portInvalid'))
    .max(65535, t('portInvalid')),
```

### Multiple related schemas in one file

When two form variants share most fields (e.g. a URL variant and a
registration-code variant), put BOTH factories and BOTH `FormData` interfaces
in one file, sharing the common field definitions.

### Default-values factory

Export a function that returns the form's default values, so the form hook and
any reset logic share one source:

```typescript
/** Default values for the <Name> form. */
export const get<Name>DefaultValues = (): <Name>FormData => ({
    email: '',
    password: '',
});
```

## Regex reference

Common patterns used by validation schemas — keep them consistent:

| Field | Regex |
| --- | --- |
| Email | `/^[a-zA-Z0-9][a-zA-Z0-9._%+-]*[a-zA-Z0-9]@[a-zA-Z0-9][a-zA-Z0-9.-]*[a-zA-Z0-9]\.[a-zA-Z]{2,}$/` |
| 6-digit OTP | `/^\d{6}$/` |
| Person name (Unicode) | `/^[\p{L}\s'-]+$/u` |
| RTSP url | `/^rtsp:\/\/.+/i` |

## Notes

- Do NOT cast `yupResolver` to `any`. Keep it strictly typed — `R17` (no `any`)
  applies. (The reference codebase casts it; this skill is deliberately
  stricter.)

Checklist:
- Exported as `create<Name>Schema` factory taking `t` (R52).
- `<Name>FormData` interface exported (R52).
- Every `.required()/.min()/.matches()/.test()` message via `t()` — no literals (R65).
- Cross-field `.test()` uses `function` (not arrow) for `this.parent`.
- All keys added to the locale JSON (R62).
- Added to the validation barrel.
