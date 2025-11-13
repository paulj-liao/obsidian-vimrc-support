# Handler Registration Audit

## Problem: Handlers Registered in Multiple Places

### vim-keypress and vim-command-done:
1. `prepareChordDisplay()` - registers these
2. `updateVimEvents()` - ALSO registers these

### vim-mode-change and keydown:
1. `updateVimEvents()` - registers these  
2. `readVimInit()` - ALSO registers these

### cursorActivity:
1. `updateSelectionEvent()` - registers this

## The Issue

Even with deduplication flags, if `getCodeMirror()` returns different object references for the same logical editor, the Map keys won't match and we'll register duplicate handlers!

```typescript
private getCodeMirror(view: MarkdownView): CodeMirror.Editor {
    return (view as any).editMode?.editor?.cm?.cm;
}
```

This chain of property accesses might return different objects at different times!

## The Real Fix

We need to:
1. Register ALL handlers in ONE place only
2. Remove duplicate registrations
3. Use the View as the key instead of the unstable CM object
