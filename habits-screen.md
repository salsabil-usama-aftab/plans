# Habits Screen Implementation Plan

## Overview

A new "Habits" screen accessible via a **5th bottom tab** in the app. It displays all habit streams a user has unlocked, split into two tabs:

- **Quality Habits tab** — habits linked to a gem/quality, grouped by quality with quality name and level displayed.
- **Ad-hoc tab** — habits with no quality or gem association, shown as a flat list.

Each habit item shows its description and status (Active / Available). Tapping a habit opens a bottom sheet:
- **Active** → informational sheet ("Habit is active — check Dashboard for stats")
- **Available** → sheet with day-range selector and a "Start Habit" button that calls the existing `start` action and navigates to the Dashboard

A new backend endpoint under `engagement-management` will power this screen. It is deployed to project `znjkyjbbycnajxepyjfi`.

---

## Design Clarifications

- The Figma design file could not be fetched (binary), so the following are based on verbal description + existing component conventions. Adjust styles to match actual design after review.
- Status labels: `ACTIVE` (user currently tracking) and `AVAILABLE` (NOT_STARTED — unlocked but not yet started).
- Ad-hoc habits = `habit.gem_id IS NULL AND habit.quality_id IS NULL`.
- Quality habits = habits where quality is derivable via `habit.gem_id → gems.quality_id` OR directly via `habit.quality_id`.

---

## 1. Backend

### 1.1 New Repository Method

**File:** `project-ihsan-backend/supabase/functions/repositories/habit-stream-progress-repository.ts`

Add a new method `listHabitStreamsForHabitsScreen` to `IHabitStreamProgressRepository` and `HabitStreamProgressRepository`:

```typescript
listHabitStreamsForHabitsScreen(
  userId: string,
): Promise<HabitStreamWithQualityInfo[]>
```

New type `HabitStreamWithQualityInfo`:
```typescript
export interface HabitStreamWithQualityInfo {
  id: number
  habit_text: string
  status: HabitStreamStatus
  start_date: string | null
  end_date: string | null
  habit_id: number | null
  gem_id: number | null       // from habits table
  quality_id: number | null   // effective quality (from gem or direct)
  quality_name: string | null
  quality_level: string | null // from levels.name
}
```

**Supabase query** (single round-trip using nested joins):
```typescript
const { data, error } = await supabaseClient
  .from('user_habit_streams')
  .select(`
    id, habit_text, status, start_date, end_date,
    habits(
      id, gem_id, quality_id,
      gems(quality_id, qualities(id, name, levels(name))),
      qualities(id, name, levels(name))
    )
  `)
  .eq('user_id', userId)
  .in('status', ['ACTIVE', 'NOT_STARTED'])
  .order('id', { ascending: true })
```

Service layer maps the raw result: quality is resolved from `habits.gems.qualities` first (gem-linked), falling back to `habits.qualities` (direct quality_id). If neither → ad-hoc.

### 1.2 New Service Method

**File:** `project-ihsan-backend/supabase/functions/services/habit-stream-service.ts`

Add to `IHabitStreamService` and `HabitStreamService`:

```typescript
buildHabitStreamsForHabitsScreen(userId: string): Promise<HabitsScreenPayload>
```

New types (in `shared/models/habit-stream-models.ts`):
```typescript
export interface HabitsScreenHabitItem {
  habitStreamId: number
  habitText: string
  status: 'ACTIVE' | 'AVAILABLE'   // NOT_STARTED maps to AVAILABLE
}

export interface HabitsScreenQualityGroup {
  qualityId: number
  qualityName: string
  qualityLevel: string
  habits: HabitsScreenHabitItem[]
}

export interface HabitsScreenPayload {
  qualityHabits: HabitsScreenQualityGroup[]
  adhocHabits: HabitsScreenHabitItem[]
}
```

**Logic:**
1. Call `listHabitStreamsForHabitsScreen(userId)`
2. Split into quality-linked and ad-hoc based on `quality_id` presence
3. Group quality-linked by `quality_id`, sort groups alphabetically by `quality_name`
4. Map `NOT_STARTED → 'AVAILABLE'`, `ACTIVE → 'ACTIVE'`
5. Return `{ qualityHabits, adhocHabits }`

### 1.3 New Controller Route

**File:** `project-ihsan-backend/supabase/functions/engagement-management/index.ts`

Add a new branch in the main router for entity `'habit-screen'`:

```
GET /functions/v1/engagement-management/habit-screen?userId={uuid}
```

Handler pseudocode:
```typescript
async function handleHabitScreenRequest(req: Request, url: URL) {
  if (req.method !== 'GET') return METHOD_NOT_ALLOWED

  const userId = url.searchParams.get('userId')
  // UUID validation (same pattern as other handlers)

  const payload = await habitStreamService.buildHabitStreamsForHabitsScreen(userId)
  return createSuccessResponse(payload, HTTP_STATUS.OK)
}
```

Add `'habit-screen'` to the entity routing block in the main `serve` handler.

### 1.4 OpenAPI Spec

**File:** `project-ihsan-backend/openapi.yaml`

Add under `paths`:

```yaml
/engagement-management/habit-screen:
  get:
    summary: Get all habit streams for Habits screen
    description: >
      Returns all of a user's unlocked habit streams (ACTIVE and NOT_STARTED),
      split into quality-grouped habits and ad-hoc habits (no quality/gem link).
    tags:
      - Habit Streams
    parameters:
      - name: userId
        in: query
        required: true
        schema:
          type: string
          format: uuid
    responses:
      '200':
        description: Habits screen payload
        content:
          application/json:
            schema:
              type: object
              properties:
                data:
                  type: object
                  properties:
                    qualityHabits:
                      type: array
                      items:
                        $ref: '#/components/schemas/HabitsScreenQualityGroup'
                    adhocHabits:
                      type: array
                      items:
                        $ref: '#/components/schemas/HabitsScreenHabitItem'
      '400':
        description: Missing or invalid userId
      '500':
        description: Internal server error
```

Add component schemas for `HabitsScreenQualityGroup` and `HabitsScreenHabitItem`.

### 1.5 Deploy

After implementing and testing locally:
```bash
cd project-ihsan-backend
supabase functions deploy engagement-management --project-ref znjkyjbbycnajxepyjfi
```

> **Do not deploy until user gives explicit go-ahead.**

---

## 2. Mobile App

### 2.1 New Models

**File:** `project-ihsan/src/models/HabitsScreenModels.ts` (new file)

```typescript
export interface HabitsScreenHabitItem {
  habitStreamId: number
  habitText: string
  status: 'ACTIVE' | 'AVAILABLE'
}

export interface HabitsScreenQualityGroup {
  qualityId: number
  qualityName: string
  qualityLevel: string
  habits: HabitsScreenHabitItem[]
}

export interface GetHabitsScreenResponse {
  data: {
    qualityHabits: HabitsScreenQualityGroup[]
    adhocHabits: HabitsScreenHabitItem[]
  }
}

export interface GetHabitsScreenRequest {
  userId: string
}
```

### 2.2 SupabaseEdgeFunctions Service Method

**File:** `project-ihsan/src/services/SupabaseEdgeFunctions.ts`

Add method:
```typescript
async getHabitsScreen(userId: string): Promise<GetHabitsScreenResponse>
```

Calls: `GET engagement-management/habit-screen?userId={userId}`

### 2.3 RTK Query Endpoint

**File:** `project-ihsan/src/store/api/supabaseEdgeFunctionsApi.ts`

Add a new `getHabitsScreen` query:
```typescript
getHabitsScreen: builder.query<GetHabitsScreenResponse, GetHabitsScreenRequest>({
  queryFn: async ({ userId }) => {
    // call SupabaseEdgeFunctions.getHabitsScreen(userId)
  },
  providesTags: (result, error, { userId }) => [
    { type: 'HabitScreen', id: userId },
  ],
})
```

Cache tag `'HabitScreen'` must be invalidated after:
- `startHabitStream` succeeds (changes AVAILABLE → ACTIVE)

Add `{ type: 'HabitScreen', id: user_id }` to `invalidatesTags` of the `startHabitStream` mutation.

### 2.4 HabitsScreen

All files under: `project-ihsan/src/views/screens/HabitsScreen/`

#### `types.ts`
```typescript
import { HabitsScreenHabitItem, HabitsScreenQualityGroup } from '../../../models/HabitsScreenModels'

export type HabitsTab = 'quality' | 'adhoc'

export interface HabitsScreenUIState {
  activeTab: HabitsTab
  selectedHabit: HabitsScreenHabitItem | null
  selectedQualityGroup: HabitsScreenQualityGroup | null
  showBottomSheet: boolean
  selectedDays: 7 | 14 | 21 | 30
  isStartingHabit: boolean
}

export interface HabitsScreenActions {
  onTabChange: (tab: HabitsTab) => void
  onHabitPress: (habit: HabitsScreenHabitItem, group?: HabitsScreenQualityGroup) => void
  onCloseBottomSheet: () => void
  onDaySelect: (days: 7 | 14 | 21 | 30) => void
  onStartHabit: () => void
}
```

#### `hooks.ts`

`useHabitsScreen()` hook:
- Reads `userId` from Redux `userProfileSlice`
- Calls `useGetHabitsScreenQuery({ userId })`
- Manages `HabitsScreenUIState`
- `onStartHabit` handler:
  1. Calls `startHabitStream` mutation with `{ habitStreamId: selectedHabit.habitStreamId, userId, startDate: today, endDate: today + selectedDays - 1 }`
  2. On success → close bottom sheet, navigate to Dashboard tab
- Returns props for view

#### `index.tsx` (container)
Imports `useHabitsScreen`, renders `HabitsScreen.view.tsx` with all props.

#### `HabitsScreen.view.tsx` (pure view)
Props: data, uiState, actions, loading/error states.

Layout:
```
<SafeAreaView>
  <Header title="Habits" />
  <TabBar tabs={['Quality Habits', 'Ad-hoc']} />
  {activeTab === 'quality'
    ? <QualityHabitsTab qualityGroups={qualityHabits} onHabitPress={onHabitPress} />
    : <AdhocHabitsTab habits={adhocHabits} onHabitPress={onHabitPress} />
  }
  <HabitDetailBottomSheet
    visible={showBottomSheet}
    habit={selectedHabit}
    qualityGroup={selectedQualityGroup}
    selectedDays={selectedDays}
    onDaySelect={onDaySelect}
    onStart={onStartHabit}
    onClose={onCloseBottomSheet}
    isStarting={isStartingHabit}
  />
</SafeAreaView>
```

#### `styles.ts`
Uses `theme.colors.*` from `customLightTheme` + responsive utils (`hp`, `responsiveFontSize`).

---

### 2.5 New Reusable Components

All under: `project-ihsan/src/views/components/habits/`

#### `QualityHabitGroup.tsx`
Props:
```typescript
interface QualityHabitGroupProps {
  group: HabitsScreenQualityGroup
  onHabitPress: (habit: HabitsScreenHabitItem) => void
}
```
Renders:
- Quality header: quality name + level badge (pill)
- `FlatList` / `map` of `HabitListItem` for each habit in the group

#### `HabitListItem.tsx`
Props:
```typescript
interface HabitListItemProps {
  habit: HabitsScreenHabitItem
  onPress: () => void
}
```
Renders:
- Habit description text
- Status chip: green "Active" or muted "Available"
- Chevron / touch ripple

#### `HabitDetailBottomSheet.tsx`
Props:
```typescript
interface HabitDetailBottomSheetProps {
  visible: boolean
  habit: HabitsScreenHabitItem | null
  qualityGroup?: HabitsScreenQualityGroup | null
  selectedDays: 7 | 14 | 21 | 30
  onDaySelect: (days: 7 | 14 | 21 | 30) => void
  onStart: () => void
  onClose: () => void
  isStarting: boolean
}
```

Renders differently based on `habit.status`:

**ACTIVE state:**
- Habit text
- Quality badge (if quality habit)
- Status indicator: "This habit is currently active"
- Info message: "View your streak and daily progress on the Dashboard"
- Close button

**AVAILABLE state:**
- Habit text
- Quality badge (if quality habit)
- Day range selector: 4 options (7 / 14 / 21 / 30 days) — pill buttons
- "Start Habit" primary button (`isStarting` shows loader)
- Close button

Use the existing `Modal` + pan gesture pattern from `HabitStreamBottomSheet.tsx` as reference.

---

### 2.6 Navigation

**File:** `project-ihsan/src/navigations/MainNavigator.tsx`

1. Import `HabitsScreen` from `../views/screens/HabitsScreen`
2. Add `Habits` to `TabParamList`:
   ```typescript
   export type TabParamList = {
     Dashboard: undefined
     Catalog: undefined
     Social: undefined
     Habits: undefined   // new
     Profile: undefined
   }
   ```
3. Register tab in `BottomTabNavigator`:
   - Icon: `MaterialCommunityIcons` — `'calendar-check-outline'` (focused) / `'calendar-check-outline'` or `'heart-outline'` — choose from existing icon set
   - Label: `'Habits'`
   - Position: 4th tab (between Social and Profile), or as preferred

---

## 3. Implementation Order

1. **Backend** (do first, deploy once implemented)
   - [ ] 1.1 Repository method `listHabitStreamsForHabitsScreen`
   - [ ] 1.2 Service method `buildHabitStreamsForHabitsScreen` + new types
   - [ ] 1.3 Controller route `habit-screen`
   - [ ] 1.4 OpenAPI spec
   - [ ] 1.5 Deploy to `znjkyjbbycnajxepyjfi`

2. **Mobile models & API layer**
   - [ ] 2.1 `HabitsScreenModels.ts`
   - [ ] 2.2 `SupabaseEdgeFunctions.ts` method
   - [ ] 2.3 RTK Query `getHabitsScreen` + `startHabitStream` invalidation

3. **Mobile screen**
   - [ ] 2.4 All `HabitsScreen/` files
   - [ ] 2.5 Component: `QualityHabitGroup`
   - [ ] 2.5 Component: `HabitListItem`
   - [ ] 2.5 Component: `HabitDetailBottomSheet`
   - [ ] 2.6 Navigation (add 5th tab)

4. **QA / Polish**
   - Test quality habits grouping (check empty states)
   - Test ad-hoc tab (may be empty for most users initially)
   - Test "Start Habit" flow end-to-end (NOT_STARTED → ACTIVE → Dashboard)
   - Confirm cache invalidation: habit screen refreshes after starting a habit
   - TypeScript: `npx tsc --noEmit`
   - Lint + format

---

## 4. Open Questions / Assumptions

| # | Question | Assumption used in plan |
|---|----------|------------------------|
| 1 | Exact icon for Habits tab | `calendar-check-outline` from MaterialCommunityIcons — adjust after design review |
| 2 | Empty state copy | Generic "No habits unlocked yet" — adjust to match design copy |
| 3 | Should "Start Habit" on the Habits screen also support accountability partner sharing? | Not included — keep it minimal (start only); partner flow stays on Dashboard |
| 4 | What if a user has multiple ACTIVE habits? | Listed as separate items within their quality group, each tappable |
| 5 | Do quality-linked habits (gem_id IS NULL but quality_id IS NOT NULL) go to Quality tab or Ad-hoc tab? | Quality tab — grouped under their quality |
