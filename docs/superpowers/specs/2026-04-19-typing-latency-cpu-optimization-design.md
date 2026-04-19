# Typing Latency & CPU/Memory Optimization Design

**Date:** 2026-04-19
**Approach:** Hybrid — Targeted Updates + Typing Suppression
**Goal:** Reduce per-keystroke main-thread work from O(N tabs) to O(1) when 10+ tabs are open.

## Problem Statement

With 10+ tabs/workspaces open, each keystroke triggers a cascade of O(N) main-thread work:

1. `dismissNotificationOnDirectInteraction` fires on every key → O(n) notification index rebuild
2. `syncBonsplitNotificationBadges` sweeps all panes × all tabs on notification change
3. Shell integration `report_shell_state` → updates `panelShellActivityStates` (@Published) → `sidebarActivityPublisher` fires → all N `TabItemView` instances re-render via `workspaceObservationGeneration` bump
4. Each `report_shell_state` dispatches individually to main thread (1 dispatch per keystroke)
5. `triggerNotificationDismissFlash` fires animation + @Published mutation on every keystroke dismissal

## Architecture

The optimizations follow two complementary strategies:

- **Strategy A (Coalesce):** Debounce/coalesce work that fires per-keystroke into single batched operations
- **Strategy B (Suppress):** Skip work that is visually redundant during active typing, catch up after typing stops

Both strategies use a shared `isTypingActive` flag on `Workspace` as the typing guard.

## Optimization 1: Debounce Notification Dismissal on Keystrokes

**File:** `Sources/GhosttyTerminalView.swift:6628`
**Strategy:** A (Coalesce)

### Current Behavior

Every keystroke calls:
```swift
AppDelegate.shared?.tabManager?.dismissNotificationOnDirectInteraction(
    tabId: terminalSurface.tabId,
    surfaceId: terminalSurface.id
)
```

This calls `markRead()` → rebuilds notification indexes O(n), `clearFocusedReadIndicator()` → publishes @Published change, and `triggerNotificationDismissFlash()` → fires animation. Even when there's nothing to dismiss, it does two hash lookups per keystroke.

### Fix

Add a coalescing debounce (100ms trailing edge) so rapid keystrokes produce at most one dismissal call per burst. The first keystroke starts a timer; subsequent keystrokes reset it. The dismissal fires 100ms after the last keystroke.

Wrap the call at line 6628 in a coalescing `DispatchWorkItem`, keyed by (tabId, surfaceId).

## Optimization 2: Target syncBonsplitNotificationBadges to Affected Tab Only

**File:** `Sources/WorkspaceContentView.swift:390`
**Strategy:** A (Coalesce)

### Current Behavior

`.onChange(of: notificationStore.notifications)` at line 348 calls `syncBonsplitNotificationBadges()` which iterates all panes × all tabs per pane to check badge state — O(N) sweep even though only one panel's badge changed.

### Fix

Add a targeted variant `syncBonsplitNotificationBadge(forPanelId:)` that updates only the single affected tab in bonsplit. The `.onChange(of: notificationStore.notifications)` handler will diff the old/new values to identify which panel IDs changed and call the targeted variant for each.

The existing full sweep remains for `onAppear` and edge cases (workspace structure changes).

## Optimization 3: Suppress Sidebar Activity Propagation for Active Workspace During Typing

**Files:**
- `Sources/Workspace.swift` — add `isTypingActive` property
- `Sources/GhosttyTerminalView.swift` — set the flag on keystroke
- `Sources/ContentView.swift:13342` — add guard in `.onReceive`

**Strategy:** B (Suppress)

### Current Behavior

Keystroke → shell integration fires `report_shell_state` → updates `workspace.panelShellActivityStates` (@Published) → `sidebarActivityPublisher` fires → all N `TabItemView` instances receive `.onReceive` (line 13342) → each calls `scheduleWorkspaceObservationInvalidation()` → bumps `workspaceObservationGeneration` → sidebar re-renders ALL N tab rows.

### Fix

Add a lightweight typing guard on `Workspace`:
- When the focused terminal receives keystrokes, set `isTypingActive = true` with a 300ms auto-reset timer (trailing edge)
- In `sidebarActivityPublisher`'s subscriber inside `TabItemView` (line 13342), skip `scheduleWorkspaceObservationInvalidation()` if `tab.id == selectedTabId && tab.isTypingActive`
- The sidebar row for the active workspace updates 300ms after typing stops (the debounce naturally catches up)
- Non-active workspace rows are unaffected — they still update on the 500ms debounce as today

### isTypingActive Design

```swift
// Workspace.swift
private var typingResetWorkItem: DispatchWorkItem?

var isTypingActive: Bool = false

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

`isTypingActive` is intentionally NOT @Published — it should not trigger any Combine propagation. It is read as a plain Bool guard by the TabItemView `.onReceive` handler.

## Optimization 4: Coalesce Shell Activity State Updates Off Main Thread

**File:** `Sources/TerminalController.swift:15365`
**Strategy:** A (Coalesce)

### Current Behavior

`report_shell_state` socket commands arrive on a background thread and immediately dispatch to main via `DispatchQueue.main.async` to call `updateSurfaceShellActivity()`. During rapid typing, each keystroke can generate a shell state update, flooding the main thread with individual dispatches.

### Fix

Coalesce shell activity updates before hitting main thread:
- Accumulate shell state reports in a thread-safe dictionary off-main (keyed by surfaceId, last-write-wins)
- Flush to main thread on a 50ms coalescing timer
- A burst of 20 keystrokes that each fire `report_shell_state` produces one main-thread update instead of 20

Modeled after the existing `panelTitleUpdateCoalescer` pattern at `TabManager.swift:5192`.

Follows the socket command threading policy: parse/validate off-main, coalesce off-main, schedule minimal UI mutation with `DispatchQueue.main.async` only when needed.

## Optimization 5: Skip Redundant Notification Flash Animation During Typing

**File:** `Sources/TabManager.swift:5182`
**Strategy:** B (Suppress)

### Current Behavior

`triggerNotificationDismissFlash(panelId:)` calls `requestAttentionFlash()` which updates @Published properties to trigger a visual flash animation. During active typing, the user is already interacting with the panel — a dismiss flash is visually meaningless.

### Fix

Guard the flash — skip `triggerNotificationDismissFlash` when `isTypingActive` is true on the workspace. The flash still fires for non-typing dismissals (focus changes, explicit clicks, programmatic dismissal).

One-line guard using the `isTypingActive` flag from Optimization 3:
```swift
guard !tab.isTypingActive else { return }
```

Added before `tab.triggerNotificationDismissFlash(panelId: panelId)` at `TabManager.swift:5182`.

## Expected Impact

| Optimization | Hot Path | Before | After |
|---|---|---|---|
| Debounce notification dismissal | Every keystroke | O(n) index rebuild per key | 1 rebuild per typing burst |
| Target bonsplit badge sync | Notification change | O(all panes × all tabs) sweep | O(1) single tab update |
| Suppress sidebar activity for active workspace | Shell state → sidebar | N TabItemView re-renders per update | 0 during typing, catch-up after 300ms |
| Coalesce shell activity off-main | Socket → main thread | 1 main dispatch per keystroke | 1 main dispatch per 50ms window |
| Skip dismiss flash during typing | Every keystroke | Animation + @Published mutation | No-op |

**Net effect with 15 tabs open:** A single keystroke goes from triggering ~15 sidebar row re-renders + O(n) notification index rebuild + O(panes×tabs) badge sweep + flash animation → effectively O(1) main-thread work. Sidebar catches up 300ms after typing stops.

## Risk Assessment

**Risk:** Low.

- All changes are additive guards/debounces — no existing code paths are removed
- Existing behavior is fully preserved for non-typing interactions
- The `isTypingActive` flag auto-resets after 300ms, so no stuck state is possible
- The coalescing timers (50ms shell activity, 100ms notification dismissal) are short enough that users won't perceive delayed updates
- The 300ms sidebar suppression window aligns with typical typing burst duration

## Testing Strategy

Behavioral verification through the existing debug event log (`dlog`):
- Add dlog entries for typing guard activations/deactivations
- Add dlog entries for coalescer flush counts (how many coalesced into one)
- Verify via `tail -f` on the debug log during typing with 10+ tabs

No meaningful unit test can be written for these optimizations — they are timing/scheduling behaviors best verified through instrumentation and manual observation. Per test quality policy, skip fake regression tests.
