# Habit Stream Status Simplification Plan

## Overview

Simplify habit stream statuses from 4 states (`ACTIVE`, `NOT_STARTED`, `COMPLETED`, `CANCELLED`) to 2 states (`ACTIVE`, `NOT_STARTED`). When a habit stream is completed or cancelled, it should transition back to `NOT_STARTED` status so it no longer appears in the "Available Habits" card on the dashboard.

Additionally, consolidate the three redundant functions (`endHabitStream`, `cancelHabitStream`, `completeHabitStream`) into a single generic `endHabitStream()` function at both the repository and service layers.

## Current Status Flow

```
NOT_STARTED → ACTIVE → COMPLETED (terminal state)
NOT_STARTED → ACTIVE → CANCELLED (terminal state)
```

## New Status Flow

```
NOT_STARTED ↔ ACTIVE → NOT_STARTED (on completion or cancellation)
```

## Impact Analysis

### Files to Modify

#### Backend (project-ihsan-backend)

| File | Change Required |
|------|-----------------|
| [`supabase/functions/shared/enums/habit-stream-status.enum.ts`](project-ihsan-backend/supabase/functions/shared/enums/habit-stream-status.enum.ts:1) | Remove `COMPLETED` and `CANCELLED` enum values |
| [`supabase/functions/repositories/habit-stream-progress-repository.ts`](project-ihsan-backend/supabase/functions/repositories/habit-stream-progress-repository.ts:14) | Remove `COMPLETED` and `CANCELLED` from enum; remove `completeHabitStream()` method |
| [`supabase/functions/repositories/habit-streams-repository.ts`](project-ihsan-backend/supabase/functions/repositories/habit-streams-repository.ts:1) | Remove `cancelHabitStream()` method; update `endHabitStream()` to set status to `NOT_STARTED` |
| [`supabase/functions/services/habit-stream-service.ts`](project-ihsan-backend/supabase/functions/services/habit-stream-service.ts:1) | Remove `cancelHabitStream()` method; update `endHabitStream()` documentation |
| [`supabase/functions/services/mappers/habit-stream-status-mapper.ts`](project-ihsan-backend/supabase/functions/services/mappers/habit-stream-status-mapper.ts:1) | Remove `COMPLETED` and `CANCELLED` mapping cases |
| [`supabase/functions/shared/models/dashboard-models.ts`](project-ihsan-backend/supabase/functions/shared/models/dashboard-models.ts:1) | No changes needed - `DailyProgress.status.COMPLETED` is for daily tracking, not stream status |

#### Database Migrations

| File | Change Required |
|------|-----------------|
| New migration | Update existing `COMPLETED` and `CANCELLED` records to `NOT_STARTED`; update status CHECK constraint |

#### Frontend (project-ihsan/src)

| File | Change Required |
|------|-----------------|
| [`models/HabitStreamModels.ts`](project-ihsan/src/models/HabitStreamModels.ts:1) | Remove `CancelHabitStreamRequest` and `CancelHabitStreamResponse` interfaces; update `EndHabitStreamResponse.status` type; update response type unions to remove `COMPLETED`/`FAILED` |
| [`models/DashboardParams.ts`](project-ihsan/src/models/DashboardParams.ts:1) | Update `HabitStreamData.status` type to remove `COMPLETED`/`FAILED` |
| [`store/api/supabaseEdgeFunctionsApi.ts`](project-ihsan/src/store/api/supabaseEdgeFunctionsApi.ts:1) | Remove `cancelHabitStream` mutation; update `endHabitStream` documentation |
| [`views/screens/DashBoardScreen/index.tsx`](project-ihsan/src/views/screens/DashBoardScreen/index.tsx:1) | Update any logic that checks for `COMPLETED` status on habit streams |
| [`views/screens/DashBoardScreen/DashboardScreen.view.tsx`](project-ihsan/src/views/screens/DashBoardScreen/DashboardScreen.view.tsx:1) | Update `isTodayCompleted` logic if it references habit stream `COMPLETED` status |

## Detailed Implementation Steps

### Step 1: Create Database Migration

- Migrate all existing `COMPLETED` and `CANCELLED` habit streams to `NOT_STARTED`
- Update the status CHECK constraint to only allow `ACTIVE` and `NOT_STARTED`

### Step 2: Update Enums

- Remove `COMPLETED` and `CANCELLED` from both the repository enum ([`habit-stream-progress-repository.ts`](project-ihsan-backend/supabase/functions/repositories/habit-stream-progress-repository.ts:14)) and the controller enum ([`habit-stream-status.enum.ts`](project-ihsan-backend/supabase/functions/shared/enums/habit-stream-status.enum.ts:8))

### Step 3: Update Status Mapper

- Remove `COMPLETED` and `CANCELLED` cases from both mapping functions in [`habit-stream-status-mapper.ts`](project-ihsan-backend/supabase/functions/services/mappers/habit-stream-status-mapper.ts:1)

### Step 4: Consolidate Repository Layer

- Remove `cancelHabitStream()` from [`habit-streams-repository.ts`](project-ihsan-backend/supabase/functions/repositories/habit-streams-repository.ts:1)
- Remove `completeHabitStream()` from [`habit-stream-progress-repository.ts`](project-ihsan-backend/supabase/functions/repositories/habit-stream-progress-repository.ts:1)
- Update `endHabitStream()` in [`habit-streams-repository.ts`](project-ihsan-backend/supabase/functions/repositories/habit-streams-repository.ts:1) to set status to `NOT_STARTED` (the only function for ending any habit stream)

### Step 5: Consolidate Service Layer

- Remove `cancelHabitStream()` from [`habit-stream-service.ts`](project-ihsan-backend/supabase/functions/services/habit-stream-service.ts:1)
- Update `endHabitStream()` to be the single function for ending any habit stream (both completion and cancellation scenarios)
- Update `IHabitStreamService` interface to reflect the removed method

### Step 6: Update Controller/Endpoint

- Find any controller endpoint that calls `cancelHabitStream` and update it to call `endHabitStream` instead
- Optionally add a reason parameter (completion vs cancellation) for analytics purposes

### Step 7: Update Frontend Models

- **File**: [`HabitStreamModels.ts`](project-ihsan/src/models/HabitStreamModels.ts:1)
  - Remove `CancelHabitStreamRequest` interface
  - Remove `CancelHabitStreamResponse` interface
  - Update `EndHabitStreamResponse.status` type from `'COMPLETED'` to `'NOT_STARTED'`
  - Update `CreateHabitStreamResponse.status` type union to remove `COMPLETED` and `FAILED`
  - Update `StartHabitStreamResponse.status` type union to remove `COMPLETED` and `FAILED`
  - Update JSDoc comments to reflect new behavior

### Step 8: Update Frontend Dashboard Models

- **File**: [`DashboardParams.ts`](project-ihsan/src/models/DashboardParams.ts:1)
  - Update `HabitStreamData.status` type union to remove `COMPLETED` and `FAILED`

### Step 9: Update RTK Query API

- **File**: [`supabaseEdgeFunctionsApi.ts`](project-ihsan/src/store/api/supabaseEdgeFunctionsApi.ts:1)
  - Remove `cancelHabitStream` mutation endpoint
  - Update `endHabitStream` mutation documentation

### Step 10: Review Frontend Dashboard Screens

- **File**: [`DashBoardScreen/index.tsx`](project-ihsan/src/views/screens/DashBoardScreen/index.tsx:1)
  - Review and update any logic that checks for habit stream `COMPLETED` status
  - Note: `DailyProgress.status === 'COMPLETED'` is for daily tracking and should remain unchanged

- **File**: [`DashboardScreen.view.tsx`](project-ihsan/src/views/screens/DashBoardScreen/DashboardScreen.view.tsx:1)
  - Review `isTodayCompleted` logic - the `backendStatus === 'COMPLETED'` check refers to daily progress status, not habit stream status, so likely no changes needed

## Testing Considerations

1. **Unit Tests**: Update any tests that expect `COMPLETED` or `CANCELLED` status
2. **Integration Tests**: Test the full flow of starting → ending → seeing in available habits
3. **Edge Cases**:
   - What happens if user tries to end an already NOT_STARTED stream?
   - Verify that ending a stream removes it from `active` array and adds it to `unlocked` array

## Migration Strategy

1. **Deploy backend changes first** (backward compatible - old clients will still work)
2. **Run database migration** to update existing records
3. **Deploy frontend changes** (if any)

## Rollback Plan

If issues arise:
1. Revert database migration to restore `COMPLETED`/`CANCELLED` statuses
2. Deploy previous version of backend code

## Frontend Impact

The frontend currently displays habit streams based on `active` and `unlocked` arrays from the dashboard API. Since `COMPLETED` and `CANCELLED` streams will now become `NOT_STARTED`, they will appear in the `unlocked` array again, which is the desired behavior.

Several frontend files need updates to remove references to `COMPLETED` and `CANCELLED` habit stream statuses, but the core dashboard display logic will work automatically with the new status flow.

## Summary

This change simplifies the habit stream lifecycle by:
- Removing terminal states (`COMPLETED`, `CANCELLED`)
- Making habit streams recyclable back to `NOT_STARTED` status
- Consolidating three redundant functions into one generic `endHabitStream()`
- Ensuring completed/cancelled streams reappear in "Available Habits" card
- Maintaining backward compatibility with existing dashboard display logic
