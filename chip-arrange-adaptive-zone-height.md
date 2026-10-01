# Plan: Adaptive Drop Zone Height for Chip Arrange Screen

## Problem

`ExerciseDragDropTriggersComponent` uses a fixed `zoneHeight: 120` for each drop zone. When more chips than ~3 are placed in one zone, the chips overflow the visible box and the hit-detection boundary, breaking both the visual and drag-and-drop behaviour.

## Decisions

| Question | Answer |
|---|---|
| Row symmetry | **Symmetric** — both zones in a row share the same height (the taller of the two) |
| Hit area | **Expand hit area** — zone bounds used for drag detection must update alongside visual height |
| Minimum height | **120 px** — empty zones stay at 120 px as a drop-target placeholder |

---

## Root Cause

Three places hard-code `zoneHeight = 120`:

1. `getLayoutDimensions()` — JS helper used when computing where to **place chips** inside zones and where to **render** row 2 zones.
2. `getZoneForPosition()` — **Reanimated worklet** running on the UI thread; uses hardcoded `120` and `291` to decide which zone a chip was dropped into.
3. `styles.dropZone` — `height: 120` in the StyleSheet, controlling the **visual box**.

Additionally, the side-label layout (`leftLabelsContainer`, second `sideLabel` `marginTop: 140`) is hard-coded and will misalign when row 1 grows.

---

## Approach

### 1. Compute dynamic row heights from chip counts

Add a derived value `zoneHeights` that is recalculated whenever `triggers` changes:

```ts
const chipHeight = 26;
const chipGap = 8;
const zonePadding = 14;
const minZoneHeight = 120;

function computeZoneHeight(chipCount: number): number {
  if (chipCount === 0) return minZoneHeight;
  const contentHeight = chipCount * (chipHeight + chipGap) - chipGap;
  return Math.max(minZoneHeight, contentHeight + zonePadding * 2);
}
```

Each row takes the **max** of its two zones:

```ts
const row1Height = Math.max(
  computeZoneHeight(chipsInZone['difficult-low']),
  computeZoneHeight(chipsInZone['difficult-high'])
);
const row2Height = Math.max(
  computeZoneHeight(chipsInZone['easy-low']),
  computeZoneHeight(chipsInZone['easy-high'])
);
```

### 2. Store heights in Reanimated shared values

The `getZoneForPosition` worklet cannot read React state. Create two shared values that shadow the JS state:

```ts
const sharedRow1Height = useSharedValue(minZoneHeight);
const sharedRow2Height = useSharedValue(minZoneHeight);
```

Update them inside `updateTriggerPosition` (already runs on JS thread after each drop) and also inside the saved-placements `useEffect`.

### 3. Update `getZoneForPosition` worklet

Replace the hardcoded `zoneHeight = 120` with the shared values:

```ts
const getZoneForPosition = (x: number, y: number): ZoneName | 'unassigned' => {
  'worklet';
  const r1H = sharedRow1Height.value;
  const r2H = sharedRow2Height.value;
  const row2StartY = 291 + r1H + 20; // gridStartY + row1Height + gridGap
  // ... zone bounds use r1H / r2H / row2StartY
};
```

### 4. Update `getLayoutDimensions` and `getDropZones`

Pass computed heights in instead of the constant `120`. The `getDropZones` call should accept `row1Height` and `row2Height` parameters (or read them from the same derived state).

### 5. Update visual rendering

- Remove the fixed `height: 120` from `styles.dropZone`.
- Pass a `height` prop to `AnimatedDropZone` and apply it inline: `style={[styles.dropZone, { height: rowHeight }]}`.
- Both zones in the same row receive the same `rowHeight` value.

### 6. Update side-label layout

The second side label uses `marginTop: 140` (≈ zoneHeight + gridGap / 2). Replace with a dynamic value:

```ts
const secondLabelMarginTop = row1Height + 20; // row1Height + gridGap
```

Pass it as an inline style on the second `sideLabel` View.

Also increase `leftLabelsContainer` height dynamically:

```ts
const labelsContainerHeight = row1Height + row2Height + 20; // two rows + gap
```

### 7. Chip background colour states

Currently `styles.chip` sets `backgroundColor: customTheme.colors.backgroundColor`, which matches the screen background and makes chips hard to distinguish.

**New colour rules:**

| State | Background |
|---|---|
| Idle / placed in zone | White (`#FFFFFF`) |
| Being dragged over a valid zone | Green (suggested: `#4CAF50` or a theme green if one exists) |
| Released into zone / unassigned | White (`#FFFFFF`) |

**Implementation** — all colour work happens inside `DraggableChip` via `useAnimatedStyle`, keeping it on the UI thread without extra JS state:

```ts
const backgroundColor = useSharedValue('#FFFFFF');

// In panGesture.onUpdate — when hoveredZone changes to a valid zone:
backgroundColor.value = zone !== 'unassigned' ? '#4CAF50' : '#FFFFFF';

// In panGesture.onEnd — always reset:
backgroundColor.value = '#FFFFFF';
```

Add `backgroundColor` to the `animatedStyle` returned by `useAnimatedStyle`. Remove `backgroundColor` from the static `styles.chip` StyleSheet entry so the animated value takes full control.

The existing `hoveredZone` shared value already tracks which zone the chip is over, so no new shared state is needed — the colour just reacts to the same signal.

---

## Files to Change

| File | Changes |
|---|---|
| `src/views/components/gems/ExerciseDragDropTriggersComponent.tsx` | All changes — see steps 1–6 above |

No other files need to change. The parent `GemScreen.view.tsx` just renders this component and passes props through.

---

## Step-by-Step Implementation

1. **Add `computeZoneHeight` helper** near the top of the component (outside JSX, inside module scope or as a module-level function).

2. **Add `chipsPerZone` derived variable** — computed via `useMemo` on `triggers`:
   ```ts
   const chipsPerZone = useMemo(() => ({
     'difficult-low': triggers.filter(t => t.zone === 'difficult-low').length,
     'difficult-high': triggers.filter(t => t.zone === 'difficult-high').length,
     'easy-low': triggers.filter(t => t.zone === 'easy-low').length,
     'easy-high': triggers.filter(t => t.zone === 'easy-high').length,
   }), [triggers]);
   ```

3. **Derive `row1Height` and `row2Height`** from `chipsPerZone` via `useMemo`.

4. **Add `sharedRow1Height` and `sharedRow2Height`** shared values; sync them in a `useEffect` that depends on `row1Height` and `row2Height`.

5. **Update `getZoneForPosition` worklet** to use shared values instead of hardcoded `120`.

6. **Update `getLayoutDimensions`** to accept/return dynamic heights; update `getDropZones` accordingly.

7. **Update `AnimatedDropZone`** to accept a `height` prop; update the two rows' render calls to pass the correct row height.

8. **Update side-label `marginTop` and container height** with derived values.

9. **Chip background colours** — inside `DraggableChip`:
   - Add a `backgroundColor` shared value initialised to `'#FFFFFF'`.
   - In `panGesture.onUpdate`, set it to green when `hoveredZone.value` is a valid zone, white otherwise.
   - In `panGesture.onEnd`, reset it to `'#FFFFFF'`.
   - Include `backgroundColor` in `useAnimatedStyle`; remove the static `backgroundColor` from `styles.chip`.
   - Update the static `styles.chip` default background to `'#FFFFFF'` as a fallback.

10. **Verify** by adding 4+ chips to one zone — the box should expand, chips should remain fully visible, drag-drop should still snap correctly, chips should appear white at rest and green while hovering over a valid zone.

---

## Edge Cases

- **Chips moving between zones**: `updateTriggerPosition` already recalculates chip positions after every move. The shared values and derived heights will also update at that point, so zone bounds stay in sync.
- **Saved placements**: The `useEffect` for saved placements sets trigger positions after mount. After that effect runs, `chipsPerZone` will update via the `triggers` state change, triggering the height sync effect.
- **Very long chip text**: Chips have `maxHeight: 38`. `computeZoneHeight` uses `chipHeight = 26` (the `minHeight`). If wrapping causes a chip to render taller, the zone might still be slightly short for that chip. This is an acceptable edge case for this iteration.
