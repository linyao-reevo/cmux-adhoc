# Worktree Workspace Button Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Add a button next to the existing plus button that creates a new workspace backed by a git worktree via `git gtr`.

**Architecture:** The new button lives in the existing `TitlebarControlsView` alongside the plus button. Clicking it shows a popover with a text field. On submit, `TabManager.addWorktreeWorkspace(name:)` runs `git gtr new <name>` and `git gtr go <name>` via `Process`, then calls `addWorkspace(workingDirectory:)` with the resolved path. On failure, an `NSAlert` is shown and a regular workspace is created as fallback.

**Tech Stack:** SwiftUI, AppKit (`NSMenu`, `NSAlert`, `Process`), macOS

---

## File Map

| File | Action | Responsibility |
|------|--------|---------------|
| `Sources/TabManager.swift` | Modify | Add `addWorktreeWorkspace(name:)` method |
| `Sources/Update/UpdateTitlebarAccessory.swift` | Modify | Add worktree button + popover to `TitlebarControlsView`, wire up in `HiddenTitlebarSidebarControlsView` and `TitlebarControlsAccessoryViewController` |

---

### Task 1: Add `addWorktreeWorkspace(name:)` to TabManager

**Files:**
- Modify: `Sources/TabManager.swift:3781-3785` (near `addTab`)

- [ ] **Step 1: Add the `addWorktreeWorkspace` method**

Insert this method right after the `addTab` convenience alias (after line 3785 in `Sources/TabManager.swift`):

```swift
    /// Creates a git worktree via `git gtr new <name>`, resolves its path via
    /// `git gtr go <name>`, then opens a new workspace rooted at that path.
    /// On failure, shows an alert and falls back to a regular new workspace.
    @discardableResult
    func addWorktreeWorkspace(name: String) -> Workspace {
        let cwd: String? = selectedWorkspace?.focusedTerminalPanel?.surface.currentWorkingDirectory
        let fallbackCwd = cwd ?? FileManager.default.currentDirectoryPath

        // Step 1: git gtr new <name>
        let createResult = runGitGtr(arguments: ["new", name], cwd: fallbackCwd)
        if let error = createResult.error {
            showWorktreeError(
                title: String(localized: "worktree.error.createTitle", defaultValue: "Failed to create worktree"),
                message: error
            )
            return addWorkspace()
        }

        // Step 2: git gtr go <name> to get the path
        let goResult = runGitGtr(arguments: ["go", name], cwd: fallbackCwd)
        guard let worktreePath = goResult.output, !worktreePath.isEmpty else {
            showWorktreeError(
                title: String(localized: "worktree.error.resolvePath", defaultValue: "Failed to resolve worktree path"),
                message: goResult.error ?? String(localized: "worktree.error.emptyPath", defaultValue: "git gtr go returned an empty path")
            )
            return addWorkspace()
        }

        return addWorkspace(
            title: name,
            workingDirectory: worktreePath
        )
    }

    private struct GitGtrResult {
        let output: String?
        let error: String?
    }

    private func runGitGtr(arguments: [String], cwd: String) -> GitGtrResult {
        let process = Process()
        process.executableURL = URL(fileURLWithPath: "/usr/bin/env")
        process.arguments = ["git", "gtr"] + arguments
        process.currentDirectoryURL = URL(fileURLWithPath: cwd)

        let stdoutPipe = Pipe()
        let stderrPipe = Pipe()
        process.standardOutput = stdoutPipe
        process.standardError = stderrPipe

        do {
            try process.run()
            process.waitUntilExit()
        } catch {
            return GitGtrResult(output: nil, error: error.localizedDescription)
        }

        let stdoutData = stdoutPipe.fileHandleForReading.readDataToEndOfFile()
        let stderrData = stderrPipe.fileHandleForReading.readDataToEndOfFile()
        let stdout = String(data: stdoutData, encoding: .utf8)?.trimmingCharacters(in: .whitespacesAndNewlines)
        let stderr = String(data: stderrData, encoding: .utf8)?.trimmingCharacters(in: .whitespacesAndNewlines)

        if process.terminationStatus != 0 {
            return GitGtrResult(output: nil, error: stderr ?? "Process exited with code \(process.terminationStatus)")
        }
        return GitGtrResult(output: stdout, error: nil)
    }

    private func showWorktreeError(title: String, message: String) {
        DispatchQueue.main.async {
            let alert = NSAlert()
            alert.messageText = title
            alert.informativeText = message
            alert.alertStyle = .warning
            alert.addButton(withTitle: String(localized: "worktree.error.ok", defaultValue: "OK"))
            alert.runModal()
        }
    }
```

- [ ] **Step 2: Verify the build compiles**

Run:
```bash
cd /Users/linyaoli/workspace/cmux-adhoc/cmux-adhoc && xcodebuild -project GhosttyTabs.xcodeproj -scheme cmux -configuration Debug -destination 'platform=macOS' -derivedDataPath /tmp/cmux-worktree-btn build 2>&1 | tail -5
```
Expected: `** BUILD SUCCEEDED **`

- [ ] **Step 3: Commit**

```bash
git add Sources/TabManager.swift
git commit -m "feat: add addWorktreeWorkspace method to TabManager

Runs git gtr new/go via Process, creates workspace at worktree path.
Falls back to regular workspace on error with NSAlert."
```

---

### Task 2: Add worktree button and popover to TitlebarControlsView

**Files:**
- Modify: `Sources/Update/UpdateTitlebarAccessory.swift:253-259` (TitlebarControlsView params)
- Modify: `Sources/Update/UpdateTitlebarAccessory.swift:399-409` (plus button area)

- [ ] **Step 1: Add `onNewWorktreeTab` parameter and popover state to TitlebarControlsView**

In `Sources/Update/UpdateTitlebarAccessory.swift`, add the new callback parameter and state. Find:

```swift
    let onNewTab: () -> Void
    let visibilityMode: TitlebarControlsVisibilityMode
```

Replace with:

```swift
    let onNewTab: () -> Void
    let onNewWorktreeTab: (String) -> Void
    let visibilityMode: TitlebarControlsVisibilityMode
```

Then find:

```swift
    @State private var isNotificationsPopoverShown = false
```

Add after it:

```swift
    @State private var isWorktreePopoverShown = false
    @State private var worktreeName = ""
```

- [ ] **Step 2: Add the worktree button with popover after the plus button**

In `Sources/Update/UpdateTitlebarAccessory.swift`, find the plus button block (lines 399-409):

```swift
            TitlebarControlButton(config: config, action: {
                #if DEBUG
                dlog("titlebar.newTab")
                #endif
                onNewTab()
            }) {
                iconLabel(systemName: "plus", config: config)
            }
            .accessibilityIdentifier("titlebarControl.newTab")
            .accessibilityLabel(String(localized: "titlebar.newWorkspace.accessibilityLabel", defaultValue: "New Workspace"))
            .safeHelp(KeyboardShortcutSettings.Action.newTab.tooltip(String(localized: "titlebar.newWorkspace.tooltip", defaultValue: "New workspace")))

        }
```

Replace with:

```swift
            TitlebarControlButton(config: config, action: {
                #if DEBUG
                dlog("titlebar.newTab")
                #endif
                onNewTab()
            }) {
                iconLabel(systemName: "plus", config: config)
            }
            .accessibilityIdentifier("titlebarControl.newTab")
            .accessibilityLabel(String(localized: "titlebar.newWorkspace.accessibilityLabel", defaultValue: "New Workspace"))
            .safeHelp(KeyboardShortcutSettings.Action.newTab.tooltip(String(localized: "titlebar.newWorkspace.tooltip", defaultValue: "New workspace")))

            TitlebarControlButton(config: config, action: {
                #if DEBUG
                dlog("titlebar.newWorktreeTab")
                #endif
                isWorktreePopoverShown = true
            }) {
                iconLabel(systemName: "arrow.triangle.branch", config: config)
            }
            .accessibilityIdentifier("titlebarControl.newWorktreeTab")
            .accessibilityLabel(String(localized: "titlebar.newWorktreeWorkspace.accessibilityLabel", defaultValue: "New Workspace with Worktree"))
            .safeHelp(String(localized: "titlebar.newWorktreeWorkspace.tooltip", defaultValue: "New workspace with worktree"))
            .popover(isPresented: $isWorktreePopoverShown) {
                WorktreeNamePopover(
                    worktreeName: $worktreeName,
                    onSubmit: { name in
                        isWorktreePopoverShown = false
                        onNewWorktreeTab(name)
                        worktreeName = ""
                    },
                    onCancel: {
                        isWorktreePopoverShown = false
                        worktreeName = ""
                    }
                )
            }

        }
```

- [ ] **Step 3: Add the WorktreeNamePopover view**

Add this struct at the end of `Sources/Update/UpdateTitlebarAccessory.swift`, just before the `TitlebarControlsVisibilityMode` enum (before line 568):

```swift
struct WorktreeNamePopover: View {
    @Binding var worktreeName: String
    let onSubmit: (String) -> Void
    let onCancel: () -> Void
    @FocusState private var isTextFieldFocused: Bool

    var body: some View {
        VStack(spacing: 8) {
            TextField(
                String(localized: "worktree.popover.placeholder", defaultValue: "Worktree name (e.g. branch-A)"),
                text: $worktreeName
            )
            .textFieldStyle(.roundedBorder)
            .frame(minWidth: 200)
            .focused($isTextFieldFocused)
            .onSubmit {
                guard !worktreeName.trimmingCharacters(in: .whitespaces).isEmpty else { return }
                onSubmit(worktreeName.trimmingCharacters(in: .whitespaces))
            }

            HStack {
                Spacer()
                Button(String(localized: "worktree.popover.cancel", defaultValue: "Cancel")) {
                    onCancel()
                }
                .keyboardShortcut(.cancelAction)
                Button(String(localized: "worktree.popover.create", defaultValue: "Create")) {
                    onSubmit(worktreeName.trimmingCharacters(in: .whitespaces))
                }
                .keyboardShortcut(.defaultAction)
                .disabled(worktreeName.trimmingCharacters(in: .whitespaces).isEmpty)
            }
        }
        .padding(12)
        .onAppear { isTextFieldFocused = true }
    }
}
```

- [ ] **Step 4: Verify the build compiles**

Run:
```bash
cd /Users/linyaoli/workspace/cmux-adhoc/cmux-adhoc && xcodebuild -project GhosttyTabs.xcodeproj -scheme cmux -configuration Debug -destination 'platform=macOS' -derivedDataPath /tmp/cmux-worktree-btn build 2>&1 | tail -5
```
Expected: Build will FAIL because `TitlebarControlsAccessoryViewController` and `HiddenTitlebarSidebarControlsView` don't pass the new `onNewWorktreeTab` parameter yet. That's expected — we fix it in the next step.

- [ ] **Step 5: Commit**

```bash
git add Sources/Update/UpdateTitlebarAccessory.swift
git commit -m "feat: add worktree button and popover to TitlebarControlsView

Adds arrow.triangle.branch button next to the plus button.
Shows a popover with text field for worktree name on click."
```

---

### Task 3: Wire up the new callback in both titlebar modes

**Files:**
- Modify: `Sources/Update/UpdateTitlebarAccessory.swift:543-566` (HiddenTitlebarSidebarControlsView)
- Modify: `Sources/Update/UpdateTitlebarAccessory.swift:790-804` (TitlebarControlsAccessoryViewController)

- [ ] **Step 1: Wire up HiddenTitlebarSidebarControlsView (minimal/sidebar mode)**

In `Sources/Update/UpdateTitlebarAccessory.swift`, find the `HiddenTitlebarSidebarControlsView` body (around line 550-563):

```swift
        TitlebarControlsView(
            notificationStore: notificationStore,
            viewModel: viewModel,
            onToggleSidebar: { _ = AppDelegate.shared?.sidebarState?.toggle() },
            onToggleNotifications: { [viewModel] in
                AppDelegate.shared?.toggleNotificationsPopover(
                    animated: true,
                    anchorView: viewModel.notificationsAnchorView
                )
            },
            onNewTab: { _ = AppDelegate.shared?.tabManager?.addTab() },
            visibilityMode: .onHover
        )
```

Replace with:

```swift
        TitlebarControlsView(
            notificationStore: notificationStore,
            viewModel: viewModel,
            onToggleSidebar: { _ = AppDelegate.shared?.sidebarState?.toggle() },
            onToggleNotifications: { [viewModel] in
                AppDelegate.shared?.toggleNotificationsPopover(
                    animated: true,
                    anchorView: viewModel.notificationsAnchorView
                )
            },
            onNewTab: { _ = AppDelegate.shared?.tabManager?.addTab() },
            onNewWorktreeTab: { name in _ = AppDelegate.shared?.tabManager?.addWorktreeWorkspace(name: name) },
            visibilityMode: .onHover
        )
```

Also update `hostWidth` to accommodate the new button. Find:

```swift
    private let hostWidth: CGFloat = 124
```

Replace with:

```swift
    private let hostWidth: CGFloat = 152
```

- [ ] **Step 2: Wire up TitlebarControlsAccessoryViewController (standard titlebar mode)**

In `Sources/Update/UpdateTitlebarAccessory.swift`, find the `init` of `TitlebarControlsAccessoryViewController` (around line 790-804):

```swift
        let toggleSidebar = { _ = AppDelegate.shared?.sidebarState?.toggle() }
        let toggleNotifications: () -> Void = { _ = AppDelegate.shared?.toggleNotificationsPopover(animated: true) }
        let newTab = { _ = AppDelegate.shared?.tabManager?.addTab() }
        hostingView = NonDraggableHostingView(
            rootView: TitlebarControlsView(
                notificationStore: notificationStore,
                viewModel: viewModel,
                onToggleSidebar: toggleSidebar,
                onToggleNotifications: toggleNotifications,
                onNewTab: newTab,
                visibilityMode: .alwaysVisible
            )
        )
```

Replace with:

```swift
        let toggleSidebar = { _ = AppDelegate.shared?.sidebarState?.toggle() }
        let toggleNotifications: () -> Void = { _ = AppDelegate.shared?.toggleNotificationsPopover(animated: true) }
        let newTab = { _ = AppDelegate.shared?.tabManager?.addTab() }
        let newWorktreeTab: (String) -> Void = { name in _ = AppDelegate.shared?.tabManager?.addWorktreeWorkspace(name: name) }
        hostingView = NonDraggableHostingView(
            rootView: TitlebarControlsView(
                notificationStore: notificationStore,
                viewModel: viewModel,
                onToggleSidebar: toggleSidebar,
                onToggleNotifications: toggleNotifications,
                onNewTab: newTab,
                onNewWorktreeTab: newWorktreeTab,
                visibilityMode: .alwaysVisible
            )
        )
```

- [ ] **Step 3: Verify the build compiles**

Run:
```bash
cd /Users/linyaoli/workspace/cmux-adhoc/cmux-adhoc && xcodebuild -project GhosttyTabs.xcodeproj -scheme cmux -configuration Debug -destination 'platform=macOS' -derivedDataPath /tmp/cmux-worktree-btn build 2>&1 | tail -5
```
Expected: `** BUILD SUCCEEDED **`

- [ ] **Step 4: Commit**

```bash
git add Sources/Update/UpdateTitlebarAccessory.swift
git commit -m "feat: wire worktree button callback in both titlebar modes

Passes onNewWorktreeTab to TitlebarControlsView from both
HiddenTitlebarSidebarControlsView and TitlebarControlsAccessoryViewController."
```

---

### Task 4: Manual Smoke Test

- [ ] **Step 1: Build and launch a tagged debug app**

```bash
cd /Users/linyaoli/workspace/cmux-adhoc/cmux-adhoc && ./scripts/reload.sh --tag worktree-btn --launch
```

- [ ] **Step 2: Verify the new button appears**

In the running app:
1. Look at the top-left titlebar controls — there should now be a branch icon (`arrow.triangle.branch`) button next to the plus button
2. Click the branch button — a popover should appear with a text field and Create/Cancel buttons
3. Press Escape — popover should dismiss
4. Click the branch button again, type a name, press Enter or click Create
5. If you are in a git repo with `git gtr` installed, a new workspace should open in the worktree directory
6. If `git gtr` is not available, an error alert should appear and a regular workspace should be created as fallback
7. Verify the plus button still works as before (creates a regular workspace immediately)

- [ ] **Step 3: Verify minimal mode**

1. Switch to minimal mode (if available in settings)
2. The branch button should appear in the sidebar overlay next to the plus button
3. Same popover and behavior as in standard mode
