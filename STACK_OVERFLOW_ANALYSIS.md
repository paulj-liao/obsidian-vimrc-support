# Deep Dive: Stack Overflow Root Cause Analysis

## The Problem
Pressing `o` or `O` causes:
```
RangeError: Maximum call stack size exceeded
at e.update → e.updatePlugins → e.update (infinite recursion)
```

## Root Causes Found

### 🐛 Bug #1: `prepareChordDisplay()` Missing Flag Update

**Location**: main.ts:663-673

```typescript
if (this.vimEventsRegistered.get(cmEditor)) {
    return;  // ✅ Checks flag
}

(cmEditor as any).off('vim-keypress', this.onVimKeypress);
(cmEditor as any).on('vim-keypress', this.onVimKeypress);
(cmEditor as any).off('vim-command-done', this.onVimCommandDone);
(cmEditor as any).on('vim-command-done', this.onVimCommandDone);
// ❌ MISSING: this.vimEventsRegistered.set(cmEditor, true);
```

**Impact**: Every time user action occurs, handlers get re-registered on top of existing ones.

---

### 🐛 Bug #2: Shared Flag Causes Handler Conflicts

**The Problem**: THREE functions use ONE flag for DIFFERENT handlers:

1. `updateSelectionEvent()` - registers `cursorActivity` → sets flag
2. `updateVimEvents()` - registers `vim-mode-change`, `vim-keypress`, `vim-command-done`, `keydown` 
3. `prepareChordDisplay()` - registers `vim-keypress`, `vim-command-done`

**Execution Order** (on tab switch):
```
1. updateSelectionEvent() runs
   - Check flag: FALSE
   - Register: cursorActivity
   - Set flag: TRUE

2. updateVimEvents() runs  
   - Check flag: TRUE ← Set by step 1!
   - RETURN WITHOUT REGISTERING ANYTHING ❌
```

**Result**: `updateVimEvents()` NEVER registers its handlers because `updateSelectionEvent()` always runs first!

---

### 🐛 Bug #3: Handler Accumulation Pattern

**Current flow when switching between 3 tabs:**

```
Tab 1:
  prepareChordDisplay: vim-keypress, vim-command-done (no flag set)
  updateSelectionEvent: cursorActivity (flag=true)
  updateVimEvents: SKIPPED (flag already true)

Tab 2 (NEW editor):
  prepareChordDisplay: NOT called (only runs once)
  updateSelectionEvent: cursorActivity (flag=true for THIS editor)
  updateVimEvents: SKIPPED (flag already true)

Tab 3 (back to Tab 1):
  updateSelectionEvent: checks flag (true), SKIPS
  updateVimEvents: checks flag (true), SKIPS
  
Result: No new handlers registered, but old ones remain
```

But wait - `prepareChordDisplay()` doesn't set flag, so if it gets called again somehow, it keeps adding handlers!

---

### 🔍 Why the Stack Overflow Happens

When you press `o` or `O`:

1. **Vim mode changes** → triggers `vim-mode-change` event
2. **Multiple duplicate handlers fire** (because flag wasn't set properly)
3. **Each handler triggers editor updates**
4. **Editor updates trigger more vim events**
5. **Infinite recursion** → `e.update` → `e.updatePlugins` → `e.update` → ...

The editor's update cycle gets stuck because duplicate handlers keep triggering each other.

---

## The Fix

### Solution 1: Add Missing Flag in `prepareChordDisplay()`

```typescript
(cmEditor as any).on('vim-keypress', this.onVimKeypress);
(cmEditor as any).on('vim-command-done', this.onVimCommandDone);

// ADD THIS LINE:
this.vimEventsRegistered.set(cmEditor, true);
```

### Solution 2: Use Separate Tracking for Different Handler Types

Instead of one boolean flag, track which handlers are registered:

```typescript
private registeredHandlers: Map<any, Set<string>> = new Map();

// Check if specific handler is registered
if (!this.registeredHandlers.get(cm)?.has('cursorActivity')) {
    cm.on('cursorActivity', ...);
    if (!this.registeredHandlers.has(cm)) {
        this.registeredHandlers.set(cm, new Set());
    }
    this.registeredHandlers.get(cm).add('cursorActivity');
}
```

### Solution 3: Register All Handlers in One Place

Create a single `registerAllEditorHandlers(cm)` function called from one location to avoid conflicts.

---

## Recommended Fix

Implement **Solution 1** (quick fix) + part of **Solution 2** (proper tracking):

1. Add missing `this.vimEventsRegistered.set(cmEditor, true)` in `prepareChordDisplay()`
2. Use separate flags:
   - `cursorActivityRegistered: Map<any, boolean>`
   - `vimEventsRegistered: Map<any, boolean>`  
3. Remove the early returns that cause handler skipping

This ensures:
- ✅ Each handler type tracked independently
- ✅ No duplicate registration
- ✅ All necessary handlers registered
- ✅ No conflicts between different functions
