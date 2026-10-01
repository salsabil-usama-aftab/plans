# Plan: Dynamic Options for the Arrange Exercise

## Problem

`question_arrange` exercises take their draggable options from a static `content.values: string[]` list that the admin types by hand. Reference chips can already personalise the arrange **description**, but the **options** cannot. We want the admin to say "the options for this arrange come from what the user entered in Exercise 3 (a `question_chips`) of this gem / an earlier gem in the same quality".

This is end to end:

1. **Admin portal**: choose and save the dynamic option sources.
2. **Backend**: hydrate sources whose answers are already submitted (earlier gems).
3. **Mobile app**: hydrate sources in the **current gem** from local answers, because answers are only submitted when the gem is completed.

## Decisions

| Question | Answer |
|---|---|
| Allowed source types | **`question_chips` only** |
| Static + dynamic | **Either/or**: an arrange is fully static (`values`) or fully dynamic (one or more chip sources). In dynamic mode the static list is hidden and not saved. |
| Multiple sources | Allowed. Values are concatenated in source order and de-duplicated (trimmed, case-insensitive; first spelling wins). |
| Empty result | Not expected: a chips exercise always has at least one answer. Handled defensively only (see Edge Cases). |
| Cap on options | **None** |
| Scope of sources | Same as reference chips: earlier exercises in the current gem, plus any exercise in earlier gems of the same quality (reuses `searchGemsForReference`). |

---

## Current State (for context)

- **Admin**: `QuestionArrangeDialog.tsx` saves `{ title, text, values, grid, references }`. `useReferenceExercises` → `searchReferenceExercisesAction` → `gemsRepository.searchGemsForReference` already returns the eligible exercises (earlier gems + earlier exercises in the current gem).
- **Backend**: `content-management` → `OnboardingService.getGem(gemId, userId)` → `hydrateExercises()` hydrates only `content.text` via `HydrationService.hydrateExerciseText()`. Responses come from `UserSubmissionsService.getUserResponsesForHydration()`.
- **Mobile**: `GemScreen/index.tsx` maps `content.values` → `ArrangeQuestion.triggers` once, at gem load. `HydrationUtility.hydrate()` fills same-gem `<reference-chip>` text from local `answers`. Local chips answers are `string[]`, and `buildResponse` submits them as `{ chips: [...] }`.

### ⚠️ Chips response-key inconsistency (must handle)

The stored key for chips answers is not consistent:

| Location | Key |
|---|---|
| Mobile `buildResponse` (GemScreen) | `chips` |
| `user-submission-validator.ts` schema | `values` (required, `additionalProperties: false`) |
| `flow-rules-hydrator.ts` | `triggers` |
| `hydration-utility.ts` | `values` or `triggers` |

**Step 0**: query real rows (`select response from user_responses ur join exercises e on e.id = ur.exercise_id where e.type = 'question_chips' limit 20`) on local and staging to confirm what is actually persisted. Whatever the result, the new extractor must accept **`chips` | `values` | `triggers`**, and handle both wrapped (`{ type, response }`) and unwrapped rows. Fixing the validator/hydrator mismatch is a separate follow-up and is out of scope here. Note it in the PR.

---

## Data Model

No migration is needed: everything lives in the `exercises.content` JSONB.

```ts
// question_arrange content
interface QuestionArrangeContent {
  title: string
  text: string                       // HTML with <reference-chip> tags (unchanged)
  references?: ContentReference[]    // text reference chips (unchanged)
  grid: { columns: {id,title}[]; rows: {id,title}[] }

  optionsMode?: 'static' | 'dynamic' // absent ⇒ 'static' (backward compatible)
  values: string[]                   // static options; [] when optionsMode = 'dynamic'
  optionSources?: ArrangeOptionSource[] // required (length ≥ 1) when 'dynamic'
}

interface ArrangeOptionSource {
  gemId: number
  exerciseId: number
  exerciseType: 'question_chips'
  answerPath: 'response.chips'       // kept for symmetry with ContentReference
  exerciseName: string               // "Gem 2: Title - Exercise 3"
  displayText: string                // admin UI label

  // ── Added by the backend at read time, never saved ──
  resolvedValues?: string[]          // chips from the submitted answer
  hydrated?: boolean                 // true when a submitted answer was found
}
```

Why keep the values per source instead of writing a merged `values` list on the server: the app needs to know **which** sources it still has to resolve locally, and in what order. Keeping them per source lets the app merge server and local values deterministically, and lets local answers override stale server values when a user retakes a gem.

---

## 1. Admin Portal (`project-ihsan-admin/`)

### 1.1 Types
- `components/features/gems/CustomExerciseDialog/types.ts`: add `optionsMode` and `optionSources` to `QuestionArrangeContent`, and export `ArrangeOptionSource`.
- `types/exercises.ts`: update the shared exercise content type if arrange content is typed there.

### 1.2 `QuestionArrangeDialog.tsx`
- Add a segmented control / radio above "Values to Arrange": **Static list** | **From previous answers**.
- **Static**: current UI, unchanged.
- **Dynamic**: replace the values editor with an **Option Sources** section:
  - "Add source" opens a picker of eligible exercises from `useReferenceExercises({ qualityId, currentGemId: gemId, currentExerciseOrder })`, **filtered to `type === 'question_chips'`**. Reuse the list and search UI from the ReferenceTiptapEditor picker (`useFilteredExercises`). Extract a small shared `ExercisePickerList` component if the existing one is too coupled to Tiptap.
  - Each selected source shows as a removable row: "Gem N · Exercise M · {title}" with a "Current gem" badge when `gemId === currentGemId`.
  - Block duplicate sources (same `exerciseId`).
  - Empty state: "No chips exercises before this one in this quality."
- Loading an existing exercise: `optionsMode = content.optionsMode ?? 'static'` and `optionSources = content.optionSources ?? []`.
- `handleSave`:
  - static → `{ ..., optionsMode: 'static', values: validValues }` (omit `optionSources`)
  - dynamic → `{ ..., optionsMode: 'dynamic', values: [], optionSources }`. Strip `resolvedValues` / `hydrated` defensively.
- `isFormValid`: static needs ≥ 1 non-empty value (as today). Dynamic needs ≥ 1 source.
- Update the helper text: "Users will arrange the chips they entered in the selected exercise(s)."

### 1.3 Ordering guard
`currentExerciseOrder` is undefined for a **new** exercise. Confirm how `searchGemsForReference` treats that for the current gem: it should include all existing exercises, since the new one is appended last. Also add a non-blocking warning in `GemExercises.tsx`: when exercises are reordered or deleted, flag any arrange exercise whose same-gem `optionSources` now point to a later or missing exercise. Check this with the same "source order < arrange order" rule.

### 1.4 Other admin touch points
- `lib/services/pdf/quality-pdf-generator.ts`: for dynamic arrange, print "Options: user's answers from Gem N · Exercise M" instead of the empty values list.
- `FlowRuleEditor/AnswerFieldRenderer.tsx`: arrange flow conditions don't depend on `values`, so no change is needed. Verify only.

---

## 2. Backend (`project-ihsan-backend/`)

### 2.1 Pure utility: `services/utilities/arrange-options-utility.ts` (new)
```ts
export interface ArrangeOptionSource { ... }   // mirrors admin type

// Normalises a stored chips response (wrapped or not; chips|values|triggers) → string[]
export function extractChipValues(stored: unknown): string[]

// Mutates nothing; returns sources annotated with resolvedValues/hydrated
export function resolveArrangeOptionSources(
  sources: ArrangeOptionSource[],
  responseMap: Map<number, unknown>,
): ArrangeOptionSource[]
```
- `extractChipValues`: unwrap `{ type, response }`, then read the first array among `chips`, `values`, `triggers`. Map items to strings (`string`, or `.text` / `.value` for objects), trim, and drop empty ones.
- No I/O, same as `hydration-utility.ts`.

### 2.2 `HydrationService`
Add:
```ts
hydrateArrangeOptionSources(
  sources: ArrangeOptionSource[],
  userId: string,
): Promise<ArrangeOptionSource[]>
```
It fetches via `userSubmissionsService.getUserResponsesForHydration(userId, ids)` and delegates to the utility. It degrades gracefully: on error it returns the sources with `hydrated: false`.

### 2.3 `OnboardingService.hydrateExercises()`
After the existing text hydration, for each exercise with `type === 'question_arrange' && content.optionsMode === 'dynamic'`, run `hydrateArrangeOptionSources()` and replace `content.optionSources`.
- **Batching**: collect `exerciseId`s from all dynamic arrange exercises in the gem and fetch once, rather than once per exercise. (Text hydration still fetches per exercise. Batching that too is a possible follow-up, not required here.)
- Same-gem sources are hydrated too. If the user previously completed this gem (a retake), the server values act as a fallback, and the app overrides them with fresh local answers.
- Set `exercise.substitution_applied = true` when any source hydrated, for consistency.

### 2.4 Types and DI
- Extend the arrange content type in `content-management/models` (and wherever `GemsDatabaseResponse` content is typed).
- DI container: no new wiring, since `HydrationService` is already injected into `OnboardingService`.

### 2.5 Submission
No validator change. Arrange placements stay `{ value, row, column }`, and `value` is now a chip string. The validator doesn't cross-check values against `content.values`, so this is fine. Confirm with a test.

### 2.6 Tests (`supabase/functions/tests/`)
- `arrange-options-utility.test.ts`: wrapped/unwrapped, `chips`/`values`/`triggers` keys, object items, empty and missing response, multiple sources in order.
- `onboarding-service` test: a dynamic arrange gets `resolvedValues`; static arrange is untouched; a hydration failure doesn't fail `getGem`.

---

## 3. Mobile App (`project-ihsan/`)

### 3.1 Types (`src/views/screens/GemScreen/types.ts`)
```ts
export interface ArrangeOptionSource {
  gemId: number
  exerciseId: number
  exerciseType: 'question_chips'
  answerPath: string
  resolvedValues?: string[]
  hydrated?: boolean
}

export interface ArrangeQuestion extends BaseQuestion {
  type: QuestionType.QUESTION_ARRANGE
  triggers: Array<{ id: number; text: string }>   // static or pre-resolved
  optionsMode: 'static' | 'dynamic'
  optionSources?: ArrangeOptionSource[]
  grid?: ...
}
```

### 3.2 Pure resolver: `src/utils/ArrangeOptionsResolver.ts` (new)
```ts
resolveArrangeTriggers(
  question: ArrangeQuestion,
  answers: QuestionAnswers,
): Array<{ id: number; text: string }>
```
- `static` → return `question.triggers`.
- `dynamic` → for each source in order:
  1. If `answers[String(source.exerciseId)]` exists locally (a same-gem chips answer, `string[]` or chip objects), use it. **Local wins**, because it is fresher on a retake.
  2. Otherwise use `source.resolvedValues ?? []` (from the server, for earlier gems).
- Flatten, trim, de-duplicate case-insensitively, then map to `{ id: index + 1, text }`.
- Typed without `any`: add a `ChipAnswerItem = string | { text?: string; value?: string }` type.

### 3.3 `GemScreen/index.tsx`
- In the exercise → question mapper (`case 'question_arrange'`), carry `optionsMode: exercise.content.optionsMode ?? 'static'` and `optionSources`. Keep `triggers` from `values` (it will be empty for dynamic).
- Add a `useMemo` that computes the `resolvedArrangeTriggers` for the current question from `(currentQuestion, answers)` and passes it to the view. Logic stays in the container, so the view stays pure. It must re-run when `answers` changes, so that going back, editing chips and returning updates the options.

### 3.4 `GemScreen.view.tsx`
- Pass `resolvedArrangeTriggers` (rather than `arrangeQuestion.triggers`) to `ExerciseDragDropTriggersComponent`.
- **Reconcile saved placements**: filter `savedPlacements` to values that are still present in the resolved triggers (the user may have removed a chip after arranging). Include the triggers in the component `key` so it remounts when the option set changes.

### 3.5 Onboarding gem
`OnboardingGemScreen/index.tsx` has its own arrange mapping. Apply the same mapping and resolver so both screens behave the same, even though the admin is unlikely to use dynamic arrange in onboarding.

### 3.6 Caching
Cross-gem values come from `get-gem` via RTK Query. Verify that the gem query cache is invalidated (or refetched) after `postUserResponses` succeeds for the previous gem. Otherwise a user who completes Gem 1 and immediately opens Gem 2 could see an empty or stale option list. Existing text reference chips have the same dependency, so check the current tag invalidation and fix it if it's missing.

### 3.7 Tests (`src/tests/`)
- `ArrangeOptionsResolver.test.ts`: static passthrough, a same-gem local answer, a cross-gem server answer, local overriding server, multi-source order and dedupe, and object chip items.
- GemScreen behaviour: chips answer → arrange shows those chips; removing a chip drops its placement.

---

## Edge Cases

| Case | Behaviour |
|---|---|
| Source exercise skipped by flow rules in the current gem | No local answer and no server answer → that source contributes nothing. If **all** sources are empty, the component renders with no chips. Arrange isn't in `questionsRequiringValidation`, so the user can still continue, and an empty `placements` is submitted. Log a warning. |
| Source exercise deleted in admin | Backend finds no response → `hydrated: false`, empty. Admin warning (1.3) catches it before publish. |
| User edits chips after arranging | The resolver re-runs; placements whose values are gone are dropped. |
| Retake of a gem | Local answers override the server's `resolvedValues` for same-gem sources. |
| Old app versions | Will render `values: []` → empty arrange. **Rollout note**: release the app before admins start authoring dynamic arranges, or gate the feature by minimum app version. |
| Duplicate chips across sources | De-duplicated (case-insensitive, trimmed). |

---

## Implementation Order

1. Step 0: confirm the stored chips response shape.
2. Backend utility, service and tests (safe to ship first: dynamic content doesn't exist yet).
3. Mobile types, resolver, GemScreen wiring and tests, then the app release.
4. Admin dialog, picker, ordering warning and PDF.
5. Docs: update `REFERENCE_CHIP_HYDRATION.md` (backend) and `REFERENCE_VARIABLES.md` (admin) with an "Arrange option sources" section, and add `optionsMode`/`optionSources` to `USER_RESPONSES_ARCHITECTURE.md`'s exercise content schema.

## Files Touched

**Admin**
- `components/features/gems/CustomExerciseDialog/QuestionArrangeDialog.tsx`
- `components/features/gems/CustomExerciseDialog/types.ts`
- `types/exercises.ts`
- `components/features/gems/GemExercises.tsx` (ordering warning)
- `lib/services/pdf/quality-pdf-generator.ts`
- (possibly) a shared exercise picker extracted from `ReferenceTiptapEditor`

**Backend**
- `supabase/functions/services/utilities/arrange-options-utility.ts` (new)
- `supabase/functions/services/hydration-service.ts`
- `supabase/functions/services/onboarding-service.ts`
- `supabase/functions/content-management/models/*` (types)
- `supabase/functions/tests/...` (new tests)

**Mobile**
- `src/utils/ArrangeOptionsResolver.ts` (new)
- `src/views/screens/GemScreen/{types.ts,index.tsx,GemScreen.view.tsx}`
- `src/views/screens/OnboardingGemScreen/index.tsx`
- `src/tests/ArrangeOptionsResolver.test.ts` (new)

## Out of Scope
- Source types other than `question_chips`.
- Mixing static and dynamic options.
- Filtering chips (e.g. "only chips placed in cell X").
- Fixing the chips key inconsistency between the validator and the flow-rules hydrator (tracked as a follow-up).
