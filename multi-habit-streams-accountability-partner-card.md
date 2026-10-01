---
name: Multi Habit Streams Accountability Partner Card
overview: Update the PartnerDetailCard component to display the partner's active quality (with progress) and all active habit streams in a unified card layout, following the new design specification.
todos:
  - id: analyze-data-structure
    content: "Verify API response structure supports multiple active habit streams"
    status: completed
  - id: update-types
    content: "Update Partner interface and component props if needed for new data structure"
    status: pending
  - id: quality-section
    content: "Create separate Quality section with name, progress bar, and percentage completion"
    status: pending
  - id: habits-section
    content: "Create Habits section with heading, count, and habit list with week dots"
    status: pending
  - id: habit-list-item
    content: "Implement individual habit list item with description, chip, and week dots"
    status: pending
  - id: update-styles
    content: "Add new styles matching the design system and React Native Paper components"
    status: pending
  - id: update-skeleton
    content: "Update PartnerCardSkeleton to match new card layout"
    status: pending
  - id: conditional-rendering
    content: "Implement conditional rendering: hide card if no active quality AND no active habits"
    status: pending
isProject: false
---

# Multi Habit Streams Accountability Partner Card

**Deliverable file:** [plans/multi-habit-streams-accountability-partner-card.md](plans/multi-habit-streams-accountability-partner-card.md)

---

## Problem (current behavior)

The current [`PartnerDetailCard`](project-ihsan/src/views/components/PartnerDetailCard.tsx) component:

1. **Shows only ONE habit stream** - renders only `partnerDetails.activeHabitStreams[0]` (line 359)
2. **Mutually exclusive display** - shows either habit stream OR quality progress, not both together
3. **Old week tracker UI** - displays day labels (Mon, Tue, etc.) below circles, which the new design removes
4. **No habit count** - doesn't show "X in progress" count for multiple habits

The API already returns multiple active habit streams via `activeHabitStreams: ActiveHabitStreamInfo[]`, but the UI only displays the first one.

---

## Target behavior (from design)

The new card layout has these sections:

### 1. Card Header (unchanged)
- Avatar + Name + Connected Since + Menu button

### 2. Stats Row (unchanged)
- Qualities Developed | Gems Completed | Habit Days Completed

### 3. Quality Section (NEW - Surface tonal card)
- **Left:** Quality name (e.g., "Kindness") in `md-title-large` with weight 500
- **Right:** Percentage (e.g., "7%") in primary color + "completed" label
- **Below:** Progress bar (4px height, primary color fill)
- **Styling:** `paper-surface-primary` background (`--md-primary-container` / `#FFF1ED`)

### 4. Habits Section (NEW - Outlined card)
- **Header:** "Habits" label (uppercase, letter-spacing) + "3 in progress" count
- **Habit List Items** (each separated by divider):
  - **Left:** Habit description text (up to 3 lines, ellipsize)
  - **Right:** Week dots row (7 dots inline, no day labels)
    - Completed: teal fill (`--md-teal` / `#14B391`)
    - Missed: grey fill (`--md-outline-variant` / `#D9CDCD`)
    - Today: teal fill with outline (`--md-teal-container` border)
    - Future: light grey (`--md-outline` / `#EFEAEA`)
  - **Below description:** Small chip showing quality name (primary container style) or "Adhoc" style

### 5. Action Buttons (unchanged)
- Poke | Celebrate

### Conditional Display Rules
- If **no active quality** AND **no active habits**: Show empty state (existing empty state UI)
- If **active quality exists**: Show quality section
- If **active habits exist**: Show habits section with all active habits
- If **both exist**: Show both sections

---

## Implementation Plan

### Step 1: Verify Data Structure (No changes needed)

The API response type [`GetAccountabilityPartnerDetailsResponse`](project-ihsan/src/models/AccountabilityPartnerModels.ts:157) already has:

```typescript
interface GetAccountabilityPartnerDetailsResponse {
  currentQuality: CurrentQualityDetails | null;
  activeHabitStreams: ActiveHabitStreamInfo[];
  // ... other fields
}
```

And [`ActiveHabitStreamInfo`](project-ihsan/src/models/DashboardParams.ts:70) has:

```typescript
interface ActiveHabitStreamInfo {
  id: number;
  text: string;
  quality: string;
  status: string;
  startDate: string;
  streak: HabitStreamStreak;
  dailyProgress: DailyProgress[];
}
```

**No type changes needed** - the data structure already supports multiple streams.

### Step 2: Refactor PartnerDetailCard Component

**File:** [`project-ihsan/src/views/components/PartnerDetailCard.tsx`](project-ihsan/src/views/components/PartnerDetailCard.tsx)

#### 2a. Replace the current progress section (lines 299-467)

**Current structure:**
```
{showCelebration ? (
  celebration section
) : (
  progressSection {
    if (activeHabitStreams[0]) → show ONE habit
    else if (currentQuality) → show quality
    else → empty state
  }
)}
```

**New structure:**
```
{showCelebration ? (
  celebration section (keep existing)
) : isLoadingDetails ? (
  loading indicator
) : hasError ? (
  error state
) : hasNoData || (no quality AND no habits) ? (
  empty state
) : (
  <>
    {currentQuality && <QualitySection />}
    {activeHabitStreams.length > 0 && <HabitsSection />}
  </>
)}
```

#### 2b. Create QualitySection component (inline or extracted)

```tsx
const QualitySection: React.FC<{ quality: CurrentQualityDetails }> = ({ quality }) => (
  <View style={styles.qualitySection}>
    <View style={styles.qualityHeaderRow}>
      <ResponsiveText baseFontSize={22} style={styles.qualityName}>
        {quality.name}
      </ResponsiveText>
      <View style={styles.qualityProgressInfo}>
        <ResponsiveText baseFontSize={14} style={styles.qualityPercentage}>
          {quality.completionPercentage}%
        </ResponsiveText>
        <ResponsiveText baseFontSize={11} style={styles.qualityCompletedLabel}>
          completed
        </ResponsiveText>
      </View>
    </View>
    <Progress.Bar
      progress={quality.completionPercentage / 100}
      width={null}
      height={4}
      color={customTheme.colors.primary}
      unfilledColor={customTheme.colors.primaryContainer}
      borderWidth={0}
      borderRadius={2}
    />
  </View>
);
```

#### 2c. Create HabitsSection component

```tsx
const HabitsSection: React.FC<{ habits: ActiveHabitStreamInfo[] }> = ({ habits }) => {
  const inProgressCount = habits.filter(h => h.status === 'ACTIVE').length;
  
  return (
    <View style={styles.habitsCard}>
      <View style={styles.habitsCardContent}>
        {/* Header */}
        <View style={styles.habitsHeaderRow}>
          <ResponsiveText baseFontSize={12} style={styles.habitsHeading}>
            Habits
          </ResponsiveText>
          <ResponsiveText baseFontSize={11} style={styles.habitsCount}>
            {inProgressCount} in progress
          </ResponsiveText>
        </View>

        {/* Habit List */}
        {habits.map((habit, index) => (
          <React.Fragment key={habit.id}>
            {index > 0 && <View style={styles.habitDivider} />}
            <HabitListItem habit={habit} />
          </React.Fragment>
        ))}
      </View>
    </View>
  );
};
```

#### 2d. Create HabitListItem component

```tsx
const HabitListItem: React.FC<{ habit: ActiveHabitStreamInfo }> = ({ habit }) => {
  // Get last 7 days of progress (or pad with future days if less than 7)
  const weekDays = habit.dailyProgress.slice(0, 7);

  return (
    <View style={styles.habitListItem}>
      <View style={styles.habitListItemContent}>
        <ResponsiveText 
          baseFontSize={14} 
          style={styles.habitDescription} 
          numberOfLines={3}
          ellipsizeMode="tail"
        >
          {habit.text}
        </ResponsiveText>
        <View style={styles.habitChip}>
          <ResponsiveText baseFontSize={12} style={styles.habitChipText}>
            {habit.quality || 'Adhoc'}
          </ResponsiveText>
        </View>
      </View>
      <View style={styles.weekDotsContainer}>
        {weekDays.map((day, idx) => {
          const dotStyle = getWeekDotStyle(day);
          return <View key={`${day.date}-${idx}`} style={[styles.weekDot, dotStyle]} />;
        })}
      </View>
    </View>
  );
};
```

#### 2e. Week dot style helper

```tsx
const getWeekDotStyle = (day: DailyProgress): ViewStyle => {
  if (day.status === 'COMPLETED') return styles.weekDotDone;
  if (day.status === 'MISSED' || day.status === 'SKIPPED') return styles.weekDotMissed;
  if (day.isToday) return styles.weekDotToday;
  return styles.weekDotFuture;
};
```

### Step 3: Add New Styles

Add to the existing `StyleSheet.create` in [`PartnerDetailCard.tsx`](project-ihsan/src/views/components/PartnerDetailCard.tsx:512):

```typescript
// Quality Section Styles
qualitySection: {
  backgroundColor: customTheme.colors.primaryContainer, // #FFF1ED
  borderRadius: wp(3),
  padding: responsiveSpacing(14),
  marginBottom: responsiveSpacing(12),
},
qualityHeaderRow: {
  flexDirection: 'row',
  justifyContent: 'space-between',
  alignItems: 'baseline',
  marginBottom: responsiveSpacing(8),
},
qualityName: {
  color: customTheme.colors.darkUiElement,
  fontFamily: 'Poppins-Regular',
  fontWeight: '500',
},
qualityProgressInfo: {
  flexDirection: 'row',
  alignItems: 'baseline',
  gap: wp(1),
},
qualityPercentage: {
  color: customTheme.colors.primary,
  fontFamily: 'Poppins-Bold',
  fontSize: responsiveFontSize(14),
},
qualityCompletedLabel: {
  color: '#79747E',
  fontFamily: 'Poppins-Medium',
  fontSize: responsiveFontSize(11),
},

// Habits Section Styles
habitsCard: {
  backgroundColor: customTheme.colors.background,
  borderRadius: wp(3),
  borderWidth: 1,
  borderColor: customTheme.colors.outline,
  overflow: 'hidden',
},
habitsCardContent: {
  padding: responsiveSpacing(16),
},
habitsHeaderRow: {
  flexDirection: 'row',
  justifyContent: 'space-between',
  alignItems: 'center',
  marginBottom: responsiveSpacing(10),
},
habitsHeading: {
  color: customTheme.colors.darkUiElement,
  fontFamily: 'Poppins-Medium',
  fontSize: responsiveFontSize(12),
  letterSpacing: 1.1,
  textTransform: 'uppercase',
},
habitsCount: {
  color: '#79747E',
  fontFamily: 'Poppins-Medium',
  fontSize: responsiveFontSize(11),
  letterSpacing: 1,
  textTransform: 'uppercase',
},
habitDivider: {
  height: 1,
  backgroundColor: customTheme.colors.outline,
  marginVertical: responsiveSpacing(4),
},

// Habit List Item Styles
habitListItem: {
  flexDirection: 'row',
  justifyContent: 'space-between',
  alignItems: 'flex-start',
  paddingVertical: responsiveSpacing(10),
  gap: wp(3),
},
habitListItemContent: {
  flex: 1,
  gap: responsiveSpacing(6),
},
habitDescription: {
  color: customTheme.colors.darkUiElement,
  fontFamily: 'Poppins-Regular',
  fontSize: responsiveFontSize(14),
  lineHeight: responsiveSpacing(20),
},
habitChip: {
  alignSelf: 'flex-start',
  paddingHorizontal: wp(2.5),
  paddingVertical: responsiveSpacing(3),
  borderRadius: wp(1.5),
  borderWidth: 1,
  borderColor: customTheme.colors.outline,
  backgroundColor: customTheme.colors.primaryContainer,
},
habitChipText: {
  color: customTheme.colors.primary,
  fontFamily: 'Poppins-Medium',
  fontSize: responsiveFontSize(12),
},

// Week Dots Styles
weekDotsContainer: {
  flexDirection: 'row',
  alignItems: 'center',
  gap: wp(0.8),
  alignSelf: 'flex-start',
  marginTop: responsiveSpacing(4),
},
weekDot: {
  width: wp(3),
  height: wp(3),
  borderRadius: wp(1.5),
},
weekDotDone: {
  backgroundColor: customTheme.colors.viewMode, // #14B391 teal
},
weekDotMissed: {
  backgroundColor: customTheme.colors.circleMissed, // #CAC4D0 grey
},
weekDotToday: {
  backgroundColor: customTheme.colors.viewMode,
  borderWidth: 2,
  borderColor: customTheme.colors.primaryContainer,
},
weekDotFuture: {
  backgroundColor: '#EFEAEA',
},
```

### Step 4: Update Skeleton Loader

**File:** [`project-ihsan/src/views/components/PartnerCardSkeleton.tsx`](project-ihsan/src/views/components/PartnerCardSkeleton.tsx)

Update `PartnerDetailCardSkeleton` to include:
1. Quality section skeleton (tonal background with shimmer)
2. Habits section skeleton (outlined card with shimmer)
3. Multiple habit item skeletons

### Step 5: Conditional Rendering Logic

The card should be hidden entirely if:
- `partnerDetails.currentQuality` is null/undefined AND
- `partnerDetails.activeHabitStreams` is empty or undefined

This check should be done in the parent component ([`SocialScreen.view.tsx`](project-ihsan/src/views/screens/SocialScreen/SocialScreen.view.tsx)) or passed as a prop.

**Implementation in SocialScreen.view.tsx:**
```tsx
const shouldShowPartnerCard = useMemo(() => {
  if (!partnerDetailsData) return true; // Show skeleton while loading
  const hasQuality = partnerDetailsData.currentQuality;
  const hasHabits = partnerDetailsData.activeHabitStreams?.length > 0;
  return hasQuality || hasHabits;
}, [partnerDetailsData]);
```

---

## Design System Mapping

| Design Token | React Native Theme |
|--------------|-------------------|
| `--md-primary` | `customTheme.colors.primary` (#F97461 in design, #FE6F5E in app) |
| `--md-primary-container` | `customTheme.colors.primaryContainer` |
| `--md-on-surface` | `customTheme.colors.darkUiElement` (#3E4B6E) |
| `--md-on-surface-variant` | `#79747E` (add to theme or use directly) |
| `--md-teal` | `customTheme.colors.viewMode` (#26A69A) |
| `--md-teal-container` | Add new color or use `#D9F2E9` directly |
| `--md-outline` | `customTheme.colors.outline` (#79747E) |
| `--md-outline-variant` | `customTheme.colors.circleMissed` (#CAC4D0) |
| `--md-elev1-bg` | `#FFFAF9` (use directly or add to theme) |

---

## React Native Paper Components to Use

1. **`ProgressBar`** - For quality progress bar (already using `react-native-progress`)
2. **`Card`** (mode=elevated) - For main card container
3. **`Card`** (mode=outlined) - For habits section
4. **`Surface`** (tonal) - For quality section background
5. **`Chip`** (assist/filter) - For habit quality chips
6. **`Divider`** - For separators between habits
7. **`Text`** variants - For typography (or keep ResponsiveText with proper styles)

**Note:** The existing codebase uses `ResponsiveText` wrapper extensively. We'll continue using it for text elements to maintain consistency with the existing codebase patterns.

---

## Files to Modify

1. **`project-ihsan/src/views/components/PartnerDetailCard.tsx`**
   - Refactor progress section to show quality + multiple habits
   - Add new components: QualitySection, HabitsSection, HabitListItem
   - Add new styles

2. **`project-ihsan/src/views/components/PartnerCardSkeleton.tsx`**
   - Update skeleton to match new layout

3. **`project-ihsan/src/views/screens/SocialScreen/SocialScreen.view.tsx`** (optional)
   - Add conditional rendering to hide card if no quality AND no habits

4. **`project-ihsan/src/views/themes/customLightTheme.ts`** (optional)
   - Add `--md-teal-container` color if needed for today's dot outline

---

## Testing Focus

1. **Quality only** (no active habits): Shows quality section with progress
2. **Habits only** (no active quality): Shows habits section with all active habits
3. **Both quality and habits**: Shows both sections
4. **Neither quality nor habits**: Shows empty state
5. **Multiple habits** (2+): All habits displayed with proper dividers
6. **Habit description overflow**: Long text ellipsized at 3 lines
7. **Week dots**: Correct colors for completed/missed/today/future states
8. **Loading state**: Skeleton matches new layout
9. **Error state**: Shows error message appropriately

---

## Implementation Order

1. Add new styles to StyleSheet
2. Create QualitySection component
3. Create HabitsSection component  
4. Create HabitListItem component
5. Create week dot helper/styles
6. Refactor main render logic
7. Update skeleton loader
8. Add conditional rendering in parent (if needed)
9. Test all scenarios
