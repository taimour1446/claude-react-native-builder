# Lifecycle Hooks — non-form custom hooks

`form-hooks.ts.md` covers the two-hook form pattern. This file covers the other
kind of custom hook: app-lifecycle hooks that wire native subscriptions, OTA
updates, and device data. Read it when building any hook that is not a form.

General rules for all custom hooks:

- A hook returns an **object** with named fields (`{ unreadCount }`), not a
  tuple — unless it returns a single value, then return that value directly.
- Native subscriptions/listeners are held in `useRef` and removed in the
  `useEffect` cleanup function.
- Every `useEffect` has a correct, exhaustive dependency array.
- Callbacks passed to children or used in effect deps are wrapped in
  `useCallback`.
- A `useRef<boolean>` guard prevents a duplicate in-flight async operation.
- Errors inside a hook are caught and either surfaced (toast) or logged — never
  swallowed silently.

---

## 1. Screen-focus refetch

Refetch data when a screen regains focus (e.g. returning from a detail screen):

```typescript
import { useFocusEffect } from '@react-navigation/native';
import { useCallback } from 'react';

// Inside the screen/hook:
useFocusEffect(
    // useCallback is REQUIRED — useFocusEffect re-runs whenever the callback
    // identity changes, so an unmemoized callback would loop.
    useCallback(() => {
        if (isAuthenticated) {
            refetch();
        }
    }, [isAuthenticated, refetch]),
);
```

## 2. Notification listener hook

Sets up foreground + tap listeners; cleans them up on unmount.

```typescript
// useNotifications.ts
// Subscribes to push-notification events and routes taps to the right screen.

import { useEffect, useRef } from 'react';
import * as Notifications from 'expo-notifications';
import { useAppDispatch } from '../store/hooks';

/** Wires notification listeners for the lifetime of the app. Returns the
 *  current unread count for badge display. */
export const useNotifications = () => {
    const dispatch = useAppDispatch();

    // Subscriptions are held in refs so the cleanup function can remove the
    // exact same subscription objects.
    const receivedRef = useRef<Notifications.Subscription>();
    const responseRef = useRef<Notifications.Subscription>();

    useEffect(() => {
        // Fires when a notification arrives while the app is foregrounded.
        receivedRef.current = Notifications.addNotificationReceivedListener((n) => {
            // …store the notification in Redux…
        });

        // Fires when the user TAPS a notification.
        responseRef.current = Notifications.addNotificationResponseReceivedListener((r) => {
            // …navigate based on r.notification.request.content.data…
        });

        // Cold start: the app may have been opened BY a notification tap.
        // Delay slightly so navigation is ready before we route.
        Notifications.getLastNotificationResponseAsync().then((response) => {
            if (response) {
                setTimeout(() => {
                    // …route to the target screen…
                }, 500);
            }
        });

        // Cleanup: remove both subscriptions.
        return () => {
            receivedRef.current?.remove();
            responseRef.current?.remove();
        };
    }, [dispatch]);

    const unreadCount = useAppSelector((s) => s.notifications.unreadCount);
    return { unreadCount };
};
```

## 3. Notification registration hook

Registers / clears the push token in step with auth state. Two separate
effects: one for login, one for logout.

```typescript
// useNotificationRegistration.ts — token lifecycle vs auth state.

useEffect(() => {
    // On login: get the device push token and send it to the backend.
    if (user && accessToken) {
        registerForPushNotificationsAsync().then(({ token, status }) => {
            dispatch(setPermissionStatus(status));
            if (token) {
                dispatch(setExpoPushToken(token));
                dispatch(notificationApi.endpoints.registerToken.initiate({ token }));
            }
        });
    }
}, [user, accessToken, dispatch]);

useEffect(() => {
    // On logout: clear notification state.
    if (!user && !accessToken) {
        dispatch(resetNotifications());
    }
}, [user, accessToken, dispatch]);
```

## 4. OTA update hook (expo-updates)

Auto-checks, downloads, and applies an EAS Update.

```typescript
// useAppUpdates.ts — automatic OTA update handling.

import { useEffect, useRef, useState } from 'react';
import * as Updates from 'expo-updates';
import { useUpdates } from 'expo-updates';
import { Platform } from 'react-native';

type UpdateStatus = 'idle' | 'downloading' | 'restarting';

export const useAppUpdates = () => {
    // expo-updates surfaces availability/progress through this hook.
    const { isUpdateAvailable, isUpdatePending, isChecking, isDownloading } = useUpdates();
    const [updateStatus, setUpdateStatus] = useState<UpdateStatus>('idle');

    // Ref guard: prevents kicking off a second download while one is running.
    const isUpdating = useRef(false);

    useEffect(() => {
        if (isUpdateAvailable && !isDownloading && !isUpdating.current) {
            isUpdating.current = true;
            setUpdateStatus('downloading');
            Updates.fetchUpdateAsync()
                .then(() => {
                    setUpdateStatus('restarting');
                    return Updates.reloadAsync();
                })
                .catch(() => {
                    // Reset on failure so a later attempt can retry.
                    isUpdating.current = false;
                    setUpdateStatus('idle');
                });
        }
    }, [isUpdateAvailable, isDownloading]);

    // A blocking update screen is shown on Android during the process.
    const showUpdateScreen =
        Platform.OS === 'android' &&
        (updateStatus === 'downloading' || updateStatus === 'restarting');

    return { showUpdateScreen, updateStatus, isChecking, isUpdatePending };
};
```

## 5. Device-data hook

A thin hook returning a single derived value uses the value directly (no
wrapping object) and skips its query when inputs are missing:

```typescript
// useUserTimezone.ts — the signed-in user's IANA timezone.

import { skipToken } from '@reduxjs/toolkit/query';

/** Returns the current user's timezone string, or undefined if unknown. */
export const useUserTimezone = (): string | undefined => {
    const userId = useAppSelector((s) => s.auth.user?.id);
    // skipToken: the query does not run at all until userId exists.
    const { data: profile } = useGetProfileQuery(userId ?? skipToken);
    return profile?.timezone;
};
```

## 6. Polling with an interval

When a screen must poll while visible:

```typescript
useEffect(() => {
    // Refetch every 30s while this screen is mounted.
    const id = setInterval(() => {
        refetch();
    }, 30000);
    // Clear the interval on unmount so it cannot fire on a dead screen.
    return () => clearInterval(id);
}, [refetch]);
```

## 7. Previous-value tracking

To react to a value CHANGING (not just its current state), keep the previous
value in a ref:

```typescript
const prevStatusRef = useRef<string | undefined>(undefined);

useEffect(() => {
    const current = item?.status;
    // Only act on an actual transition, and only after the first render.
    if (current && prevStatusRef.current !== undefined && current !== prevStatusRef.current) {
        if (current === 'Offline' && prevStatusRef.current === 'Online') {
            dispatch(showToast({ message: t('itemWentOffline'), type: 'warning' }));
        }
    }
    prevStatusRef.current = current;
}, [item?.status, dispatch, t]);
```
