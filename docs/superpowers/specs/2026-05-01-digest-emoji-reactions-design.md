# Digest Emoji Reactions — Design

**Date:** 2026-05-01
**Status:** Approved (pending implementation plan)

## Goal

Let the user "react" to action items in any digest with a small fixed set of emojis, and surface every reacted item in a pinned section above the digest list. Pinned items can be completed or un-reacted to remove them from the section.

## Scope

- Applies to **action items only** (items inside non-`info` sections). `InfoItem` rows are not reactable.
- Works across **all digests**, regardless of which type pill (`All/Daily/EOD/Weekly/Archived`) is currently selected.
- Persists locally — no backend changes.

## Reaction model

- **Fixed set:** `🧠 ❗ ⭐ 📌` (no custom emojis, no other set).
- **Cardinality:** one reaction per item. Re-reacting with a different emoji replaces the previous one.
- **State shape:** `reactions: { [itemId: string]: { emoji: string, ts: number } }`. An item with no reaction is simply absent from the object. `ts` is `Date.now()` at the time the reaction was set; it's used to sort the pinned list (most recent on top).
- **Persistence:** stored in `localStorage` alongside `completed` and `edits`, using the same load/save pattern those use today.

## Triggering a reaction

Three input modes, all opening the same popover (`ReactionPicker`):

1. **Desktop hover button** — on `CheckItem` hover, a small emoji button appears immediately to the left of the existing pencil edit button. Click opens the picker anchored to the button.
2. **Right-click anywhere on the item row** — opens the picker at the cursor position. Default browser context menu is suppressed only on action items.
3. **Long-press (~500ms) on touch** — opens the picker centered on the item. Cancelled if the user starts scrolling. The long-press also suppresses the row's normal `onClick` toggle so the item doesn't get marked done as a side effect.

The picker is a small floating panel containing:

- The 4 emojis as buttons.
- The currently-selected emoji is visually highlighted if the item already has a reaction.
- A `Remove` button shown only when the item already has a reaction.
- Click outside or `Esc` closes it.

Selecting an emoji writes to `reactions`; selecting the same emoji that's already set is a no-op (UI just closes). `Remove` deletes the key.

## CheckItem visual changes

- When an item has a reaction, its current emoji is shown as a small badge (e.g. inline pill or chip) at the right end of the row, after the urgent/due dots and the "edited" badge. The badge is always visible (not hover-only) so the user can see which items they've reacted to without hovering.
- The new emoji button on hover sits to the immediate left of the pencil. Same visual style/sizing as the pencil. Tooltip: "React".
- When the item is `done`, both the badge and the hover button still render, but at the same opacity treatment as the rest of the (faded) row.

## Pinned section

A new component, rendered between the type-pill row and the digest list, only when `Object.keys(reactions).length > 0`.

**Header:** `📌 Pinned (N)` — N is the total count of reacted items, regardless of sub-filter.

**Sub-filter chips:** rendered as a horizontal row under the header.

- `All` chip is always shown and always selectable.
- One chip per emoji that has at least one reacted item, in the order `🧠 ❗ ⭐ 📌`. Chip label is `<emoji> <count>`.
- Chips that would have count 0 are not rendered.
- Active sub-filter is local to this component (not part of saved state). Defaults to `All` on every page load. If the active emoji's count drops to 0 (e.g. user un-reacted the last item of that emoji), the active chip auto-resets to `All`.

**Item list:** flat list (no grouping by digest, no headers), showing items that match the active sub-filter, in stable insertion order (most recently reacted at the top).

Each row renders:

- Checkbox on the left, identical to `CheckItem`'s checkbox. Clicking it toggles `completed[itemId]`. If the item becomes `done`, it disappears from the pinned section on the next render (because completing removes it from the section per the rule below).
- The item's `displayLabel` (respects `edits` like the digest does). Clicking the text does nothing.
- The reaction emoji as a badge on the right.
- An `×` button after the badge that removes the reaction (`reactions[itemId] = undefined`). Tooltip: "Remove reaction".

**Scope:** uses the *global* `reactions` map and the live `entries`/`digests` data. The type pill filter (`All/Daily/EOD/Weekly/Archived`) does not affect it. Search and date filters also do not affect it (the section's purpose is to be a stable workspace).

**Auto-clean rules:**

- Completing an item (anywhere — pinned section or digest below) removes it from the pinned section's rendered list. The reaction record itself is **kept** in `reactions` so re-uncompleting brings it back. (This matches how `completed` works as the source of truth for "is it done".)
- Removing the reaction (×) deletes from `reactions` permanently.
- If after either action no items remain pinned (taking completion into account), the section unmounts.

## Data flow

```
state in App component
├─ completed: { [itemId]: bool }      // existing
├─ edits:     { [itemId]: {label, detail} }  // existing
└─ reactions: { [itemId]: emoji }     // NEW
```

- `setReaction(itemId, emojiOrNull)` — single handler used by `ReactionPicker` and the pinned `×` button.
- `toggle(itemId)` — existing handler, used unchanged by both `CheckItem` and `PinnedSection` rows.
- Item lookup for the pinned section: a `useMemo` that flattens all action items across non-hidden digests and joins against `reactions`. Items whose digest is hidden (archived) or deleted are filtered out of the pinned section, but their reaction record is preserved in `reactions`. Order in the rendered list: reverse-chronological by reaction time — see "Insertion order" below.

**Insertion order:** to render most-recently-reacted at the top, `reactions` stores not just the emoji but the timestamp of when it was set. Shape becomes `reactions: { [itemId]: { emoji, ts } }`. The pinned list sorts by `ts` descending. Re-reacting updates `ts`. Removing deletes the entry.

## Components

- **`ReactionPicker`** (new) — floating popover. Props: `currentEmoji`, `onPick(emoji)`, `onRemove()`, `onClose()`, `anchor` ({x,y} or DOMRect).
- **`PinnedSection`** (new) — header + sub-filter chips + list of pinned rows. Props: `digests`, `reactions`, `completed`, `edits`, `onToggle`, `onSetReaction`.
- **`CheckItem`** (modified) — adds:
  - Hover emoji button.
  - Reaction badge on the row when `reactions[item.id]` exists.
  - `onContextMenu` handler that opens picker.
  - Touch long-press handler.
  - New props: `reaction`, `onSetReaction`.

## Edge cases

- **Reacting then archiving the digest:** the reaction stays in `reactions` but the item won't render in the pinned section because its digest is hidden. Restoring the digest brings it back.
- **Reacting then deleting the digest:** if `deleteDigest` permanently removes a digest's items, orphaned reactions accumulate. Acceptable — they don't render anywhere and are negligible in size. No GC needed for v1.
- **Right-click on `InfoItem`:** falls through to default browser context menu (info items are not reactable).
- **Long-press conflict with scroll:** if the user moves their finger more than ~10px before the 500ms threshold, the long-press is cancelled.
- **Picker positioning near viewport edges:** clamp to viewport with a small margin so the popover never overflows.
- **Keyboard:** `Esc` closes the picker. Tab navigation through the 4 emoji buttons + Remove. Reaction button on the item row is reachable via Tab.

## Out of scope (not in v1)

- Custom emojis or arbitrary emoji picker.
- Multiple reactions on one item.
- Sorting / grouping the pinned section beyond the sub-filter.
- Notes or comments attached to a reaction.
- Sync across devices (everything is local).
- Reactions on `InfoItem` rows.
- Keyboard shortcut to open the picker without a pointer (could add later).
