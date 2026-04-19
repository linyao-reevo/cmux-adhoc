# Typing Latency & CPU Optimization Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Reduce per-keystroke main-thread work from O(N tabs) to O(1) when 10+ tabs are open.

**Architecture:** Five targeted optimizations using two strategies: (A) coalesce rapid per-keystroke work into single batched operations, and (B) suppress visually redundant sidebar/notification updates during active typing using a shared `isTypingActive` flag on `Workspace`. All changes are additive guards/debounces — no existing code paths are removed.

**Tech Stack:** Swift, AppKit, Combine, GCD (DispatchWorkItem, DispatchQueue)

**Key types:** `Tab` is a typealias for `Workspace` (TabManager.swift:10). `TabManager` is an `ObservableObject` (TabManager.swift:717). `GhosttyNSView` is the NSView subclass handling keyboard events (GhosttyTerminalView.swift:5192). `NotificationBurstCoalescer` is a reusable coalescing utility (TabManager.swift:513-548).

---

### Task 1: Add `isTypingActive` Flag to Workspace

This is the shared foundation used by Tasks 3 and 5. It must be implemented first.

**Files:**
- Modify: `Sources/Workspace.swift` — add `isTypingActive` property and `markTypingActive()` method near the other published properties (around line 6553)

- [ ] **Step 1: Add the typing guard property and method to Workspace**

Add after line 6561 (`@Published private(set) var terminalScrollBarHidden: Bool = false`) in `Sources/Workspace.swift`:

```swift
    // MARK: - Typing Guard

    /// True while the user is actively typing in this workspace's focused terminal.
    /// NOT @Published — must not trigger Combine propagation. Read as a plain Bool
    /// guard by TabItemView and TabManager to skip redundant work during typing bursts.
    private(set) var isTypingActive: Bool = false
    private var typingResetWorkItem: DispatchWorkItem?

    /// Call from the keystroke hot path. Sets the typing guard and schedules
    /// auto-reset 300ms after the last keystroke.
    func markTypingActive() {
        isTypingActive = true
        typingResetWorkItem?.cancel()
        let item = DispatchWorkItem { [weak self] in
            self?.isTypingActive = false
        }
        typingResetWorkItem = item
        DispatchQueue.main.asyncAfter(deadline: .now() + 0.3, execute: item)
    }
```

- [ ] **Step 2: Wire `markTypingActive()` into the keystroke hot path**

In `Sources/GhosttyTerminalView.swift`, inside `keyDown(with:)` method, add immediately after line 6624 (`if let terminalSurface {`), before the `dismissNotificationOnDirectInteraction` call:

```swift
        if let terminalSurface {
            // Mark workspace typing-active to suppress redundant sidebar/notification work.
            if let tabManager = AppDelegate.shared?.tabManager,
               let tab = tabManager.tabs.first(where: { $0.id == terminalSurface.tabId }) {
                tab.markTypingActive()
            }
```

Note: This lookup is O(N) on tabs. However, `terminalSurface.tabId` is the active tab, so we can optimize with `tabManager.selectedTab` if available. Check if `tabManager.selectedTabId == terminalSurface.tabId` and use a direct reference. If no direct `selectedTab` property exists, the linear scan is acceptable for now since N is typically < 50 and this is a simple identity comparison.

Actually, let's avoid the O(N) scan entirely. The `dismissNotificationOnDirectInteraction` call right below already does `guard selectedTabId == tabId` (TabManager.swift:5170). We can route through TabManager instead:

Replace the block above with a simpler approach — add a method to TabManager:

In `Sources/TabManager.swift`, add after `dismissNotificationOnDirectInteraction` (after line 5185):

```swift
    /// Marks the workspace for the given tab as typing-active.
    /// Called from the keystroke hot path — must be O(1).
    func markTypingActiveForSelectedTab(tabId: UUID) {
        guard selectedTabId == tabId,
              let tab = selectedTab else { return }
        tab.markTypingActive()
    }
```

Then in `Sources/GhosttyTerminalView.swift`, add at line 6625 (inside the `if let terminalSurface` block, before `dismissNotificationOnDirectInteraction`):

```swift
            AppDelegate.shared?.tabManager?.markTypingActiveForSelectedTab(tabId: terminalSurface.tabId)
```

Check if `selectedTab` exists on TabManager. If not, use:
```swift
    func markTypingActiveForSelectedTab(tabId: UUID) {
        guard selectedTabId == tabId,
              let tab = tabs.first(where: { $0.id == tabId }) else { return }
        tab.markTypingActive()
    }
```

- [ ] **Step 3: Add debug logging**

In the `markTypingActive()` method in `Sources/Workspace.swift`, add a dlog for activation (only log when transitioning from false to true to avoid spam):

```swift
    func markTypingActive() {
        let wasActive = isTypingActive
        isTypingActive = true
        typingResetWorkItem?.cancel()
        let item = DispatchWorkItem { [weak self] in
            guard let self else { return }
            self.isTypingActive = false
#if DEBUG
            dlog("typing.guard workspace=\(self.id.uuidString.prefix(5)) deactivated")
#endif
        }
        typingResetWorkItem = item
        DispatchQueue.main.asyncAfter(deadline: .now() + 0.3, execute: item)
#if DEBUG
        if !wasActive {
            dlog("typing.guard workspace=\(id.uuidString.prefix(5)) activated")
        }
#endif
    }
```

- [ ] **Step 4: Verify build compiles**

Run:
```bash
xcodebuild -project GhosttyTabs.xcodeproj -scheme cmux -configuration Debug -destination 'platform=macOS' -derivedDataPath /tmp/cmux-typing-opt build 2>&1 | tail -5
```

Expected: `** BUILD SUCCEEDED **`

- [ ] **Step 5: Commit**

```bash
git add Sources/Workspace.swift Sources/GhosttyTerminalView.swift Sources/TabManager.swift
git commit -m "perf: add isTypingActive guard on Workspace for keystroke hot path"
```

---

### Task 2: Debounce Notification Dismissal on Keystrokes

**Files:**
- Modify: `Sources/GhosttyTerminalView.swift:6624-6634` — replace synchronous call with coalesced debounce

- [ ] **Step 1: Add a debounce work item property to GhosttyNSView**

In `Sources/GhosttyTerminalView.swift`, add near the other `DispatchWorkItem?` properties (around line 1552, near `scrollEndTimer`):

```swift
    /// Coalesces notification dismissal during rapid typing.
    /// Reset on each keystroke; fires 100ms after the last key.
    private var notificationDismissWorkItem: DispatchWorkItem?
```

- [ ] **Step 2: Replace synchronous dismissal with debounced version**

In `Sources/GhosttyTerminalView.swift`, replace the notification dismissal block at lines 6624-6635. The current code is:

```swift
        if let terminalSurface {
#if DEBUG
            let dismissNotificationStart = ProcessInfo.processInfo.systemUptime
#endif
            AppDelegate.shared?.tabManager?.dismissNotificationOnDirectInteraction(
                tabId: terminalSurface.tabId,
                surfaceId: terminalSurface.id
            )
#if DEBUG
            dismissNotificationMs = (ProcessInfo.processInfo.systemUptime - dismissNotificationStart) * 1000.0
#endif
        }
```

Replace with:

```swift
        if let terminalSurface {
#if DEBUG
            let dismissNotificationStart = ProcessInfo.processInfo.systemUptime
#endif
            // Coalesce rapid-typing notification dismissals into one call per burst.
            // The work item resets on each keystroke and fires 100ms after the last key.
            notificationDismissWorkItem?.cancel()
            let tabId = terminalSurface.tabId
            let surfaceId = terminalSurface.id
            let item = DispatchWorkItem { [weak self] in
                guard self != nil else { return }
                AppDelegate.shared?.tabManager?.dismissNotificationOnDirectInteraction(
                    tabId: tabId,
                    surfaceId: surfaceId
                )
            }
            notificationDismissWorkItem = item
            DispatchQueue.main.asyncAfter(deadline: .now() + 0.1, execute: item)
#if DEBUG
            dismissNotificationMs = (ProcessInfo.processInfo.systemUptime - dismissNotificationStart) * 1000.0
#endif
        }
```

Note: The DEBUG timing now measures the scheduling cost (near-zero) rather than the dismissal itself. This is intentional — the hot path cost is what matters for typing latency.

- [ ] **Step 3: Verify build compiles**

Run:
```bash
xcodebuild -project GhosttyTabs.xcodeproj -scheme cmux -configuration Debug -destination 'platform=macOS' -derivedDataPath /tmp/cmux-typing-opt build 2>&1 | tail -5
```

Expected: `** BUILD SUCCEEDED **`

- [ ] **Step 4: Commit**

```bash
git add Sources/GhosttyTerminalView.swift
git commit -m "perf: debounce notification dismissal during typing (100ms coalesce)"
```

---

### Task 3: Suppress Sidebar Activity Propagation for Active Workspace During Typing

**Files:**
- Modify: `Sources/ContentView.swift:13342-13358` — add typing guard in `.onReceive` for `sidebarActivityPublisher`

- [ ] **Step 1: Add typing guard to the sidebarActivityPublisher handler**

In `Sources/ContentView.swift`, the `.onReceive` handler for `tab.sidebarActivityPublisher` at line 13342 currently looks like:

```swift
        .onReceive(
            tab.sidebarActivityPublisher
                .receive(on: RunLoop.main)
                .debounce(for: .milliseconds(500), scheduler: RunLoop.main)
        ) { _ in
#if DEBUG
            let description = tab.customDescription ?? ""
            dlog(
                "sidebar.row.invalidate workspace=\(tab.id.uuidString.prefix(8)) " +
                "source=activity " +
                "title=\"\(debugCommandPaletteTextPreview(tab.title))\" " +
                "descLen=\((description as NSString).length) " +
                "desc=\"\(debugCommandPaletteTextPreview(description))\""
            )
#endif
            scheduleWorkspaceObservationInvalidation()
        }
```

Replace the body of the closure (after the `.onReceive(`) with:

```swift
        .onReceive(
            tab.sidebarActivityPublisher
                .receive(on: RunLoop.main)
                .debounce(for: .milliseconds(500), scheduler: RunLoop.main)
        ) { _ in
            // Skip sidebar re-render for the active workspace while the user is typing.
            // The 500ms debounce + 300ms typing guard ensure the sidebar catches up
            // after the typing burst ends.
            if tab.isTypingActive && isActive {
#if DEBUG
                dlog(
                    "sidebar.row.invalidate.SKIPPED workspace=\(tab.id.uuidString.prefix(8)) " +
                    "source=activity reason=typingActive"
                )
#endif
                return
            }
#if DEBUG
            let description = tab.customDescription ?? ""
            dlog(
                "sidebar.row.invalidate workspace=\(tab.id.uuidString.prefix(8)) " +
                "source=activity " +
                "title=\"\(debugCommandPaletteTextPreview(tab.title))\" " +
                "descLen=\((description as NSString).length) " +
                "desc=\"\(debugCommandPaletteTextPreview(description))\""
            )
#endif
            scheduleWorkspaceObservationInvalidation()
        }
```

The `isActive` property is already available on `TabItemView` (line 12663) — it's true when `tab.id == selectedTabId`. This ensures only the actively-typed-in workspace's sidebar row is suppressed; all other workspace rows update normally.

- [ ] **Step 2: Verify build compiles**

Run:
```bash
xcodebuild -project GhosttyTabs.xcodeproj -scheme cmux -configuration Debug -destination 'platform=macOS' -derivedDataPath /tmp/cmux-typing-opt build 2>&1 | tail -5
```

Expected: `** BUILD SUCCEEDED **`

- [ ] **Step 3: Commit**

```bash
git add Sources/ContentView.swift
git commit -m "perf: suppress sidebar activity re-render for active workspace during typing"
```

---

### Task 4: Coalesce Shell Activity State Updates Off Main Thread

**Files:**
- Modify: `Sources/TerminalController.swift` — add coalescing to the `reportShellState` handler's main-thread dispatch

- [ ] **Step 1: Add a shell activity coalescer to SocketFastPathState**

The existing `SocketFastPathState` class (TerminalController.swift:431) already handles deduplication on its private dispatch queue. Extend it with a coalescing flush mechanism.

Add a new method and storage to `SocketFastPathState` (after the existing `shouldPublishShellActivity` method, around line 468):

```swift
        // MARK: - Shell Activity Coalescing

        private var pendingShellActivityUpdates: [SocketSurfaceKey: Workspace.PanelShellActivityState] = [:]
        private var shellActivityFlushScheduled = false
        private static let shellActivityFlushDelay: TimeInterval = 0.05  // 50ms

        /// Enqueue a shell activity update for coalesced delivery to the main thread.
        /// Multiple updates for the same surface within the flush window are last-write-wins.
        func enqueueShellActivityUpdate(
            workspaceId: UUID,
            panelId: UUID,
            state: Workspace.PanelShellActivityState
        ) {
            let key = SocketSurfaceKey(workspaceId: workspaceId, panelId: panelId)
            queue.async { [self] in
                guard lastReportedShellStates[key] != state else { return }
                if lastReportedShellStates.count >= maxTrackedShellStates {
                    lastReportedShellStates.removeAll(keepingCapacity: true)
                }
                lastReportedShellStates[key] = state
                pendingShellActivityUpdates[key] = state
                guard !shellActivityFlushScheduled else { return }
                shellActivityFlushScheduled = true
                DispatchQueue.main.asyncAfter(deadline: .now() + Self.shellActivityFlushDelay) { [self] in
                    self.flushShellActivityUpdates()
                }
            }
        }

        private func flushShellActivityUpdates() {
            var updates: [SocketSurfaceKey: Workspace.PanelShellActivityState] = [:]
            queue.sync {
                updates = pendingShellActivityUpdates
                pendingShellActivityUpdates.removeAll(keepingCapacity: true)
                shellActivityFlushScheduled = false
            }
            guard !updates.isEmpty else { return }
    #if DEBUG
            if updates.count > 1 {
                dlog("shellActivity.coalesce flushed=\(updates.count) updates")
            }
    #endif
            for (key, state) in updates {
                guard let tabManager = AppDelegate.shared?.tabManagerFor(tabId: key.workspaceId) else { continue }
                tabManager.updateSurfaceShellActivity(tabId: key.workspaceId, surfaceId: key.panelId, state: state)
            }
        }
```

Note: `SocketSurfaceKey` has `workspaceId` and `panelId` properties (TerminalController.swift:426). Verify these are accessible (they may be stored with different names). Check the struct definition:

```swift
    private struct SocketSurfaceKey: Hashable {
        let workspaceId: UUID
        let panelId: UUID
    }
```

If the properties are named differently, adjust accordingly.

- [ ] **Step 2: Route the explicit-scope fast path through the coalescer**

In the `reportShellState` method (TerminalController.swift:15353), replace the explicit-scope branch's `DispatchQueue.main.async` block. The current code at lines 15363-15372:

```swift
            guard Self.socketFastPathState.shouldPublishShellActivity(
                workspaceId: scope.workspaceId,
                panelId: scope.panelId,
                state: state
            ) else {
                return "OK"
            }
            DispatchQueue.main.async {
                guard let tabManager = AppDelegate.shared?.tabManagerFor(tabId: scope.workspaceId) else { return }
                tabManager.updateSurfaceShellActivity(tabId: scope.workspaceId, surfaceId: scope.panelId, state: state)
            }
```

Replace with:

```swift
            Self.socketFastPathState.enqueueShellActivityUpdate(
                workspaceId: scope.workspaceId,
                panelId: scope.panelId,
                state: state
            )
```

The `shouldPublishShellActivity` check is now done inside `enqueueShellActivityUpdate` (same dedup logic), so the separate guard is no longer needed.

- [ ] **Step 3: Route the legacy (non-explicit-scope) path through the coalescer**

The legacy path at lines 15381-15417 uses `DispatchQueue.main.sync` and does additional work (resolve tab, prune surface metadata). This path cannot be fully coalesced because it returns error messages synchronously. Leave this path as-is — it's only hit by v1 socket commands which don't provide explicit `--tab`/`--panel` flags. The fast path (v2 with explicit scope) is the one that fires on every keystroke.

- [ ] **Step 4: Verify build compiles**

Run:
```bash
xcodebuild -project GhosttyTabs.xcodeproj -scheme cmux -configuration Debug -destination 'platform=macOS' -derivedDataPath /tmp/cmux-typing-opt build 2>&1 | tail -5
```

Expected: `** BUILD SUCCEEDED **`

- [ ] **Step 5: Commit**

```bash
git add Sources/TerminalController.swift
git commit -m "perf: coalesce shell activity state updates off main thread (50ms window)"
```

---

### Task 5: Skip Notification Dismiss Flash During Typing

**Files:**
- Modify: `Sources/TabManager.swift:5180-5183` — add `isTypingActive` guard before flash

- [ ] **Step 1: Add the typing guard before the flash call**

In `Sources/TabManager.swift`, the current code at lines 5180-5183:

```swift
        if let panelId = surfaceId,
           let tab = tabs.first(where: { $0.id == tabId }) {
            tab.triggerNotificationDismissFlash(panelId: panelId)
        }
```

Replace with:

```swift
        if let panelId = surfaceId,
           let tab = tabs.first(where: { $0.id == tabId }) {
            // Skip the visual flash animation during active typing — the user
            // is already interacting with the panel, so the flash is redundant.
            if !tab.isTypingActive {
                tab.triggerNotificationDismissFlash(panelId: panelId)
            }
        }
```

- [ ] **Step 2: Verify build compiles**

Run:
```bash
xcodebuild -project GhosttyTabs.xcodeproj -scheme cmux -configuration Debug -destination 'platform=macOS' -derivedDataPath /tmp/cmux-typing-opt build 2>&1 | tail -5
```

Expected: `** BUILD SUCCEEDED **`

- [ ] **Step 3: Commit**

```bash
git add Sources/TabManager.swift
git commit -m "perf: skip notification dismiss flash animation during active typing"
```

---

### Task 6: Target syncBonsplitNotificationBadges to Affected Panel

**Files:**
- Modify: `Sources/WorkspaceContentView.swift:348-353, 390-416` — add targeted variant, update onChange handler

- [ ] **Step 1: Add the targeted badge update method**

In `Sources/WorkspaceContentView.swift`, add after the existing `syncBonsplitNotificationBadges()` method (after line 416):

```swift
    /// Update bonsplit notification badge for a single panel.
    /// O(1) per panel vs O(all panes × all tabs) for the full sweep.
    private func syncBonsplitNotificationBadge(forPanelId panelId: UUID) {
        let manualUnread = workspace.manualUnreadPanelIds
        let shouldShow = notificationStore.hasVisibleNotificationIndicator(
            forTabId: workspace.id, surfaceId: panelId
        ) || manualUnread.contains(panelId)

        // Find the bonsplit tab for this panel by scanning surfaces.
        // panelIdFromSurfaceId maps surface→panel, so we need the reverse:
        // find the surface whose panel ID matches.
        for paneId in workspace.bonsplitController.allPaneIds {
            for tab in workspace.bonsplitController.tabs(inPane: paneId) {
                guard workspace.panelIdFromSurfaceId(tab.id) == panelId else { continue }
                let expectedKind = workspace.panelKind(panelId: panelId)
                let expectedPinned = workspace.isPanelPinned(panelId)
                let kindUpdate: String?? = expectedKind.map { .some($0) }

                if tab.showsNotificationBadge != shouldShow ||
                    tab.isPinned != expectedPinned ||
                    (expectedKind != nil && tab.kind != expectedKind) {
                    workspace.bonsplitController.updateTab(
                        tab.id,
                        kind: kindUpdate,
                        showsNotificationBadge: shouldShow,
                        isPinned: expectedPinned
                    )
                }
                return  // Found the tab, done.
            }
        }
    }
```

- [ ] **Step 2: Update the onChange handler to diff and target**

Replace the `.onChange(of: notificationStore.notifications)` handler at line 348:

Current:
```swift
        .onChange(of: notificationStore.notifications) { _, _ in
            syncBonsplitNotificationBadges()
        }
```

Replace with:
```swift
        .onChange(of: notificationStore.notifications) { oldValue, newValue in
            // Diff old/new to find which panel IDs changed, then update only those.
            let oldPanelIds = Set(oldValue.map(\.surfaceId).compactMap { $0 })
            let newPanelIds = Set(newValue.map(\.surfaceId).compactMap { $0 })
            let changedPanelIds = oldPanelIds.symmetricDifference(newPanelIds)

            if changedPanelIds.isEmpty || changedPanelIds.count > 5 {
                // No specific diff available or too many changes — fall back to full sweep.
                syncBonsplitNotificationBadges()
            } else {
                for panelId in changedPanelIds {
                    syncBonsplitNotificationBadge(forPanelId: panelId)
                }
            }
        }
```

Note: The `surfaceId` property on notifications must exist. Verify the notification type has a `surfaceId: UUID?` field. If the field is named differently (e.g., `panelId`), adjust accordingly. If notifications don't carry a surface/panel ID, fall back to the full sweep approach and instead just add an early-return guard:

```swift
        .onChange(of: notificationStore.notifications) { _, _ in
            // During active typing, the debounced dismissal handles badge updates.
            // Skip the full sweep until typing stops.
            if workspace.isTypingActive { return }
            syncBonsplitNotificationBadges()
        }
```

Use whichever approach works based on the notification type's available fields.

- [ ] **Step 3: Verify build compiles**

Run:
```bash
xcodebuild -project GhosttyTabs.xcodeproj -scheme cmux -configuration Debug -destination 'platform=macOS' -derivedDataPath /tmp/cmux-typing-opt build 2>&1 | tail -5
```

Expected: `** BUILD SUCCEEDED **`

- [ ] **Step 4: Commit**

```bash
git add Sources/WorkspaceContentView.swift
git commit -m "perf: target bonsplit notification badge updates to affected panel"
```

---

### Task 7: Build and Verify

**Files:** None (verification only)

- [ ] **Step 1: Clean build to verify all changes compile together**

```bash
xcodebuild -project GhosttyTabs.xcodeproj -scheme cmux -configuration Debug -destination 'platform=macOS' -derivedDataPath /tmp/cmux-typing-opt clean build 2>&1 | tail -10
```

Expected: `** BUILD SUCCEEDED **`

- [ ] **Step 2: Build with reload.sh for manual testing**

```bash
./scripts/reload.sh --tag typing-opt
```

Expected: Build succeeds, prints app path.

- [ ] **Step 3: Verify debug logging**

After launching the tagged app, open 10+ workspaces, then type rapidly in one terminal. Tail the debug log:

```bash
tail -f /tmp/cmux-debug-typing-opt.log | grep -E "typing\.guard|sidebar\.row\.invalidate|shellActivity\.coalesce"
```

Expected output pattern:
- `typing.guard workspace=XXXXX activated` — once at start of typing
- `sidebar.row.invalidate.SKIPPED workspace=XXXXX source=activity reason=typingActive` — during typing
- `shellActivity.coalesce flushed=N updates` — when multiple shell states coalesced
- `typing.guard workspace=XXXXX deactivated` — 300ms after last keystroke
- `sidebar.row.invalidate workspace=XXXXX source=activity` — normal update after typing stops
