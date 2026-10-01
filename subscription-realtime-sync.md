# Plan: Real-time Subscription Sync + Full UI Adoption

## Background

The app's subscription state (`subscriptionTier`, `activeSeasonalPackIds`) is stored in Redux and persisted via AsyncStorage. It is only refreshed in four places: sign-in, auth screen completion, profile screen mount, and profile save. This means admin-granted subscription changes are invisible until the next app restart or profile screen visit.

### Current gaps

| Gap | Impact |
|---|---|
| No periodic or event-driven subscription refresh | Admin granting/revoking premium is invisible until next restart |
| Catalog `isEntitled` per quality comes from API — stale after tier change | Lock icons don't update without a manual pull-to-refresh |
| `GemScreen` has no paywall handling for 403 | Free user hitting a gated gem sees a generic error state |
| `CatalogScreen` main view has no upsell banner | Only the dashboard has one; messaging is inconsistent |

### Confirmed: Supabase Realtime will work with existing client

The mobile `supabaseClient.ts` creates a singleton with `persistSession: true`, `autoRefreshToken: true`, and `storage: AsyncStorage`. Once the user is signed in, the Supabase JS SDK automatically attaches the user's JWT to the Realtime WebSocket handshake. RLS on `user_subscriptions` limits `postgres_changes` events to the current user's own rows. No extra auth wiring is needed — subscribing after login (inside the authenticated navigator) is sufficient.

### Why `user_subscriptions` and not `user_profiles`

Subscribing to `UPDATE` on `user_profiles` would fire a Realtime message on *any* column change — `last_activity_date`, `default_notification_time`, `profile_picture_url`, etc. `last_activity_date` alone likely updates every session, generating constant paid messages even though `subscription_tier` didn't change. Supabase Realtime's `filter` option is a row filter only — there is no native column-level filter.

`user_subscriptions` is the correct table to watch because rows are only written when a subscription is actually granted or revoked. In practice this fires a handful of times per user ever, not per session. The payload won't contain the derived `subscription_tier` string directly, so the client responds by calling `fetchUserProfile` — one extra network call that only happens on a real subscription event.

---

## Phase 1 — Enable Supabase Realtime on `user_subscriptions`

**File:** `supabase/migrations/0045_realtime_user_subscriptions.sql`

```sql
-- Allow Realtime to broadcast full row data (old + new)
ALTER TABLE public.user_subscriptions REPLICA IDENTITY FULL;

-- Add table to the Supabase Realtime publication
ALTER PUBLICATION supabase_realtime ADD TABLE public.user_subscriptions;
```

`REPLICA IDENTITY FULL` is required so the Realtime payload includes the complete new row on `UPDATE` events (e.g. status changing to `cancelled`).

**Deploy:** `supabase db push --linked` to project `znjkyjbbycnajxepyjfi`.

---

## Phase 2 — `useSubscriptionSync` hook

**File:** `src/hooks/useSubscriptionSync.ts`

A new hook mounted once inside the authenticated navigator. It owns two responsibilities:

### 2a. Supabase Realtime channel

Subscribe to `INSERT` and `UPDATE` events on `user_subscriptions` filtered to the current user:

```ts
supabase
  .channel('user-subscription-changes')
  .on('postgres_changes', {
    event: 'INSERT',
    schema: 'public',
    table: 'user_subscriptions',
    filter: `user_id=eq.${userId}`,
  }, handleSubscriptionChange)
  .on('postgres_changes', {
    event: 'UPDATE',
    schema: 'public',
    table: 'user_subscriptions',
    filter: `user_id=eq.${userId}`,
  }, handleSubscriptionChange)
  .subscribe();
```

`INSERT` fires when premium or a seasonal pack is granted. `UPDATE` fires when a subscription row is set to `cancelled` or `expired` (revoke).

On receiving either event:
1. Dispatch `fetchUserProfile(userId)` — gets fresh `subscriptionTier` + `activeSeasonalPackIds` in one call
2. On fulfillment, dispatch RTK Query cache invalidation (Phase 3)

Clean up: remove the channel on logout or hook unmount.

### 2b. AppState foreground fallback

Realtime connections can drop silently on poor networks or when the OS suspends the app. As a safety net, listen to `AppState` changes:

- On transition to `'active'`: call `fetchUserProfile` if more than 5 minutes have elapsed since the last successful fetch
- Track `lastFetchAt` in a `useRef` — no Redux state needed

This guarantees subscription status is at most 5 minutes stale even if the Realtime channel is down.

### Mount point

Add `<SubscriptionSync />` (a renderless component wrapping the hook) inside `MainNavigator` after the user is authenticated, so `userId` is always available.

---

## Phase 3 — RTK Query cache invalidation on subscription change

When `fetchUserProfile` fulfills after a Realtime event or AppState foreground trigger, invalidate the following RTK Query cache tags so affected screens re-fetch automatically:

| Tag | Screens affected |
|---|---|
| `Qualities` | CatalogScreen, CatalogSeeMoreScreen — refreshes `isEntitled` per quality |
| `HabitStreams` | HabitsScreen — adhoc habits may become accessible |
| `DashboardParams` | Dashboard — habit streams list may change |

Invalidation is dispatched via `store.dispatch(supabaseEdgeFunctionsApi.util.invalidateTags([...]))` inside the `useSubscriptionSync` hook after the profile fetch resolves.

Note: the `setSubscriptionTier` synchronous action from the earlier plan draft is no longer needed — `fetchUserProfile` updates the full profile in Redux including both `subscriptionTier` and `activeSeasonalPackIds`.

---

## Phase 4 — UI adoption sweep

### 4a. `GemScreen` — paywall on 403

**File:** `src/views/screens/GemScreen/index.tsx`

The backend returns `SUBSCRIPTION_REQUIRED` (HTTP 403) when a free user tries to access a premium gem. Currently this surfaces as a generic error state. Change the error handler to:
- Detect `error?.data?.error === 'SUBSCRIPTION_REQUIRED' || error?.status === 403`
- Show `PremiumPaywallBottomSheet` with `context="quality"` instead of the error state

Consistent with the pattern already implemented in `DashBoardScreen` and `HabitsScreen`.

### 4b. `CatalogScreen` main view — upsell banner

**File:** `src/views/screens/CatalogScreen/CatalogScreen.view.tsx`

Add the same gold upsell banner that exists on the dashboard, rendered when `isFreeTier` from `useSubscription()`. Keeps upgrade messaging consistent across the app.

### 4c. `HabitsScreen` — confirm gate is complete

**File:** `src/views/screens/HabitsScreen/hooks.ts`

The adhoc tab is gated via `isEntitledToAdhoc` but verify the `SUBSCRIPTION_REQUIRED` catch in `onStartHabitStream` correctly surfaces the paywall for quality habit streams too (not just adhoc). Add coverage if missing.

### 4d. Paywall dismiss → profile refresh

**File:** `src/views/components/PremiumPaywallBottomSheet/index.tsx`

When the paywall is dismissed after a successful upgrade (future in-app purchase flow), dispatch `fetchUserProfile` to pick up the new tier immediately rather than waiting for the next Realtime event or AppState trigger.

---

## Execution order

| # | Step | Files touched | Effort |
|---|---|---|---|
| 1 | Migration + deploy Realtime on `user_subscriptions` | `migrations/0045_...sql` | Small |
| 2 | `useSubscriptionSync` hook | `src/hooks/useSubscriptionSync.ts` | Medium |
| 3 | Mount hook in navigator | `MainNavigator.tsx` | Small |
| 4 | RTK Query invalidation on subscription change | `useSubscriptionSync.ts` | Small |
| 5 | `GemScreen` paywall on 403 | `GemScreen/index.tsx` | Small |
| 6 | Catalog upsell banner | `CatalogScreen.view.tsx` | Small |
| 7 | HabitsScreen gate audit | `HabitsScreen/hooks.ts` | Small |
| 8 | Paywall dismiss → profile refresh | `PremiumPaywallBottomSheet/index.tsx` | Small |
