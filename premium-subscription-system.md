# Premium Subscription System

## Summary

Introduces a two-tier subscription system (Free / Premium) plus Seasonal Packs (Ramadan, Dhul Hijjah). Subscription management is admin-toggle only in this phase — no in-app payment SDK yet. The backend enforces entitlement on every gated API call, and the mobile app surfaces locked states and a reusable paywall bottom sheet across screens.

## Deployment Target

All backend changes (migrations and edge functions) must be deployed to Supabase project **`znjkyjbbycnajxepyjfi`**.

```bash
# Apply migrations
supabase db push --project-ref znjkyjbbycnajxepyjfi

# Deploy a specific edge function
supabase functions deploy <function-name> --project-ref znjkyjbbycnajxepyjfi

# Deploy the new subscription-management function
supabase functions deploy subscription-management --project-ref znjkyjbbycnajxepyjfi
```

**Never run `supabase db push` or `supabase functions deploy` without explicit user confirmation**, regardless of environment.

## Key Decisions


| Decision                 | Choice                                                               |
| ------------------------ | -------------------------------------------------------------------- |
| Payment provider         | Admin-toggle only (no RevenueCat / IAP yet)                          |
| Free quality designation | Admin-assigned in admin portal                                       |
| Existing users           | Grandfathered as `legacy_premium` (permanent, never expires)         |
| Adhoc habits             | Premium-only; completely unavailable on free tier                    |
| Accountability partners  | Unlimited for all tiers, no gate                                     |
| Expired premium habits   | Visible but read-only (stats shown, daily check-in blocked)          |
| Free user catalog view   | Premium qualities visible with lock icon; tap → paywall bottom sheet |
| Paywall UX               | Reuse existing bottom-sheet design from `PremiumAdhocState`          |
| Seasonal pack access     | Anyone can purchase (free or premium users)                          |
| Consultation             | Out of scope                                                         |


---

## Phase 1 — Database Migrations

**Migration 0037 — Add `subscription_tier` to `user_profiles`**

```sql
CREATE TYPE public.subscription_tier AS ENUM ('free', 'premium', 'legacy_premium');

ALTER TABLE public.user_profiles
  ADD COLUMN subscription_tier public.subscription_tier NOT NULL DEFAULT 'free';

-- Grandfather all existing users as permanent premium
UPDATE public.user_profiles SET subscription_tier = 'legacy_premium';
```

`legacy_premium` = permanent, never expires, never revocable by subscription logic. The admin portal toggle switches between `'free'` and `'premium'` — `'legacy_premium'` is only set by this migration.

---

**Migration 0038 — Add `tier` to `qualities` and `habits`**

```sql
CREATE TYPE public.content_tier AS ENUM ('free', 'premium', 'seasonal');

ALTER TABLE public.qualities
  ADD COLUMN tier public.content_tier NOT NULL DEFAULT 'premium';

ALTER TABLE public.habits
  ADD COLUMN tier public.content_tier NOT NULL DEFAULT 'premium';

-- Ad-hoc habits are identified by gem_id IS NULL. Mark all gem-linked habits as
-- 'premium' by default; admins will explicitly mark 2 qualities as 'free'.
-- Existing users are legacy_premium so this has no immediate impact.
```

---

**Migration 0039 — Create `seasonal_packs` table**

```sql
CREATE TYPE public.seasonal_pack_type AS ENUM ('ramadan2027', 'dhul_hijjah2027', 'custom');

CREATE TABLE public.seasonal_packs (
  id         INTEGER PRIMARY KEY GENERATED ALWAYS AS IDENTITY,
  name       VARCHAR(255) NOT NULL,
  slug       VARCHAR(100) NOT NULL UNIQUE,
  description TEXT,
  type       public.seasonal_pack_type NOT NULL,
  icon_url   TEXT,
  price_usd  NUMERIC(10,2),
  state      public.publish_state NOT NULL DEFAULT 'DRAFT',
  starts_at  TIMESTAMPTZ,
  ends_at    TIMESTAMPTZ,
  created_at TIMESTAMPTZ DEFAULT NOW() NOT NULL,
  updated_at TIMESTAMPTZ DEFAULT NOW() NOT NULL
);

ALTER TABLE public.seasonal_packs ENABLE ROW LEVEL SECURITY;

-- master_admin: full access including publishing
CREATE POLICY "master_admin_manage_seasonal_packs" ON public.seasonal_packs
  FOR ALL TO authenticated
  USING (auth.users.raw_user_meta_data->>'role' = 'master_admin')
  WITH CHECK (auth.users.raw_user_meta_data->>'role' = 'master_admin');

-- admin: can INSERT and UPDATE, but state must remain DRAFT
CREATE POLICY "admin_draft_only_seasonal_packs" ON public.seasonal_packs
  FOR INSERT TO authenticated
  WITH CHECK (
    auth.users.raw_user_meta_data->>'role' = 'admin'
    AND state = 'DRAFT'
  );

CREATE POLICY "admin_update_draft_seasonal_packs" ON public.seasonal_packs
  FOR UPDATE TO authenticated
  USING (
    auth.users.raw_user_meta_data->>'role' = 'admin'
    AND state = 'DRAFT'
  )
  WITH CHECK (
    auth.users.raw_user_meta_data->>'role' = 'admin'
    AND state = 'DRAFT'   -- cannot transition state to PUBLISHED
  );

-- All authenticated users can read published packs
CREATE POLICY "authenticated_read_published_seasonal_packs" ON public.seasonal_packs
  FOR SELECT TO authenticated
  USING (state = 'PUBLISHED');
```

---

**Migration 0040 — Create `seasonal_pack_qualities` junction table**

```sql
CREATE TABLE public.seasonal_pack_qualities (
  seasonal_pack_id INTEGER NOT NULL REFERENCES public.seasonal_packs(id) ON DELETE CASCADE,
  quality_id       INTEGER NOT NULL REFERENCES public.qualities(id) ON DELETE CASCADE,
  PRIMARY KEY (seasonal_pack_id, quality_id)
);

-- A quality with tier='seasonal' gets its pack membership via this table.
-- qualities.tier must equal 'seasonal' for records here.
```

---

**Migration 0041 — Add `seasonal_pack_id` to `habits`, update RLS**

```sql
ALTER TABLE public.habits
  ADD COLUMN seasonal_pack_id INTEGER REFERENCES public.seasonal_packs(id) ON DELETE SET NULL;

CREATE INDEX idx_habits_seasonal_pack_id ON public.habits(seasonal_pack_id);

-- Seasonal ad-hoc habits: gem_id IS NULL AND tier = 'seasonal' AND seasonal_pack_id IS NOT NULL
```

---

**Migration 0042 — Create `user_subscriptions` table**

```sql
CREATE TYPE public.subscription_type AS ENUM (
  'premium_monthly',
  'premium_annual',
  'seasonal_pack'
);

CREATE TYPE public.subscription_status AS ENUM ('active', 'expired', 'cancelled');

CREATE TABLE public.user_subscriptions (
  id                 INTEGER PRIMARY KEY GENERATED ALWAYS AS IDENTITY,
  user_id            UUID NOT NULL REFERENCES public.user_profiles(user_id) ON DELETE CASCADE,
  subscription_type  public.subscription_type NOT NULL,
  seasonal_pack_id   INTEGER REFERENCES public.seasonal_packs(id) ON DELETE SET NULL,
  status             public.subscription_status NOT NULL DEFAULT 'active',
  starts_at          TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  expires_at         TIMESTAMPTZ,          -- NULL = never expires (admin grant)
  granted_by_admin   BOOLEAN NOT NULL DEFAULT FALSE,
  created_at         TIMESTAMPTZ DEFAULT NOW() NOT NULL,
  updated_at         TIMESTAMPTZ DEFAULT NOW() NOT NULL,

  CONSTRAINT seasonal_pack_required
    CHECK (subscription_type != 'seasonal_pack' OR seasonal_pack_id IS NOT NULL)
);

CREATE INDEX idx_user_subscriptions_user_id ON public.user_subscriptions(user_id);
CREATE INDEX idx_user_subscriptions_status  ON public.user_subscriptions(status);

ALTER TABLE public.user_subscriptions ENABLE ROW LEVEL SECURITY;

CREATE POLICY "users_read_own_subscriptions" ON public.user_subscriptions
  FOR SELECT TO authenticated
  USING (user_id = auth.uid());

CREATE POLICY "admins_manage_subscriptions" ON public.user_subscriptions
  FOR ALL TO authenticated
  USING (auth.users.raw_user_meta_data->>'role' IN ('master_admin', 'admin'));
```

---

## Phase 2 — Backend: New Subscription Infrastructure

### 2.1 New repository: `subscription-repository.ts`

Location: `supabase/functions/repositories/subscription-repository.ts`

Methods:

- `getUserSubscriptionTier(userId: string): Promise<'free' | 'premium' | 'legacy_premium'>` — reads `user_profiles.subscription_tier`
- `getActiveSubscriptions(userId: string): Promise<UserSubscription[]>` — reads `user_subscriptions` WHERE `status = 'active'` AND (`expires_at IS NULL` OR `expires_at > NOW()`)
- `getActiveSeasonalPackIds(userId: string): Promise<number[]>` — returns seasonal_pack_ids from active seasonal_pack subscriptions
- `grantPremium(userId: string, grantedByAdmin: boolean): Promise<void>` — inserts into `user_subscriptions` AND updates `user_profiles.subscription_tier = 'premium'`
- `revokePremium(userId: string): Promise<void>` — sets subscription status to `cancelled` AND updates `user_profiles.subscription_tier = 'free'`

### 2.2 New service: `subscription-service.ts`

Location: `supabase/functions/services/subscription-service.ts`

```typescript
class SubscriptionService {
  constructor(private subscriptionRepo: SubscriptionRepository) {}

  // Returns the user's effective entitlement level
  async getUserTier(userId: string): Promise<SubscriptionTier>

  // Returns true for premium, legacy_premium, or active seasonal pack owner
  async isEntitledToQuality(userId: string, qualityTier: ContentTier, qualityId: number): Promise<boolean>

  // Returns true for premium or legacy_premium
  async isEntitledToAdhocHabits(userId: string): Promise<boolean>

  // Returns list of pack IDs the user has active access to
  async getActiveSeasonalPackIds(userId: string): Promise<number[]>
}
```

Entitlement logic:

- `legacy_premium` → entitled to everything (qualities, adhoc habits, all seasonal packs)
- `premium` → entitled to all `free` and `premium` qualities/habits; NOT automatically entitled to seasonal packs (separate purchase)
- `free` → entitled to `free` tier qualities/habits only; NOT adhoc habits

### 2.3 Wire into DI container

File: `supabase/functions/shared/di-container.ts`

Add `SubscriptionRepository` and `SubscriptionService` to the container. Both are instantiated once and injected wherever needed.

---

## Phase 3 — Backend: Enforcement in Existing Services

### 3.1 `habit-stream-service.ts` — `createHabitStream()`

Inject `SubscriptionService`. Before creating a stream:

1. Look up the habit's `tier` via `habits-repository`
2. If `tier = 'premium'` → call `subscriptionService.isEntitledToAdhocHabits(userId)` for ad-hoc, or `isEntitledToQuality()` for quality-linked habits
3. If not entitled → throw `SubscriptionRequiredError` (HTTP 403, code `SUBSCRIPTION_REQUIRED`)

For habits where the quality tier is `premium` and the user previously held premium but it's now expired: allow `getHabitStreams()` (reading) but block `createHabitStreamProgress()` (new check-in entry).

### 3.2 `qualities-service.ts` — `getQualityCatalog()`

Inject `SubscriptionService`. The catalog response for each quality must include:

- `tier: 'free' | 'premium' | 'seasonal'`
- `isEntitled: boolean` (derived server-side from user's subscription)
- `seasonalPackId?: number` (if tier = 'seasonal')

This lets the mobile client show lock state without making a separate entitlement call.

### 3.3 `activity-management` endpoint — Ad-hoc habit creation

When a user creates an ad-hoc habit stream, check `subscriptionService.isEntitledToAdhocHabits(userId)`. Reject with `SUBSCRIPTION_REQUIRED` if not entitled.

### 3.4 Error convention

Add a shared error type:

```typescript
// supabase/functions/shared/errors.ts
export class SubscriptionRequiredError extends Error {
  readonly code = 'SUBSCRIPTION_REQUIRED'
  readonly httpStatus = 403
  constructor(message = 'This feature requires a premium subscription') {
    super(message)
  }
}
```

All edge function index files catch this error and return:

```json
{ "error": "SUBSCRIPTION_REQUIRED", "message": "..." }
```

with HTTP 403.

---

## Phase 4 — Backend: User Profile API Update

The `user-management` edge function's `GET /profile` response must include:

- `subscription_tier: 'free' | 'premium' | 'legacy_premium'`
- `active_seasonal_pack_ids: number[]`

Update `UserService.getUserProfile()` to join this data and include it in the response. Update the response model/mapper accordingly.

---

## Phase 5 — Backend: `subscription-management` Edge Function

New deployable edge function at `supabase/functions/subscription-management/`.

### Structure

```
supabase/functions/subscription-management/
  index.ts          ← CORS, routing, error handling
  models/
    requests.ts     ← typed request bodies
    responses.ts    ← typed response shapes
```

### Routes

**`GET /subscription-management/seasonal-packs`**

Returns all published seasonal packs visible to the caller, with per-pack entitlement status.

Response shape:
```typescript
{
  seasonalPacks: Array<{
    id: number
    name: string
    slug: string
    type: 'ramadan2027' | 'dhul_hijjah2027' | 'custom'
    iconUrl: string | null
    description: string | null
    startsAt: string | null
    endsAt: string | null
    priceUsd: number | null
    isOwned: boolean   // true if caller has an active subscription for this pack
  }>
}
```

Calls `SubscriptionService.getSeasonalPacksForUser(userId)` which joins `seasonal_packs` with `user_subscriptions` to populate `isOwned`.

---

**`POST /subscription-management/seasonal-packs/:packId/activate`**

Admin-only. Grants a specific user access to a seasonal pack by inserting a `user_subscriptions` row of type `seasonal_pack`.

Request body:
```typescript
{
  userId: string      // UUID of the target user
  expiresAt?: string  // ISO timestamp; omit for no expiry
}
```

Response:
```typescript
{ success: true, subscriptionId: number }
```

Authorization: caller's JWT must have `role = 'master_admin'` or `role = 'admin'` in `raw_user_meta_data`. Return HTTP 403 otherwise.

Calls `SubscriptionRepository.grantSeasonalPack(userId, packId, expiresAt, grantedByAdmin: true)`.

---

### DI wiring

`subscription-management/index.ts` instantiates its own `SubscriptionRepository` and `SubscriptionService` via the shared DI container (same pattern as other edge functions). It does not depend on any other function group.

### Error handling

Catches `SubscriptionRequiredError` (403), standard `NotFoundError` (404), and unhandled errors (500) at the top-level router — same pattern as other edge functions.

These are the only two endpoints needed for Phase 1. Payment flow and user-initiated purchase are deferred.

---

## Phase 6 — Admin Portal Changes

### 6.1 Quality editor — Add `tier` field

File: `app/(admin-panel)/qualities/[qualityId]/` (edit form component)

Add a `tier` select field to the quality create/edit form:

- Options: Free | Premium | Seasonal
- When "Seasonal" is selected: show a secondary dropdown listing published `seasonal_packs` — the selected pack ID is saved to `seasonal_pack_qualities`
- Update `qualities-repository.ts` to read/write the `tier` column and manage the `seasonal_pack_qualities` junction row

### 6.2 New admin section: Seasonal Packs

New route: `app/(admin-panel)/seasonal-packs/`

Pages:

- `page.tsx` — list all seasonal packs (name, type, dates, state, quality count)
- `new/page.tsx` — create seasonal pack form (name, slug, type, dates, icon_url, description, price, state)
- `[packId]/edit/page.tsx` — edit form + list of linked qualities (with ability to add/remove)
- `[packId]/habits/page.tsx` — manage seasonal ad-hoc habits linked to this pack

Add nav link to the admin sidebar.

New service: `lib/services/supabase/services/database/seasonal-packs-repository.ts`

- CRUD for `seasonal_packs`
- `linkQuality(packId, qualityId)` / `unlinkQuality(packId, qualityId)` for the junction table

### 6.3 New admin section: User Management

New route: `app/(admin-panel)/users/`

Pages:

- `page.tsx` — paginated list of users (display_name, email, subscription_tier, created_at)
- `[userId]/page.tsx` — user detail with:
  - Current subscription tier badge
  - "Grant Premium" / "Revoke Premium" toggle button (calls backend)
  - Subscription history table (from `user_subscriptions`)

This requires calling the Supabase auth admin API (via `SUPABASE_SERVICE_ROLE_KEY`) for user listing, similar to the existing `user-repository.ts` in the admin portal. Add new API route handlers under `app/api/users/`.

---

## Phase 7 — Mobile App: Data Layer

### 7.1 Update `UserProfile` model

File: `src/models/UserProfile.ts`

```typescript
export type SubscriptionTier = 'free' | 'premium' | 'legacy_premium'

export interface UserProfile {
  user_id: string
  displayName?: string
  email?: string
  profilePictureUrl?: string
  last_activity_date?: string
  defaultNotificationTime?: string
  subscriptionTier: SubscriptionTier          // new
  activeSeasonalPackIds: number[]             // new
}
```

### 7.2 Update `Quality` model / catalog response types

Add to the quality catalog item interface (in `src/controllers/interfaces/` or `src/models/`):

```typescript
tier: 'free' | 'premium' | 'seasonal'
isEntitled: boolean
seasonalPackId?: number
```

### 7.3 New hook: `useSubscription()`

File: `src/hooks/useSubscription.ts`

```typescript
export function useSubscription() {
  const profile = useAppSelector(state => state.userProfile.profile)
  const tier = profile?.subscriptionTier ?? 'free'
  return {
    tier,
    isPremium: tier === 'premium' || tier === 'legacy_premium',
    isFreeTier: tier === 'free',
    activeSeasonalPackIds: profile?.activeSeasonalPackIds ?? [],
    isEntitledToSeasonal: (packId: number) =>
      tier === 'legacy_premium' ||
      (profile?.activeSeasonalPackIds ?? []).includes(packId),
  }
}
```

---

## Phase 8 — Mobile App: `PremiumPaywallBottomSheet` Component

File: `src/views/components/PremiumPaywallBottomSheet/index.tsx`

Extract and generalize the existing `PremiumAdhocState` design from `HabitsScreen.view.tsx` into a reusable bottom sheet modal.

Props:

```typescript
interface PremiumPaywallBottomSheetProps {
  visible: boolean
  onClose: () => void
  context: 'adhoc_habit' | 'quality' | 'profile' | 'expired'
  // context changes the description text:
  // - 'adhoc_habit': "Adhoc habits are available to Premium members..."
  // - 'quality': "This Quality is available to Premium members..."
  // - 'profile': "Upgrade to unlock all Qualities and ad-hoc habits..."
  // - 'expired': "Your Premium subscription has expired. Resubscribe to continue..."
}
```

Design (matches existing `PremiumAdhocState`):

- Crown icon in a circle
- "Premium Feature" or "Premium Required" title (based on context)
- Context-specific description text
- "Upgrade — Monthly ✦" button (primary, placeholder `onPress`)
- "Upgrade — Annual ✦" button (secondary, price shown, placeholder `onPress`)
- "Restore Purchase" text link (placeholder)
- Dismiss handle at the top

The `HabitsScreen.view.tsx` `PremiumAdhocState` component should be replaced by a call to `PremiumPaywallBottomSheet` once this is built, removing the duplicate code.

---

## Phase 9 — Mobile App: Per-Screen Changes

### 9.1 Catalog Screen (`src/views/screens/CatalogScreen/`)

**View layer (`CatalogScreen.view.tsx`):**

- Quality cards with `isEntitled = false`: render a lock icon overlay (`MaterialCommunityIcons name='lock'`) in the top-right corner. The card itself is still tappable.
- Premium qualities: show a small crown badge below the quality name.
- Seasonal qualities: show a special badge (e.g., crescent moon icon for Ramadan).

**Index layer (`index.tsx`):**

- On quality card press: if `!isEntitled` → call `actions.onLockedQualityPress(quality)` instead of navigating
- `onLockedQualityPress`: sets `uiState.paywallVisible = true` and stores the context

**New state in `types.ts`:**

```typescript
paywallVisible: boolean
paywallContext: 'quality' | 'adhoc_habit' | 'expired'
```

**Controller layer:**

- Pass `isEntitled` flag per quality down from the catalog API response (already added in Phase 4)

### 9.2 Dashboard Screen (`src/views/screens/DashBoardScreen/`)

For free-tier users, render a premium upsell card within the existing scroll view, positioned after the quality progress section.

**New component: `PremiumUpsellCard`** (local to DashboardScreen or in `src/views/components/`)

Design:

- Gradient background (gold/amber, matching crown theme)
- Crown icon left side
- "Unlock All Qualities" title
- "Upgrade to Premium to access all Islamic qualities and habits" subtitle
- "Upgrade ✦" button → triggers `PremiumPaywallBottomSheet` with `context='profile'`

Only render this card when `isFreeTier === true`. Hidden for premium/legacy_premium users.

### 9.3 Profile Screen (`src/views/screens/ProfileScreen/`)

**Add a new "Subscription" section** to the profile view, placed above the settings/logout section:

For free tier:

```
[Crown icon]  Free Tier
You have access to 2 free Qualities.
[Upgrade to Premium ✦]  ← button → PremiumPaywallBottomSheet context='profile'
```

For premium:

```
[Crown icon filled gold]  Premium Member ✦
Your subscription is active.
[Subscription details link (future)]
```

For legacy_premium:

```
[Crown icon filled gold]  Premium Member ✦
Founding member. Premium access for life.
```

For seasonal pack display (all tiers): If the user has active seasonal subscriptions, show a list of active pack badges below their tier info.

**Types update (`types.ts`):** Add `onUpgradePress: () => void` to `ProfileScreenActions`
**Index update:** Handle `showPaywall` state, wire `onUpgradePress` to set it true

### 9.4 Habits Screen (`src/views/screens/HabitsScreen/`)

**Adhoc tab gating:**
Currently `PremiumAdhocState` shows only when `adhocTabIsEmpty`. The actual gate should be based on subscription, not emptiness.

Change in `HabitsScreen.view.tsx`:

```typescript
// Before:
uiState.activeTab === 'adhoc' && adhocTabIsEmpty ? <PremiumAdhocState /> : ...

// After:
uiState.activeTab === 'adhoc' && !uiState.isAdhocEntitled
  ? null // PremiumPaywallBottomSheet is shown, tab appears "locked"
  : uiState.activeTab === 'adhoc' && adhocTabIsEmpty
    ? <EmptyAdhocState onCreateHabit={actions.onCreateAdhocHabit} />
    : ...
```

When the adhoc tab is tapped by a non-entitled user, auto-trigger `PremiumPaywallBottomSheet` with `context='adhoc_habit'`.

Add `isAdhocEntitled: boolean` to `HabitsScreenUIState` (derived from `useSubscription().isPremium`).

**Quality habits — expired premium lock:**
In `QualityHabitGroup` component (`src/views/components/habits/QualityHabitGroup`): accept an `isLocked: boolean` prop. When locked:

- Render a premium icon overlay on each habit card
- Tapping a locked habit card: show `PremiumPaywallBottomSheet` with `context='expired'` instead of the normal `HabitDetailBottomSheet`
- Stats (streak, days completed) still visible in read-only fashion

The controller in `HabitsScreen/index.tsx` determines `isLocked` per quality group: a quality group is locked when `quality.tier === 'premium' && !isPremium && group has existing streams` (expired premium state). Free users who never had premium don't see these groups at all (they aren't returned by the API).

---

## Phase 10 — Mobile App: API Error Handling

In the RTK Query base query or in the `SupabaseEdgeFunctions` service layer:

When a response comes back with HTTP 403 and `error === 'SUBSCRIPTION_REQUIRED'`:

1. Dispatch a Redux action `setPaywallRequired(true)` (or use a local navigation event)
2. The active screen handles this by showing `PremiumPaywallBottomSheet`

Alternatively, each screen's controller catches the specific error from RTK Query and sets its own `paywallVisible` state. This is simpler and avoids global state for a UI concern.

Recommended: screen-level handling. Each controller's `useEffect` or mutation handler checks:

```typescript
if (error?.data?.error === 'SUBSCRIPTION_REQUIRED') {
  setPaywallVisible(true)
}
```

---

## Implementation Order

Recommended sequence to allow incremental testing:

1. **Migrations 0037–0042** — schema is the foundation for everything
2. **Phase 2–3** — backend subscription service + enforcement (backend fully independent of frontend)
3. **Phase 4** — user profile API update (required before mobile can read tier)
4. **Phase 6** — admin portal (so qualities/users can be configured before mobile changes go live)
5. **Phase 7–8** — mobile data layer + shared bottom sheet component
6. **Phase 9** — per-screen mobile changes (can be done in parallel across screens)
7. **Phase 5 + 10** — seasonal packs endpoint + mobile error handling (can be done last)
8. **Phase 6.2–6.3** — admin seasonal packs + user management screens

---

## Out of Scope (Future Phases)

- RevenueCat / Apple IAP / Google Play Billing integration
- Receipt validation
- Subscription renewal webhooks
- Grace periods for expired subscriptions
- On-demand consultation feature
- Free trial flow
- Family/team subscription plans

