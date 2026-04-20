# Worktree Workspace Button

## Summary

Add a new button next to the existing plus (New Workspace) button in the titlebar controls. When clicked, it shows a popover with a text field for a worktree name. On submit, it creates a git worktree via `git gtr`, resolves the worktree directory path, and opens a new workspace with the terminal starting in that directory.

## UI Changes

### New Button

- **Location:** Immediately next to the existing plus button, in both titlebar mode and minimal/sidebar mode
- **Icon:** SF Symbol `"arrow.triangle.branch"` (git branch icon)
- **Accessibility label:** "New Workspace with Worktree"
- **Styled identically** to the existing plus button via `TitlebarControlButton`

### Popover

- Triggered by clicking the new button
- Contains:
  - A text field with placeholder "Worktree name (e.g. branch-A)"
  - A "Create" button, enabled only when text field is non-empty
- Enter key submits (same as clicking Create)
- Escape key dismisses the popover
- Minimal chrome: no title, just the input and action button

## Worktree Creation Flow

When the user submits a worktree name (e.g. `branch-A`):

1. Dismiss the popover
2. Get the current workspace's working directory as the shell cwd
3. Run `git gtr new branch-A` as a background `Process` from that directory
4. If step 3 succeeds, run `git gtr go branch-A` and capture stdout to get the worktree path
5. Create a new workspace via `TabManager.addWorkspace()` with `workingDirectory` set to the captured worktree path
6. The new workspace is automatically selected (default `addWorkspace` behavior)

### Error Handling

If `git gtr new` or `git gtr go` fails:
- Show an `NSAlert` with the error message (stderr output)
- Create a regular workspace in the current directory as fallback (same as clicking the normal plus button)

## Files to Modify

### `Sources/Update/UpdateTitlebarAccessory.swift`

- Add the new `arrow.triangle.branch` button in `TitlebarControlsView` (next to the existing plus button, ~line 399)
- Add `@State` for popover visibility
- Add the popover view (text field + Create button)
- Add `onNewWorktreeTab: (String) -> Void` callback parameter to `TitlebarControlsView`
- Same changes in `HiddenTitlebarSidebarControlsView` for minimal/sidebar mode

### `Sources/ContentView.swift`

- Update the minimal mode sidebar overlay (~line 10277) to pass the new `onNewWorktreeTab` closure
- The closure calls `TabManager.addWorktreeWorkspace(name:)`

### `Sources/TabManager.swift`

- Add `addWorktreeWorkspace(name: String)` method that:
  1. Runs `git gtr new <name>` via `Process` from the current workspace's working directory
  2. Runs `git gtr go <name>` via `Process` and captures stdout
  3. Trims the output path
  4. Calls `addWorkspace(workingDirectory: worktreePath)`
  5. On failure: shows `NSAlert`, falls back to `addWorkspace()` with no worktree path

## No New Files

All changes are additions to existing files.

## No Model Changes

`Workspace` already accepts a `workingDirectory` parameter. No changes to the data model.
