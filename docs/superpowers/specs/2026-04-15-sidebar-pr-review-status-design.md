# Sidebar PR Review Status

Display GitHub pull request review status alongside the PR lifecycle state in the sidebar.

## Goal

Open PRs in the sidebar currently show only their lifecycle state (`open`, `merged`, `closed`). This change adds review status so users can see at a glance whether a PR has been approved or has changes requested, without leaving the terminal.

## Display Format

The status text for open PRs gains an optional suffix:

| Review Decision | Sidebar Text |
|---|---|
| Approved | `open · approved` |
| Changes requested | `open · changes requested` |
| No decision / pending / commented / dismissed / fetch failure | `open` |

Merged and closed PRs never show a review suffix. If the review status fetch fails or returns no actionable decision, the display falls back to `open`.

## Data Model

### New enum (`Workspace.swift`)

```swift
enum SidebarPullRequestReviewStatus: String {
    case approved
    case changesRequested = "changes_requested"
}
```

### Updated struct (`Workspace.swift`)

Add one optional field to `SidebarPullRequestState`:

```swift
struct SidebarPullRequestState {
    let number: Int
    let label: String
    let url: URL
    let status: SidebarPullRequestStatus
    let branch: String?
    let isStale: Bool
    let reviewStatus: SidebarPullRequestReviewStatus?  // nil = no actionable decision
}
```

## Data Sources

### 1. Shell integration (`cmux-zsh-integration.zsh`)

The existing `_cmux_report_pr_for_path` function calls `gh pr view --json number,state,url`.

Changes:
- Add `reviewDecision` to the `--json` fields and `--jq` template.
- Map the GitHub GraphQL `reviewDecision` value:
  - `APPROVED` -> `--review=approved`
  - `CHANGES_REQUESTED` -> `--review=changes_requested`
  - Any other value or empty -> omit `--review` flag
- This runs on directory change, providing immediate data.

### 2. Background poller (`TabManager.swift`)

The existing poller fetches `GET /repos/{owner}/{repo}/pulls?state=all` in bulk. The REST list endpoint does not include review decisions.

Changes:
- After the bulk fetch resolves which PRs are open and visible in the sidebar, make one additional REST call per open PR: `GET /repos/{owner}/{repo}/pulls/{number}/reviews`.
- Only fetch reviews for open PRs (not merged/closed).
- Compute the aggregate review status:
  1. Parse the reviews array from the response.
  2. For each reviewer, take their most recent review.
  3. If any reviewer's latest review is `CHANGES_REQUESTED`, the aggregate is `changesRequested`.
  4. If all reviewers' latest reviews are `APPROVED` (and at least one exists), the aggregate is `approved`.
  5. Otherwise, the aggregate is `nil` (no actionable decision).
- Reuse existing auth header, timeout, error handling, and cache infrastructure.
- On transient failure, preserve the last known review status (same pattern as existing stale handling).
- Respect the existing polling intervals: `selectedPollInterval` for focused panels, `backgroundPollInterval` for others. Reviews are fetched as part of the same poll cycle, not on a separate timer.

### 3. Socket command (`TerminalController.swift`)

Update the `report_pr` command to accept an optional `--review` flag:

```
report_pr <number> <url> [--label=PR] [--state=open|merged|closed] [--review=approved|changes_requested] [--branch=<name>] [--tab=X] [--panel=Y]
```

- `--review` is optional. When omitted, `reviewStatus` is `nil`.
- Invalid `--review` values produce an error response.
- `--review` is ignored when `--state` is `merged` or `closed`.

## UI Changes (`ContentView.swift`)

### Status label

Update `pullRequestStatusLabel()`:

```swift
private func pullRequestStatusLabel(_ status: SidebarPullRequestStatus, reviewStatus: SidebarPullRequestReviewStatus?) -> String {
    switch status {
    case .open:
        switch reviewStatus {
        case .approved:
            return String(localized: "sidebar.pullRequest.statusOpenApproved", defaultValue: "open \u{00B7} approved")
        case .changesRequested:
            return String(localized: "sidebar.pullRequest.statusOpenChangesRequested", defaultValue: "open \u{00B7} changes requested")
        case nil:
            return String(localized: "sidebar.pullRequest.statusOpen", defaultValue: "open")
        }
    case .merged:
        return String(localized: "sidebar.pullRequest.statusMerged", defaultValue: "merged")
    case .closed:
        return String(localized: "sidebar.pullRequest.statusClosed", defaultValue: "closed")
    }
}
```

### No icon or color changes

The existing `PullRequestStatusIcon` and `pullRequestForegroundColor` remain unchanged. The review status is conveyed through the text suffix only.

## Localization

New localization keys in `Resources/Localizable.xcstrings`:

- `sidebar.pullRequest.statusOpenApproved` (en: `"open \u{00B7} approved"`, ja: TBD)
- `sidebar.pullRequest.statusOpenChangesRequested` (en: `"open \u{00B7} changes requested"`, ja: TBD)

## Files Changed

| File | Change |
|---|---|
| `Sources/Workspace.swift` | Add `SidebarPullRequestReviewStatus` enum, add `reviewStatus` field to `SidebarPullRequestState` |
| `Sources/TerminalController.swift` | Parse `--review` option in `reportPullRequest()` |
| `Sources/TabManager.swift` | Fetch `/pulls/{number}/reviews` for open PRs, compute aggregate, pass to workspace |
| `Sources/ContentView.swift` | Update `pullRequestStatusLabel()` to include review suffix |
| `Resources/shell-integration/cmux-zsh-integration.zsh` | Add `reviewDecision` to `gh pr view` JSON fields, emit `--review` flag |
| `Resources/Localizable.xcstrings` | Add two new localization keys |

## Non-Goals

- CI/check status display (explicitly excluded).
- Draft PR state display.
- Review status for merged/closed PRs.
- Icon or color changes based on review status.
- Separate polling timer for reviews (piggybacks on existing PR poll cycle).
