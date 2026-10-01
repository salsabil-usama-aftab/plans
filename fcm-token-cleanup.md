# FCM Token Cleanup — Fix Stuck & Partial-Failure Notifications

## Problem Summary

Push notifications were broken in two ways:

1. **Stuck in `pending` (post May 12):** The `push-notification-dispatcher` edge function was deleted from the codebase during the March 28 consolidation but was never re-added, so the Supabase Database Webhook had nothing to call. Fixed: function restored and redeployed.

2. **`partial_failure` on every dispatch:** `user_devices` accumulates stale and cross-project FCM tokens that never get cleaned up. Every notification hits all of them, and some always fail:
   - `messaging/registration-token-not-registered` — token from a previous install, now invalidated by Firebase
   - `messaging/mismatched-credential` — token minted by a different Firebase project (staging vs production)

## Root Cause

- The backend dispatcher marks the queue entry as `partial_failure` but **never removes the bad tokens** from `user_devices`.
- The mobile app registers the current token on every login but **never unregisters the old token** when the FCM token rotates.
- `user_devices` has a UNIQUE constraint on `(user_id, fcm_token)` so each new token adds a new row; old tokens are never evicted.

---

## Changes Required

### Change 1 — Backend: Delete dead tokens after a failed FCM dispatch

**File:** `supabase/functions/services/push-notification-dispatcher-service.ts`

After `sendMulticastMessage` returns, zip `tokens[]` with `fcmResult.responses[]`. For every response whose error code is one of:

- `messaging/registration-token-not-registered`
- `messaging/mismatched-credential`
- `messaging/invalid-registration-token`

…call `userDevicesRepository.deleteDeviceByToken(request.userId, token)` to remove it immediately.

These are permanent failures — the token will never become valid again. Cleaning up inline prevents them from being retried on every future notification for that user.

`IUserDevicesRepository.deleteDeviceByToken(userId, fcmToken)` already exists. No new repository method needed.

**Also needs:** A new migration to add `partial_failure` to the `notifications_queue_status_check` constraint. *(Already applied as migration 0035.)*

---

### Change 2 — Mobile: Persist current token and unregister old one on token rotation

**File:** `src/controllers/implementations/NotificationController.ts`

**Part A — Persist token after registration (`initializeNotifications`, line ~196):**

After `registerDevice(userId, fcmToken)` succeeds, write the token to AsyncStorage under a stable key (e.g. `@sabiqoon/fcm_token`).

**Part B — On refresh, unregister old token before registering new one (`setupTokenRefreshListener`, line ~216):**

When `onTokenRefresh` fires with `newToken`:
1. Read `@sabiqoon/fcm_token` from AsyncStorage.
2. If the stored token exists and differs from `newToken`, call `unregisterDevice(userId, oldToken)`.
3. Call `registerDevice(userId, newToken)`.
4. Write `newToken` to AsyncStorage.

This covers reinstalls: Firebase invalidates the old token and issues a new one, `onTokenRefresh` fires, and the stale row is deleted from `user_devices`.

---

### Change 3 — Mobile: Detect stale token on cold login

**File:** `src/controllers/implementations/NotificationController.ts`  
**Method:** `initializeNotifications` (line ~167)

Between getting the FCM token (step 3) and calling `registerDevice` (step 4), add:

1. Read `@sabiqoon/fcm_token` from AsyncStorage.
2. If the stored token exists and differs from the newly fetched token, call `unregisterDevice(userId, storedToken)` to remove the now-dead row from `user_devices`.
3. Proceed with `registerDevice(userId, newToken)` and persist the new token.

This covers the case where the app was reinstalled and the token changed while the user was logged out (no `onTokenRefresh` fires in that window).

---

## Implementation Order

| Step | Change | Files |
|------|--------|-------|
| 1 | Backend: delete bad tokens post-dispatch | `push-notification-dispatcher-service.ts` |
| 2 | Mobile: persist token to AsyncStorage after registration | `NotificationController.ts` |
| 3 | Mobile: unregister old token on cold login | `NotificationController.ts` |
| 4 | Mobile: unregister old token on token rotation | `NotificationController.ts` |
| 5 | Deploy updated `push-notification-dispatcher` to both projects | CLI: `supabase functions deploy` |

Steps 2–4 are all in the same file and can be done together.

---

## Out of Scope

- The `mismatched-credential` tokens that already exist in `user_devices` for both projects will be cleaned up automatically once Change 1 is deployed and the next notification is dispatched. No manual DB cleanup needed.
- No new repository methods are needed on the backend.
- No new API endpoints are needed on the mobile side — `unregisterDevice` already calls the existing `unregister` action on `engagement-management/notification`.
