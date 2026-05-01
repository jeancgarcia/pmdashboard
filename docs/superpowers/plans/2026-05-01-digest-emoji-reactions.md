# Digest Emoji Reactions Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Add emoji reactions (🧠 ❗ ⭐ 📌) to digest action items and a global "Pinned" section above the digest list that surfaces every reacted item.

**Architecture:** Add a new `reactions` state (`{ [itemId]: { emoji, ts } }`) to the existing App-level state alongside `completed` and `edits`. Persist via the same `debouncedWrite` → localStorage + Supabase path. Modify `CheckItem` to expose a reaction button (hover), right-click handler, long-press handler, and a visible badge. Add two new components: `ReactionPicker` (floating popover) and `PinnedSection` (rendered between the type-pill row and the digest list when at least one reaction exists). No build step changes — this is a single-file React app transformed in-browser by Babel.

**Tech Stack:** React 18 (UMD via CDN), Babel-standalone in-browser JSX transform, TailwindCSS (CDN), Supabase JS client. All in `index.html`. No bundler, no test runner — manual verification via `npm run dev` (which runs `serve .`).

**Spec:** [docs/superpowers/specs/2026-05-01-digest-emoji-reactions-design.md](../specs/2026-05-01-digest-emoji-reactions-design.md)

**Verification approach:** This codebase has no automated test suite. Each task that ships behavior includes a manual verification step using the dev server (`npm run dev`, default `http://localhost:3000`).

---

## Task 1: Add `reactions` state + persistence

**Files:**
- Modify: `index.html` (App component state, load, save, handler)

- [ ] **Step 1: Add `reactions` state declaration**

In `index.html` inside `function App()`, find the line `const [edits, setEdits] = useState({});` (around line 2254) and add immediately after it:

```javascript
const [reactions, setReactions] = useState({});
```

- [ ] **Step 2: Load `reactions` from localStorage on mount**

In the `useEffect` that loads state on mount (around line 2331), find the block that reads `local.edits` and add a sibling line. The block currently looks like:

```javascript
const local = JSON.parse(localStorage.getItem("llf-state") || "{}");
if (local.completed && Object.keys(local.completed).length) setCompleted(local.completed);
if (local.hidden && local.hidden.length) setHidden(local.hidden);
// ...
if (local.edits && Object.keys(local.edits).length) setEdits(local.edits);
```

Add after the `local.edits` line:

```javascript
if (local.reactions && Object.keys(local.reactions).length) setReactions(local.reactions);
```

- [ ] **Step 3: Load `reactions` from remote**

In the same `useEffect`, inside `remoteRead().then(state => { ... })`, find the `if (!userActed.current)` branch. Currently:

```javascript
if (!userActed.current) {
  if (state && state.completed) setCompleted(state.completed);
  if (state && state.hidden) setHidden(state.hidden);
  if (state && state.edits) setEdits(state.edits);
}
```

Add after the `state.edits` line:

```javascript
  if (state && state.reactions) setReactions(state.reactions);
```

Then in the `else` branch (where the comment says "User already acted"), add after the existing edits-merge line:

```javascript
  if (state && state.reactions) setReactions(prev => Object.keys(prev).length ? prev : (state.reactions || {}));
```

- [ ] **Step 4: Include `reactions` in the persistence write**

Find the `useEffect` that calls `debouncedWrite` (around line 2382). It currently looks like:

```javascript
useEffect(() => {
  if (!stateLoaded || !userActed.current) return;
  debouncedWrite({ completed, hidden, reminders, reminderHistory, edits });
}, [completed, hidden, reminders, reminderHistory, edits, stateLoaded]);
```

Replace it with:

```javascript
useEffect(() => {
  if (!stateLoaded || !userActed.current) return;
  debouncedWrite({ completed, hidden, reminders, reminderHistory, edits, reactions });
}, [completed, hidden, reminders, reminderHistory, edits, reactions, stateLoaded]);
```

- [ ] **Step 5: Add `setReaction` handler**

Below the existing `handleEditItem` `useCallback` (around line 2392), add:

```javascript
const setReaction = useCallback((id, emoji) => {
  userActed.current = true;
  setReactions(prev => {
    const next = { ...prev };
    if (emoji == null) {
      delete next[id];
    } else {
      next[id] = { emoji, ts: Date.now() };
    }
    return next;
  });
}, []);
```

- [ ] **Step 6: Manual verify — state plumbing**

Run: `npm run dev`
Open the app, open DevTools → Application → Local Storage. The `llf-state` key should still parse and load. Check console for `[state] remote read OK:` — make sure the app still boots and digests render. No UI change yet.

- [ ] **Step 7: Commit**

```bash
git add index.html
git commit -m "feat(reactions): add reactions state with persistence

Adds reactions: { [itemId]: { emoji, ts } } to the App-level state,
load/save through localStorage and Supabase via the existing
debouncedWrite path."
```

---

## Task 2: Add the `ReactionPicker` component

**Files:**
- Modify: `index.html` (new component, placed before `CheckItem`)

- [ ] **Step 1: Add the `REACTION_EMOJIS` constant**

Find the line just before `function highlight(text, query)` (around line 1322). Add above it:

```javascript
/* ═══ Reactions ═══ */
const REACTION_EMOJIS = ["🧠", "❗", "⭐", "📌"];
```

- [ ] **Step 2: Add the `ReactionPicker` component**

Immediately below the `REACTION_EMOJIS` constant, add:

```javascript
function ReactionPicker({ currentEmoji, anchor, onPick, onRemove, onClose }) {
  const ref = useRef(null);

  useEffect(() => {
    const onDown = (e) => { if (ref.current && !ref.current.contains(e.target)) onClose(); };
    const onKey = (e) => { if (e.key === "Escape") onClose(); };
    document.addEventListener("mousedown", onDown);
    document.addEventListener("keydown", onKey);
    return () => {
      document.removeEventListener("mousedown", onDown);
      document.removeEventListener("keydown", onKey);
    };
  }, [onClose]);

  // Clamp position to viewport so picker never overflows.
  const PAD = 8;
  const W = 220;  // approx popover width
  const H = 44;   // approx popover height
  const vw = typeof window !== "undefined" ? window.innerWidth : 1024;
  const vh = typeof window !== "undefined" ? window.innerHeight : 768;
  const x = Math.max(PAD, Math.min((anchor?.x ?? PAD), vw - W - PAD));
  const y = Math.max(PAD, Math.min((anchor?.y ?? PAD), vh - H - PAD));

  return (
    <div
      ref={ref}
      style={{ position: "fixed", left: x, top: y, zIndex: 50 }}
      onClick={e => e.stopPropagation()}
      onContextMenu={e => e.preventDefault()}
      className="flex items-center gap-1 px-2 py-1.5 rounded-xl bg-white dark:bg-zinc-800 shadow-lg border border-zinc-200 dark:border-zinc-700"
    >
      {REACTION_EMOJIS.map(em => (
        <button
          key={em}
          onClick={() => { onPick(em); onClose(); }}
          className={`text-lg leading-none w-9 h-9 rounded-lg transition-all hover:scale-110 active:scale-95 ${currentEmoji === em ? "bg-zinc-100 dark:bg-zinc-700 ring-1 ring-blue-400" : "hover:bg-zinc-100 dark:hover:bg-zinc-700"}`}
          title={`React with ${em}`}
        >
          {em}
        </button>
      ))}
      {currentEmoji && (
        <button
          onClick={() => { onRemove(); onClose(); }}
          className="ml-1 text-[10px] font-bold uppercase tracking-wider text-zinc-500 hover:text-red-500 px-2 h-9 rounded-lg hover:bg-red-50 dark:hover:bg-red-500/10 transition-colors"
          title="Remove reaction"
        >
          Remove
        </button>
      )}
    </div>
  );
}
```

- [ ] **Step 3: Manual verify — no UI yet, just no errors**

Run: `npm run dev`. Reload the app. Confirm no console errors and digests still render (the component is defined but not yet referenced).

- [ ] **Step 4: Commit**

```bash
git add index.html
git commit -m "feat(reactions): add ReactionPicker component

Floating popover with 4 fixed emojis (🧠 ❗ ⭐ 📌) plus a Remove
button when an item already has a reaction. Closes on click outside
or Escape; clamps position to the viewport."
```

---

## Task 3: Wire `CheckItem` to support reactions

**Files:**
- Modify: `index.html` (`CheckItem` component, around lines 1354–1454, and the `Section` component that renders it, around line 1478)

- [ ] **Step 1: Update `CheckItem` signature and add picker state**

Replace the line `function CheckItem({ item, done, onToggle, searchQuery, editOverride, onEdit }) {` with:

```javascript
function CheckItem({ item, done, onToggle, searchQuery, editOverride, onEdit, reaction, onSetReaction }) {
```

Then find the existing `useState`/`useRef` block at the top of the function (where `editing`, `editLabel`, `editDetail`, `labelRef` are declared) and add immediately after the `labelRef` line:

```javascript
  const [picker, setPicker] = useState(null); // null | {x, y}
  const longPressTimer = useRef(null);
  const longPressFired = useRef(false);
  const touchStart = useRef(null);
```

- [ ] **Step 2: Add picker open/close handlers**

Below the `cancelEdit` function (still inside `CheckItem`), add:

```javascript
  const openPicker = (e, x, y) => {
    if (e) { e.preventDefault(); e.stopPropagation(); }
    setPicker({ x, y });
  };
  const closePicker = () => setPicker(null);

  const onContextMenu = (e) => {
    if (done) return;
    openPicker(e, e.clientX, e.clientY);
  };

  const onTouchStart = (e) => {
    if (done) return;
    longPressFired.current = false;
    const t = e.touches[0];
    touchStart.current = { x: t.clientX, y: t.clientY };
    longPressTimer.current = setTimeout(() => {
      longPressFired.current = true;
      const rect = e.currentTarget.getBoundingClientRect();
      openPicker(null, rect.left + 16, rect.top + rect.height + 4);
    }, 500);
  };
  const onTouchMove = (e) => {
    if (!touchStart.current) return;
    const t = e.touches[0];
    if (Math.abs(t.clientX - touchStart.current.x) > 10 || Math.abs(t.clientY - touchStart.current.y) > 10) {
      clearTimeout(longPressTimer.current);
    }
  };
  const onTouchEnd = () => {
    clearTimeout(longPressTimer.current);
    touchStart.current = null;
  };

  const onRowClick = (e) => {
    if (longPressFired.current) { longPressFired.current = false; return; }
    onToggle();
  };

  const openPickerFromButton = (e) => {
    e.stopPropagation();
    const rect = e.currentTarget.getBoundingClientRect();
    openPicker(e, rect.left, rect.bottom + 4);
  };
```

- [ ] **Step 3: Replace the row's render block with reaction-aware version**

Replace the entire return block at the bottom of `CheckItem` (the one starting with `return (` followed by the `<div onClick={onToggle} ...>`, around lines 1427–1454) with:

```javascript
  return (
    <div
      onClick={onRowClick}
      onContextMenu={onContextMenu}
      onTouchStart={onTouchStart}
      onTouchMove={onTouchMove}
      onTouchEnd={onTouchEnd}
      onTouchCancel={onTouchEnd}
      className={`group relative flex items-start gap-3 py-3 px-3 rounded-xl cursor-pointer select-none transition-all duration-200 hover:bg-zinc-100/50 dark:hover:bg-zinc-800/50 ${done ? "opacity-40" : ""}`}
    >
      <div className={`mt-0.5 w-5 h-5 rounded-full flex-shrink-0 flex items-center justify-center transition-all duration-300 ${done ? "bg-emerald-500 shadow-sm shadow-emerald-500/20 scale-105" : "border border-zinc-300 dark:border-zinc-600 bg-white dark:bg-zinc-900 group-hover:border-zinc-400 dark:group-hover:border-zinc-500"}`}>
        {done && <Icon.Check size={11} color="white" />}
      </div>
      <div className="flex-1 min-w-0 transition-transform duration-200 group-hover:translate-x-1">
        <div className="flex items-center gap-2 flex-wrap">
          <span className={`text-sm leading-snug transition-colors duration-200 ${done ? "line-through text-zinc-400 dark:text-zinc-500" : "text-zinc-800 dark:text-zinc-200 font-medium"}`}>{highlight(displayLabel, searchQuery)}</span>
          {item.flag === "urgent" && !done && <span className="w-1.5 h-1.5 rounded-full bg-red-500 shadow-[0_0_8px_rgba(239,68,68,0.5)] flex-shrink-0" />}
          {item.flag === "due" && !done && <span className="w-1.5 h-1.5 rounded-full bg-amber-500 shadow-[0_0_8px_rgba(245,158,11,0.5)] flex-shrink-0" />}
          {editOverride && !done && <span className="text-[9px] uppercase tracking-wider font-bold text-blue-400 dark:text-blue-500 px-1 py-0.5 bg-blue-50 dark:bg-blue-500/10 rounded flex-shrink-0">edited</span>}
          {reaction && <span className="text-base leading-none flex-shrink-0" title="Your reaction">{reaction.emoji}</span>}
        </div>
        {displayDetail && !done && <p className="text-xs text-zinc-500 dark:text-zinc-400 leading-relaxed mt-1">{highlight(displayDetail, searchQuery)}</p>}
      </div>
      {!done && onSetReaction && (
        <button
          onClick={openPickerFromButton}
          className="opacity-0 group-hover:opacity-100 transition-all duration-150 flex-shrink-0 mt-0.5 p-1.5 rounded-lg text-zinc-400 hover:text-amber-500 hover:bg-amber-50 dark:hover:bg-amber-500/10 dark:text-zinc-500 dark:hover:text-amber-400"
          title="React"
        >
          <span className="text-xs leading-none">😊</span>
        </button>
      )}
      {!done && onEdit && (
        <button
          onClick={startEdit}
          className="opacity-0 group-hover:opacity-100 transition-all duration-150 flex-shrink-0 mt-0.5 p-1.5 rounded-lg text-zinc-400 hover:text-blue-500 hover:bg-blue-50 dark:hover:bg-blue-500/10 dark:text-zinc-500 dark:hover:text-blue-400"
          title="Edit this item"
        >
          <Icon.Pencil size={12} />
        </button>
      )}
      {picker && (
        <ReactionPicker
          currentEmoji={reaction?.emoji}
          anchor={picker}
          onPick={(em) => onSetReaction(item.id, em)}
          onRemove={() => onSetReaction(item.id, null)}
          onClose={closePicker}
        />
      )}
    </div>
  );
```

- [ ] **Step 4: Pipe `reaction` and `onSetReaction` through `Section`**

Find `function Section({ section, completed, onToggle, defaultCollapsed, searchQuery, edits, onEditItem }) {` (around line 1478). Replace the signature with:

```javascript
function Section({ section, completed, onToggle, defaultCollapsed, searchQuery, edits, onEditItem, reactions, onSetReaction }) {
```

Then in the `section.items.map(item => (...))` that renders `CheckItem` (around lines 1506–1515), pass the new props. Replace:

```javascript
              : section.items.map(item => (
                  <CheckItem
                    key={item.id}
                    item={item}
                    done={!!completed[item.id]}
                    onToggle={() => onToggle(item.id)}
                    searchQuery={searchQuery}
                    editOverride={edits ? edits[item.id] : null}
                    onEdit={onEditItem || null}
                  />
                ))
```

with:

```javascript
              : section.items.map(item => (
                  <CheckItem
                    key={item.id}
                    item={item}
                    done={!!completed[item.id]}
                    onToggle={() => onToggle(item.id)}
                    searchQuery={searchQuery}
                    editOverride={edits ? edits[item.id] : null}
                    onEdit={onEditItem || null}
                    reaction={reactions ? reactions[item.id] : null}
                    onSetReaction={onSetReaction || null}
                  />
                ))
```

- [ ] **Step 5: Pipe `reactions` and `onSetReaction` through `DigestCard`**

Find `function DigestCard({ digest, completed, onToggle, onDelete, onRestore, defaultOpen, searchQuery, edits, onEditItem }) {` (around line 1526). Update the signature to:

```javascript
function DigestCard({ digest, completed, onToggle, onDelete, onRestore, defaultOpen, searchQuery, edits, onEditItem, reactions, onSetReaction }) {
```

Then find the `<Section ... />` JSX inside `DigestCard` (around line 1629). Replace it with:

```javascript
              <Section key={si} section={section} completed={completed} onToggle={onToggle} defaultCollapsed={false} searchQuery={searchQuery} edits={edits} onEditItem={onEditItem} reactions={reactions} onSetReaction={onSetReaction} />
```

- [ ] **Step 6: Pipe `reactions` and `onSetReaction` through `MonthGroup`**

Find `function MonthGroup({ monthKey, items, defaultExpanded, completed, onToggle, onDelete, onRestore, isArchiveView, searchQuery, edits, onEditItem }) {` (around line 2027). Update the signature to:

```javascript
function MonthGroup({ monthKey, items, defaultExpanded, completed, onToggle, onDelete, onRestore, isArchiveView, searchQuery, edits, onEditItem, reactions, onSetReaction }) {
```

Then find the `<DigestCard ... />` rendered inside `MonthGroup` (around lines 2055–2068) and add the two new props. The result should pass `reactions={reactions}` and `onSetReaction={onSetReaction}` alongside the existing props.

- [ ] **Step 7: Pass the props from App into the digest list renderers**

In the App component, find the two locations that render `<DigestCard ... />` and `<MonthGroup ... />` (around lines 2871–2905, in the `(search.trim() || selectedDate)` ternary). Add `reactions={reactions}` and `onSetReaction={setReaction}` to each.

For example, the `<DigestCard>` block becomes:

```javascript
                      <DigestCard
                        key={d.id}
                        digest={d}
                        completed={completed}
                        onToggle={toggle}
                        onDelete={isArchiveView ? null : deleteDigest}
                        onRestore={isArchiveView ? restoreDigest : null}
                        defaultOpen={isArchiveView ? false : i === 0}
                        searchQuery={search}
                        edits={edits}
                        onEditItem={handleEditItem}
                        reactions={reactions}
                        onSetReaction={setReaction}
                      />
```

And the `<MonthGroup>` block becomes:

```javascript
                      <MonthGroup
                        key={monthKey}
                        monthKey={monthKey}
                        items={items}
                        defaultExpanded={monthKey >= currentMonthKey}
                        completed={completed}
                        onToggle={toggle}
                        onDelete={deleteDigest}
                        onRestore={restoreDigest}
                        isArchiveView={isArchiveView}
                        searchQuery={search}
                        edits={edits}
                        onEditItem={handleEditItem}
                        reactions={reactions}
                        onSetReaction={setReaction}
                      />
```

- [ ] **Step 8: Manual verify — reacting works**

Run: `npm run dev`. Open a digest. Hover an action item: a smiley (😊) button should appear next to the pencil. Click it → ReactionPicker opens. Click 🧠 → picker closes, 🧠 appears as a badge on the row. Right-click another item → picker opens at cursor; pick ⭐. Reload the page → both reactions persist (badges still visible).

- [ ] **Step 9: Commit**

```bash
git add index.html
git commit -m "feat(reactions): add reaction button, badge, and picker triggers to CheckItem

Adds a hover emoji button beside the pencil, right-click handler,
touch long-press, and a visible badge for the current reaction.
Pipes reactions and onSetReaction through Section, DigestCard, and
MonthGroup."
```

---

## Task 4: Build the `PinnedSection` component

**Files:**
- Modify: `index.html` (new component, placed before `function App()`)

- [ ] **Step 1: Add the `PinnedSection` component**

Add this function just before `function App() {` (around line 2240):

```javascript
/* ═══ PinnedSection — global view of items the user has reacted to ═══ */
function PinnedSection({ digests, hidden, reactions, completed, edits, onToggle, onSetReaction }) {
  const [activeFilter, setActiveFilter] = useState("all"); // "all" | emoji

  // Build a flat lookup of action items from non-hidden digests, then match against reactions.
  const pinned = useMemo(() => {
    const hiddenSet = new Set(hidden || []);
    const itemMap = new Map();
    for (const d of digests) {
      if (hiddenSet.has(d.id)) continue;
      for (const s of d.sections || []) {
        if (s.info) continue;
        for (const it of s.items || []) itemMap.set(it.id, it);
      }
    }
    const list = [];
    for (const [id, rec] of Object.entries(reactions || {})) {
      const it = itemMap.get(id);
      if (!it) continue;
      list.push({ item: it, emoji: rec.emoji, ts: rec.ts || 0 });
    }
    list.sort((a, b) => b.ts - a.ts);
    return list;
  }, [digests, hidden, reactions]);

  // Counts per emoji (for chips), not affected by completion or active filter.
  const counts = useMemo(() => {
    const c = {};
    for (const p of pinned) c[p.emoji] = (c[p.emoji] || 0) + 1;
    return c;
  }, [pinned]);

  // Drop "done" items from the rendered list, then apply the sub-filter.
  const visible = useMemo(() => {
    const undone = pinned.filter(p => !completed[p.item.id]);
    if (activeFilter === "all") return undone;
    return undone.filter(p => p.emoji === activeFilter);
  }, [pinned, completed, activeFilter]);

  // If the active emoji has no items left (after completion), reset to All.
  useEffect(() => {
    if (activeFilter !== "all" && !counts[activeFilter]) setActiveFilter("all");
  }, [activeFilter, counts]);

  // Hide the entire section when nothing is pinned (counting completed items too).
  if (pinned.length === 0) return null;

  const totalUndone = pinned.filter(p => !completed[p.item.id]).length;

  return (
    <div className="mb-6 rounded-[2rem] border border-zinc-200 dark:border-zinc-700 bg-white/70 dark:bg-zinc-800/40 backdrop-blur-md p-4">
      <div className="flex items-center gap-2 mb-3">
        <span className="text-base">📌</span>
        <h2 className="text-sm font-bold text-zinc-700 dark:text-zinc-200">Pinned</h2>
        <span className="text-[10px] font-bold text-zinc-400 bg-zinc-100 dark:bg-zinc-700 px-1.5 py-0.5 rounded-md">{totalUndone}</span>
      </div>

      <div className="flex gap-2 flex-wrap mb-3">
        <button
          onClick={() => setActiveFilter("all")}
          className={`px-3 py-1 rounded-full text-[11px] font-bold transition-all ${activeFilter === "all" ? "bg-zinc-900 text-white dark:bg-white dark:text-zinc-900" : "bg-white dark:bg-zinc-800 text-zinc-600 dark:text-zinc-300 border border-zinc-200 dark:border-zinc-700"}`}
        >
          All
        </button>
        {REACTION_EMOJIS.filter(em => counts[em]).map(em => (
          <button
            key={em}
            onClick={() => setActiveFilter(em)}
            className={`px-3 py-1 rounded-full text-[11px] font-bold transition-all flex items-center gap-1 ${activeFilter === em ? "bg-zinc-900 text-white dark:bg-white dark:text-zinc-900" : "bg-white dark:bg-zinc-800 text-zinc-600 dark:text-zinc-300 border border-zinc-200 dark:border-zinc-700"}`}
          >
            <span>{em}</span>
            <span>{counts[em]}</span>
          </button>
        ))}
      </div>

      {visible.length === 0 ? (
        <p className="text-xs text-zinc-400 dark:text-zinc-500 italic px-1 py-2">All caught up — nothing pinned with {activeFilter}.</p>
      ) : (
        <div className="space-y-1">
          {visible.map(({ item, emoji }) => {
            const editOverride = edits ? edits[item.id] : null;
            const label = editOverride?.label || item.label;
            return (
              <div key={item.id} className="group flex items-center gap-3 py-2 px-3 rounded-xl hover:bg-zinc-100/60 dark:hover:bg-zinc-800/60 transition-colors">
                <button
                  onClick={() => onToggle(item.id)}
                  className="w-5 h-5 rounded-full flex-shrink-0 border border-zinc-300 dark:border-zinc-600 bg-white dark:bg-zinc-900 hover:border-zinc-400 dark:hover:border-zinc-500 transition-colors"
                  title="Mark as done"
                />
                <span className="flex-1 min-w-0 text-sm text-zinc-800 dark:text-zinc-200 font-medium truncate">{label}</span>
                <span className="text-base leading-none flex-shrink-0">{emoji}</span>
                <button
                  onClick={() => onSetReaction(item.id, null)}
                  className="opacity-0 group-hover:opacity-100 transition-all flex-shrink-0 p-1 rounded-md text-zinc-400 hover:text-red-500 hover:bg-red-50 dark:hover:bg-red-500/10"
                  title="Remove reaction"
                >
                  <Icon.X size={12} />
                </button>
              </div>
            );
          })}
        </div>
      )}
    </div>
  );
}
```

- [ ] **Step 2: Manual verify — no errors**

Run: `npm run dev`. Reload. Confirm no console errors and the digest list still renders unchanged. The component is defined but not yet used.

- [ ] **Step 3: Commit**

```bash
git add index.html
git commit -m "feat(reactions): add PinnedSection component

Renders a global panel of reacted items grouped by emoji sub-filter.
Hidden when no reactions exist. Auto-clean: completed items drop out
of the rendered list, last-of-its-kind emoji resets active filter to
All, and the whole section unmounts when nothing remains pinned."
```

---

## Task 5: Mount `PinnedSection` between the type pills and the digest list

**Files:**
- Modify: `index.html` (App render, between the type-pill `</div>` and the `{/* Digest content */}` block)

- [ ] **Step 1: Insert `<PinnedSection />` after the type pills**

Find the closing `</div>` of the type-filter container (the one ending the `{types.filter(t => t !== "toggl").map(t => { ... })}` block, around line 2851) followed by `)}` (closing the `isRemindersView` ternary). Just before the comment `{/* Digest content */}` (around line 2855), insert:

```javascript
              {!isRemindersView && (
                <PinnedSection
                  digests={digests}
                  hidden={hidden}
                  reactions={reactions}
                  completed={completed}
                  edits={edits}
                  onToggle={toggle}
                  onSetReaction={setReaction}
                />
              )}
```

The exact context — replace this:

```javascript
                </div>
              </div>
              )}

              {/* Digest content */}
```

with:

```javascript
                </div>
              </div>
              )}

              {!isRemindersView && (
                <PinnedSection
                  digests={digests}
                  hidden={hidden}
                  reactions={reactions}
                  completed={completed}
                  edits={edits}
                  onToggle={toggle}
                  onSetReaction={setReaction}
                />
              )}

              {/* Digest content */}
```

- [ ] **Step 2: Manual verify — full flow**

Run: `npm run dev`. Reload.

1. Open a digest. Right-click an action item → pick 🧠.
   - Expected: pinned section appears between the pills and the digest list, header `📌 Pinned (1)`, `All` chip + a `🧠 1` chip, and the item listed inside.
2. React to a different item with ⭐.
   - Expected: section now shows `📌 Pinned (2)`, chips `All`, `🧠 1`, `⭐ 1`. Both items listed (most recent on top).
3. Click `⭐ 1` chip.
   - Expected: list filters to only the ⭐ item.
4. Hover the ⭐ row in the pinned section, click the `×`.
   - Expected: ⭐ chip disappears, active filter resets to `All`, only the 🧠 item remains, header shows `📌 Pinned (1)`.
5. Click the checkbox on the 🧠 row in the pinned section.
   - Expected: pinned section unmounts entirely (last item is now done). The original item in the digest below also shows as done with the ✅ check.
6. Uncheck that item in the digest below.
   - Expected: pinned section reappears (reaction record was preserved).
7. On a touch device or DevTools mobile emulation, long-press an item → picker opens; release → no toggle fires.
8. Switch the type pill from `All` to `Daily`.
   - Expected: digest list filters but the pinned section keeps showing all reactions (global scope).
9. Reload the page.
   - Expected: all reactions and the pinned section persist.

- [ ] **Step 3: Commit**

```bash
git add index.html
git commit -m "feat(reactions): mount PinnedSection between type pills and digest list

Section is global (ignores the type-pill filter) and only renders when
at least one item has a reaction. Hidden on the Reminders view."
```

---

## Self-review checklist (run before handing off)

- [ ] Spec coverage:
  - Fixed set 🧠 ❗ ⭐ 📌 → Task 2 step 1.
  - One reaction per item → Task 1 step 5 (`setReaction` overwrites).
  - State shape `{ [id]: { emoji, ts } }` → Task 1 step 5.
  - Persistence local + remote → Task 1 steps 2, 3, 4.
  - Hover button + right-click + long-press → Task 3 steps 2, 3.
  - Visible badge on item → Task 3 step 3 (`{reaction && <span>`).
  - Picker remove button → Task 2 step 2.
  - Esc + click-outside close → Task 2 step 2.
  - Position clamping → Task 2 step 2.
  - Pinned section between pills and digest list → Task 5 step 1.
  - Hidden when empty → Task 4 step 1 (`if (pinned.length === 0) return null`).
  - Header with count → Task 4 step 1.
  - Sub-filter chips with counts → Task 4 step 1 (`REACTION_EMOJIS.filter(em => counts[em])`).
  - Active sub-filter resets when its emoji's count hits 0 → Task 4 step 1 (`useEffect`).
  - Auto-clean on complete (no render) and on remove (delete from `reactions`) → Task 4 step 1.
  - Reaction record preserved on complete/uncomplete cycle → Task 4 step 1 (only filtered out at render).
  - Global scope, ignores type-pill filter → Task 4 step 1 (uses `digests` directly).
  - Hidden digests excluded but reactions preserved → Task 4 step 1 (`hiddenSet`).
  - Click on text does nothing → Task 4 step 1 (the `<span>` has no handler).
  - Long-press suppresses toggle → Task 3 step 2 (`onRowClick` checks `longPressFired`).

- [ ] Type consistency:
  - `setReaction(id, emoji|null)` signature is consistent across App, CheckItem, ReactionPicker, PinnedSection.
  - Prop name `onSetReaction` consistent across CheckItem, Section, DigestCard, MonthGroup, PinnedSection.
  - Reaction record shape `{ emoji, ts }` is read consistently (`reaction.emoji`, `rec.emoji`).

- [ ] No placeholders, no "similar to Task N", no "TODO".

---

## Execution Handoff

Plan complete and saved to [docs/superpowers/plans/2026-05-01-digest-emoji-reactions.md](docs/superpowers/plans/2026-05-01-digest-emoji-reactions.md). Two execution options:

**1. Subagent-Driven (recommended)** — I dispatch a fresh subagent per task, review between tasks, fast iteration.

**2. Inline Execution** — Execute tasks in this session using executing-plans, batch execution with checkpoints.

Which approach?
