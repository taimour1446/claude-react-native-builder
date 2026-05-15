# Template — Two-hook form pair

Paths:
- `src/hooks/<feature>/use<Feature>Forms.ts`
- `src/hooks/<feature>/use<Feature>Handlers.ts`
- `src/hooks/<feature>/index.ts` (barrel)

Non-trivial forms split into a **state hook** (`Forms`) and a **logic hook**
(`Handlers`) so the screen stays thin (R54).

---

## `use<Feature>Forms.ts` — form state

```typescript
// use<Feature>Forms.ts
// Owns the react-hook-form instance for the <feature> form.

// ---- Imports ----
import { useMemo } from 'react';
import { useForm } from 'react-hook-form';
import { yupResolver } from '@hookform/resolvers/yup';
import { useTranslation } from 'react-i18next';
import { create<Feature>Schema, type <Feature>FormData } from '../../validation/<feature>';

/**
 * Sets up the <feature> form: resolver, validation mode, default values.
 * @param editMode  When true, validation rules relax for editing an existing item.
 * @returns The react-hook-form object for the screen and the handlers hook.
 */
export const use<Feature>Forms = (editMode: boolean = false) => {
    const { t } = useTranslation();

    // Memoize the schema: it depends on `t` and `editMode`, so without useMemo
    // it would be rebuilt on every render and reset validation.
    const schema = useMemo(() => create<Feature>Schema(t, editMode), [t, editMode]);

    const form = useForm<<Feature>FormData>({
        resolver: yupResolver(schema),
        mode: 'onChange',                       // validate live for instant feedback
        defaultValues: {
            name: '',
        },
    });

    return { form };
};
```

---

## `use<Feature>Handlers.ts` — business logic

```typescript
// use<Feature>Handlers.ts
// Owns the business logic for the <feature> form: API submission, data
// transformation, backend error mapping, toasts, and navigation.

// ---- Imports ----
import type { UseFormReturn } from 'react-hook-form';
import { useTranslation } from 'react-i18next';
import { useNavigation } from '@react-navigation/native';
import type { NativeStackNavigationProp } from '@react-navigation/native-stack';
import { useAppDispatch } from '../../store/hooks';
import { showToast } from '../../store/slices/toastSlice';
import { useCreate<Feature>Mutation } from '../../services/<feature>Api';
import { parseApiErrors, setFormErrors, getGenericErrorMessage } from '../../utils/errorHelpers';
import type { <Feature>FormData } from '../../validation/<feature>';
import type { RootStackParamList } from '../../navigation/RootNavigator';

type NavigationProp = NativeStackNavigationProp<RootStackParamList>;

/**
 * Provides the submit handler and loading state for the <feature> form.
 * @param form  The react-hook-form object from use<Feature>Forms.
 */
export const use<Feature>Handlers = (form: UseFormReturn<<Feature>FormData>) => {
    const { t } = useTranslation();
    const dispatch = useAppDispatch();
    const navigation = useNavigation<NavigationProp>();
    const [create<Feature>, { isLoading }] = useCreate<Feature>Mutation();

    /**
     * Validates and submits the form. On success: toast + navigate back.
     * On a 400 field error: maps messages onto the form fields. Other errors
     * (401/403/5xx) are handled globally by baseQuery.
     */
    const handleSubmit = form.handleSubmit(async (values) => {
        try {
            // .unwrap() makes a failed request throw so it lands in catch.
            await create<Feature>(values).unwrap();
            dispatch(showToast({ message: t('<feature>Created'), type: 'success' }));
            navigation.goBack();
        } catch (error) {
            // parseApiErrors returns per-field messages for a 400 validation
            // error (backend field names already mapped to frontend names).
            const fieldErrors = parseApiErrors(error);
            if (fieldErrors) {
                setFormErrors(form.setError, fieldErrors);
            } else {
                // Not a field error — surface one localized message.
                dispatch(showToast({
                    message: getGenericErrorMessage(error, t('genericError')),
                    type: 'error',
                }));
            }
        }
    });

    return { handleSubmit, isLoading };
};
```

---

## Barrel `index.ts`

```typescript
export { use<Feature>Forms } from './use<Feature>Forms';
export { use<Feature>Handlers } from './use<Feature>Handlers';
```

The screen then does only:

```typescript
const { form } = use<Feature>Forms(editMode);
const { handleSubmit, isLoading } = use<Feature>Handlers(form);
```

## Multi-form variant

When one screen has several form variants (e.g. a type picker that swaps the
form), the Forms hook returns an OBJECT OF FORMS and the Handlers hook returns
an object of multiple named handlers plus shared UI state:

```typescript
// Forms hook returns every form, each memoized against the same params:
return { pnpForm, onvifUrlForm, onvifRegCodeForm, genericUrlForm };

// Handlers hook returns one handler per variant + shared state:
return {
    isSubmitting,
    errorMessage,
    successMessage,
    handleSubmitPnp,
    handleSubmitOnvifUrl,
    handleSubmitOnvifRegCode,
    handleSubmitGenericUrl,
    closeError,
    closeSuccess,
};
```

Each variant's schema is memoized separately with `useMemo` against
`[t, editMode, ...flags]`.

Checklist:
- Two hooks: `Forms` (state) + `Handlers` (logic) (R54).
- Schema memoized when prop-dependent (conventions §13).
- `useForm` with `yupResolver`, `mode: 'onChange'` (R50, R53).
- 400 errors via `parseApiErrors` + `setFormErrors`; nothing else handled
  locally (R44, R55).
- Typed Redux hooks; localized toast messages.
- For multi-variant screens: Forms returns an object of forms, Handlers returns
  named handlers + shared UI state.
- Both hooks added to the feature barrel.
