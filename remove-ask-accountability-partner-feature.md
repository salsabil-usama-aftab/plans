# Remove Ask Accountability Partner Feature - Implementation Plan

## Overview
This plan outlines the removal of the "Ask Accountability Partner" feature from the Project Ihsan mobile application. This includes:
1. Removing "Ask Accountability Partner" from the GemScreen menu
2. Removing "Answer Questions" button from the Social screen
3. Cleaning up all related components, hooks, state management, and unused code

## Analysis Summary

### Feature Locations Identified

#### 1. **GemScreen Menu - "Ask Accountability Partner"**
- **Location**: [`project-ihsan/src/views/screens/GemScreen/GemScreen.view.tsx`](project-ihsan/src/views/screens/GemScreen/GemScreen.view.tsx:462-479)
- **Menu Item**: Lines 462-479 - Tooltip menu item with "Ask Accountability Partner" text
- **State Management**: [`project-ihsan/src/views/screens/GemScreen/index.tsx`](project-ihsan/src/views/screens/GemScreen/index.tsx:161)
  - Line 161: `showAskPartnerSheet` state
  - Lines 828-835: `handleOpenAskPartnerSheet` handler
  - Lines 833-835: `handleCloseAskPartnerSheet` handler
  - Lines 837-844: `handleSendQuestion` handler
- **Props Passed**: Lines 1134-1136 in GemScreen/index.tsx
- **Bottom Sheet Usage**: Lines 550-554 in GemScreen.view.tsx

#### 2. **Social Screen - "Answer Questions" Button**
- **Location**: [`project-ihsan/src/views/components/PartnerDetailCard.tsx`](project-ihsan/src/views/components/PartnerDetailCard.tsx:489-496)
- **Button**: Lines 489-496 - "Answer Questions" button in partner detail card
- **Handler**: [`project-ihsan/src/views/screens/SocialScreen/index.tsx`](project-ihsan/src/views/screens/SocialScreen/index.tsx:702-704)
  - Line 702-704: `onAnswerQuestions` callback
- **State**: Line 244 in SocialScreen/index.tsx: `showQuestionsSheet` state
- **Props Chain**:
  - SocialScreen/index.tsx → SocialScreen.view.tsx (line 1051)
  - SocialScreen.view.tsx → PartnerDetailCard (lines 456, 476)

### Components to Remove

#### Primary Components
1. **[`AskPartnerQuestionBottomSheet.tsx`](project-ihsan/src/views/components/AskPartnerQuestionBottomSheet.tsx)**
   - Full component file (483 lines)
   - Used in GemScreen for asking partners questions
   - Has partner selection, question input, and send functionality

2. **[`PartnerQuestionsBottomSheet.tsx`](project-ihsan/src/views/components/PartnerQuestionsBottomSheet.tsx)**
   - Full component file (431 lines)
   - Used in SocialScreen for answering partner questions
   - Has question carousel, answer input, and navigation

### Files to Modify

#### Frontend (project-ihsan)

1. **GemScreen Files**
   - [`src/views/screens/GemScreen/index.tsx`](project-ihsan/src/views/screens/GemScreen/index.tsx)
     - Remove: `showAskPartnerSheet` state (line 161)
     - Remove: `mappedPartners` state (line 162) - only used for ask partner feature
     - Remove: Partner fetching logic (lines 73-134)
     - Remove: `handleOpenAskPartnerSheet` (lines 828-831)
     - Remove: `handleCloseAskPartnerSheet` (lines 833-835)
     - Remove: `handleSendQuestion` (lines 837-844)
     - Remove: Props passed to view (lines 1134-1136)
     - Remove: `showAskPartnerSheet` prop (line 1141)
   
   - [`src/views/screens/GemScreen/GemScreen.view.tsx`](project-ihsan/src/views/screens/GemScreen/GemScreen.view.tsx)
     - Remove: Import of `AskPartnerQuestionBottomSheet` (line 14)
     - Remove: Menu item "Ask Accountability Partner" (lines 462-479)
     - Remove: Bottom sheet component (lines 549-554)
     - Remove: Props from interface (lines 63-64, 72-74)
     - Remove: `showAskPartnerSheet` prop (line 80)
     - Remove: `partners` prop (line 64)
   
   - [`src/views/screens/GemScreen/types.ts`](project-ihsan/src/views/screens/GemScreen/types.ts)
     - Remove: `partners` prop (lines 202-206)
     - Remove: `onOpenAskPartnerSheet` prop (line 214)
     - Remove: `onCloseAskPartnerSheet` prop (line 215)
     - Remove: `onSendQuestion` prop (line 216)

2. **SocialScreen Files**
   - [`src/views/screens/SocialScreen/index.tsx`](project-ihsan/src/views/screens/SocialScreen/index.tsx)
     - Remove: `showQuestionsSheet` state (line 244)
     - Remove: `mockQuestions` data (lines 421-438)
     - Remove: `onAnswerQuestions` callback (lines 702-704)
     - Remove: Question sheet handlers (lines 706-730)
     - Remove: `PartnerQuestionsBottomSheet` component (lines 1084-1095)
     - Remove: Props passed to view (line 1051)
   
   - [`src/views/screens/SocialScreen/SocialScreen.view.tsx`](project-ihsan/src/views/screens/SocialScreen/SocialScreen.view.tsx)
     - Remove: `onAnswerQuestions` from Props interface (line 59)
     - Remove: `onAnswerQuestions` from destructuring (line 209)
     - Remove: Props passed to PartnerDetailCard (lines 456, 476)

3. **PartnerDetailCard Component**
   - [`src/views/components/PartnerDetailCard.tsx`](project-ihsan/src/views/components/PartnerDetailCard.tsx)
     - Remove: `onAnswerQuestions` from Props interface (line 40)
     - Remove: `onAnswerQuestions` from destructuring (line 60)
     - Remove: "Answer Questions" button (lines 489-496)

4. **Component Files to Delete**
   - [`src/views/components/AskPartnerQuestionBottomSheet.tsx`](project-ihsan/src/views/components/AskPartnerQuestionBottomSheet.tsx) - DELETE ENTIRE FILE
   - [`src/views/components/PartnerQuestionsBottomSheet.tsx`](project-ihsan/src/views/components/PartnerQuestionsBottomSheet.tsx) - DELETE ENTIRE FILE

### Backend Considerations

**Note**: The backend accountability partner functionality should be **PRESERVED**. The backend handles:
- Partner invitations
- Partner connections (accept/decline)
- Partner interactions (poke, celebrate)
- Partner details retrieval
- Partner listing

**What we're removing is ONLY**:
- The "Ask Question" feature (frontend only - no backend implementation found)
- The "Answer Questions" feature (frontend only - no backend implementation found)

These features appear to be UI mockups that were never fully implemented with backend support.

### Files NOT to Modify

The following accountability partner features should remain intact:
- ✅ Partner invitation system
- ✅ Partner connection management (accept/decline)
- ✅ Partner detail viewing
- ✅ Poke and Celebrate features
- ✅ Partner list display
- ✅ All backend services and repositories
- ✅ API endpoints in SupabaseEdgeFunctions
- ✅ RTK Query hooks for partner management

## Implementation Steps

### Phase 1: Remove Bottom Sheet Components
1. Delete [`src/views/components/AskPartnerQuestionBottomSheet.tsx`](project-ihsan/src/views/components/AskPartnerQuestionBottomSheet.tsx)
2. Delete [`src/views/components/PartnerQuestionsBottomSheet.tsx`](project-ihsan/src/views/components/PartnerQuestionsBottomSheet.tsx)

### Phase 2: Clean Up GemScreen
1. Remove "Ask Accountability Partner" menu item from GemScreen.view.tsx
2. Remove AskPartnerQuestionBottomSheet import and usage
3. Remove all related state, handlers, and props from GemScreen/index.tsx
4. Remove partner fetching logic (only used for ask partner feature)
5. Update GemScreen types to remove related props

### Phase 3: Clean Up SocialScreen
1. Remove "Answer Questions" button from PartnerDetailCard
2. Remove PartnerQuestionsBottomSheet usage from SocialScreen
3. Remove related state and handlers from SocialScreen/index.tsx
4. Remove mock questions data
5. Update prop interfaces

### Phase 4: Testing & Verification
1. Verify GemScreen menu only shows "Go to Dashboard"
2. Verify SocialScreen partner detail card only shows Poke and Celebrate buttons
3. Test that partner invitation/connection features still work
4. Test poke and celebrate features
5. Verify no console errors or warnings
6. Check that no unused imports remain

## Risk Assessment

### Low Risk
- These features appear to be incomplete/mockup implementations
- No backend API endpoints found for question/answer functionality
- Removal is isolated to specific UI components
- Core accountability partner features remain intact

## Files Summary

### Files to Delete (2)
1. `project-ihsan/src/views/components/AskPartnerQuestionBottomSheet.tsx`
2. `project-ihsan/src/views/components/PartnerQuestionsBottomSheet.tsx`

### Files to Modify (7)
1. `project-ihsan/src/views/screens/GemScreen/index.tsx`
2. `project-ihsan/src/views/screens/GemScreen/GemScreen.view.tsx`
3. `project-ihsan/src/views/screens/GemScreen/types.ts`
4. `project-ihsan/src/views/screens/SocialScreen/index.tsx`
5. `project-ihsan/src/views/screens/SocialScreen/SocialScreen.view.tsx`
6. `project-ihsan/src/views/components/PartnerDetailCard.tsx`
7. `project-ihsan/src/views/screens/SocialScreen/types.ts` (if onAnswerQuestions is defined there)

### Backend Files - NO CHANGES NEEDED
- All backend accountability partner services remain unchanged
- All API endpoints remain functional
- Database schema unchanged

## Estimated Impact
- **Lines of Code Removed**: ~1000+ lines
- **Components Removed**: 2 major bottom sheet components
- **Features Affected**: Only "Ask Question" and "Answer Questions" UI features
- **Core Features Preserved**: All other accountability partner functionality

## Next Steps
After approval of this plan:
1. Create a feature branch for the changes
2. Implement changes in phases as outlined
3. Test thoroughly on both iOS and Android
4. Submit for code review
5. Merge to main branch

---

**Plan Created**: 2026-04-25
**Status**: Ready for Review
