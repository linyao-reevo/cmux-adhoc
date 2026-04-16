# Sidebar PR Review Status Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Display GitHub PR review status (approved, changes requested) alongside the PR lifecycle state in the sidebar.

**Architecture:** Add `SidebarPullRequestReviewStatus` enum and thread it through the existing data pipeline: shell integration emits `--review` on the `report_pr` socket command, `TerminalController` parses it, `Workspace` stores it, background poller fetches `/pulls/{number}/reviews` for open PRs, and `ContentView` renders the suffix text.

**Tech Stack:** Swift, SwiftUI, zsh, GitHub REST API

---

### Task 1: Add Data Model (`Workspace.swift`)

**Files:**
- Modify: `Sources/Workspace.swift:6038-6072` (enum + struct area)
- Modify: `Sources/Workspace.swift:7655-7697` (`updatePanelPullRequest`)

- [ ] **Step 1: Add `SidebarPullRequestReviewStatus` enum**

Insert immediately after the `SidebarPullRequestStatus` enum (after line 6042):

```swift
enum SidebarPullRequestReviewStatus: String {
    case approved
    case changesRequested = "changes_requested"
}
```

- [ ] **Step 2: Add `reviewStatus` field to `SidebarPullRequestState`**

Update the struct at line 6050 to include `reviewStatus`:

```swift
struct SidebarPullRequestState: Equatable {
    let number: Int
    let label: String
    let url: URL
    let status: SidebarPullRequestStatus
    let branch: String?
    let isStale: Bool
    let reviewStatus: SidebarPullRequestReviewStatus?

    init(
        number: Int,
        label: String,
        url: URL,
        status: SidebarPullRequestStatus,
        branch: String? = nil,
        isStale: Bool = false,
        reviewStatus: SidebarPullRequestReviewStatus? = nil
    ) {
        self.number = number
        self.label = label
        self.url = url
        self.status = status
        self.branch = normalizedSidebarBranchName(branch)
        self.isStale = isStale
        self.reviewStatus = reviewStatus
    }
}
```

- [ ] **Step 3: Update `updatePanelPullRequest` to accept `reviewStatus`**

At `Workspace.swift:7655`, add the parameter and thread it through:

```swift
func updatePanelPullRequest(
    panelId: UUID,
    number: Int,
    label: String,
    url: URL,
    status: SidebarPullRequestStatus,
    branch: String? = nil,
    isStale: Bool = false,
    reviewStatus: SidebarPullRequestReviewStatus? = nil
) {
    let existing = panelPullRequests[panelId]
    let normalizedBranch = normalizedSidebarBranchName(branch)
    let currentPanelBranch = normalizedSidebarBranchName(panelGitBranches[panelId]?.branch)
    let resolvedBranch: String? = {
        if let normalizedBranch {
            return normalizedBranch
        }
        if let currentPanelBranch {
            return currentPanelBranch
        }
        guard let existing,
              existing.number == number,
              existing.label == label,
              existing.url == url,
              existing.status == status else {
            return nil
        }
        return existing.branch
    }()
    let state = SidebarPullRequestState(
        number: number,
        label: label,
        url: url,
        status: status,
        branch: resolvedBranch,
        isStale: isStale,
        reviewStatus: reviewStatus
    )
    if existing != state {
        panelPullRequests[panelId] = state
    }
    if panelId == focusedPanelId, pullRequest != state {
        pullRequest = state
    }
}
```

- [ ] **Step 4: Build to verify compilation**

```bash
xcodebuild -project GhosttyTabs.xcodeproj -scheme cmux -configuration Debug -destination 'platform=macOS' -derivedDataPath /tmp/cmux-pr-review build 2>&1 | tail -5
```

Expected: BUILD SUCCEEDED (the new field has a default value, so all existing call sites compile unchanged).

- [ ] **Step 5: Commit**

```bash
git add Sources/Workspace.swift
git commit -m "feat: add SidebarPullRequestReviewStatus model and reviewStatus field"
```

---

### Task 2: Parse `--review` in Socket Command (`TerminalController.swift`)

**Files:**
- Modify: `Sources/TerminalController.swift:15114-15175` (`reportPullRequest`)
- Modify: `Sources/TerminalController.swift:386-411` (`shouldReplacePullRequest`)

- [ ] **Step 1: Parse `--review` option in `reportPullRequest`**

In `reportPullRequest` at line 15137 (after the `branch` line, before the `checks` guard), add review parsing:

```swift
let reviewStatusRaw = normalizedOptionValue(parsed.options["review"])
let reviewStatus: SidebarPullRequestReviewStatus?
if let reviewStatusRaw {
    guard let parsed = SidebarPullRequestReviewStatus(rawValue: reviewStatusRaw.lowercased()) else {
        return "ERROR: Invalid review status '\(reviewStatusRaw)' — use: approved, changes_requested"
    }
    reviewStatus = status == .open ? parsed : nil
} else {
    reviewStatus = nil
}
```

- [ ] **Step 2: Update usage strings**

Update the three usage strings in `reportPullRequest` (lines 15117, 15144, 15153) from:

```
report_pr <number> <url> [--label=PR] [--state=open|merged|closed] [--branch=<name>] [--tab=X] [--panel=Y]
```

to:

```
report_pr <number> <url> [--label=PR] [--state=open|merged|closed] [--review=approved|changes_requested] [--branch=<name>] [--tab=X] [--panel=Y]
```

- [ ] **Step 3: Thread `reviewStatus` into `shouldReplacePullRequest` and `updatePanelPullRequest`**

Update the `shouldReplacePullRequest` call at line 15155 to include `reviewStatus`:

```swift
guard Self.shouldReplacePullRequest(
    current: tab.panelPullRequests[surfaceId],
    number: number,
    label: label,
    url: url,
    status: status,
    branch: branch,
    reviewStatus: reviewStatus
) else {
    return
}

tab.updatePanelPullRequest(
    panelId: surfaceId,
    number: number,
    label: label,
    url: url,
    status: status,
    branch: branch,
    reviewStatus: reviewStatus
)
```

- [ ] **Step 4: Update `shouldReplacePullRequest` to include `reviewStatus`**

At `TerminalController.swift:386`, add the parameter and comparison:

```swift
nonisolated static func shouldReplacePullRequest(
    current: SidebarPullRequestState?,
    number: Int,
    label: String,
    url: URL,
    status: SidebarPullRequestStatus,
    branch: String?,
    reviewStatus: SidebarPullRequestReviewStatus? = nil
) -> Bool {
    guard let current else { return true }
    let normalizedBranch = branch?.trimmingCharacters(in: .whitespacesAndNewlines)
    let effectiveBranch: String? = {
        if let normalizedBranch, !normalizedBranch.isEmpty {
            return normalizedBranch
        }
        guard current.number == number,
              current.label == label,
              current.url == url,
              current.status == status else {
            return nil
        }
        return current.branch
    }()
    return current.number != number
        || current.label != label
        || current.url != url
        || current.status != status
        || current.branch != effectiveBranch
        || current.reviewStatus != reviewStatus
}
```

- [ ] **Step 5: Build to verify compilation**

```bash
xcodebuild -project GhosttyTabs.xcodeproj -scheme cmux -configuration Debug -destination 'platform=macOS' -derivedDataPath /tmp/cmux-pr-review build 2>&1 | tail -5
```

Expected: BUILD SUCCEEDED.

- [ ] **Step 6: Commit**

```bash
git add Sources/TerminalController.swift
git commit -m "feat: parse --review option in report_pr socket command"
```

---

### Task 3: Add Review Fetch to Background Poller (`TabManager.swift`)

**Files:**
- Modify: `Sources/TabManager.swift:757-762` (`WorkspacePullRequestResolvedItem`)
- Modify: `Sources/TabManager.swift:2459-2526` (`resolveWorkspacePullRequestRefreshResults`)
- Modify: `Sources/TabManager.swift:1262-1287` (refresh task)
- Modify: `Sources/TabManager.swift:1345-1482` (`applyWorkspacePullRequestRefreshResults`)

- [ ] **Step 1: Add `reviewStatusRawValue` to `WorkspacePullRequestResolvedItem`**

At `TabManager.swift:757`:

```swift
private struct WorkspacePullRequestResolvedItem: Sendable {
    let number: Int
    let urlString: String
    let statusRawValue: String
    let branch: String
    let reviewStatusRawValue: String?
}
```

- [ ] **Step 2: Add REST review item struct**

Add a new decodable struct near the other REST structs (after `WorkspacePullRequestRESTItem` around line 839):

```swift
private struct WorkspacePullRequestReviewRESTItem: Decodable, Sendable {
    let state: String
    let user: ReviewUser?

    struct ReviewUser: Decodable, Sendable {
        let login: String
    }
}
```

- [ ] **Step 3: Add review fetch function**

Add a new static function near `performWorkspacePullRequestRequest` (around line 2805):

```swift
private nonisolated static func fetchPullRequestReviewStatus(
    repoSlug: String,
    prNumber: Int,
    session: URLSession,
    authHeader: String?
) async -> SidebarPullRequestReviewStatus? {
    let endpoint = "repos/\(repoSlug)/pulls/\(prNumber)/reviews"
    guard let response = await performWorkspacePullRequestRequest(
        session: session,
        endpoint: endpoint,
        authHeader: authHeader
    ) else {
        return nil
    }
    guard response.statusCode == 200,
          let reviews = decodeJSON([WorkspacePullRequestReviewRESTItem].self, from: response.data) else {
        return nil
    }
    return aggregateReviewStatus(from: reviews)
}

private nonisolated static func aggregateReviewStatus(
    from reviews: [WorkspacePullRequestReviewRESTItem]
) -> SidebarPullRequestReviewStatus? {
    var latestByUser: [String: String] = [:]
    for review in reviews {
        guard let login = review.user?.login else { continue }
        let state = review.state.uppercased()
        if state == "APPROVED" || state == "CHANGES_REQUESTED" {
            latestByUser[login] = state
        } else if state == "DISMISSED" {
            latestByUser.removeValue(forKey: login)
        }
    }
    guard !latestByUser.isEmpty else { return nil }
    if latestByUser.values.contains("CHANGES_REQUESTED") {
        return .changesRequested
    }
    if latestByUser.values.allSatisfy({ $0 == "APPROVED" }) {
        return .approved
    }
    return nil
}
```

- [ ] **Step 4: Fetch reviews for open resolved PRs in the refresh task**

In the refresh task at line 1262, after `resolveWorkspacePullRequestRefreshResults` produces `results`, add a review-fetch pass before dispatching to main. Replace the block from line 1262-1287:

```swift
workspacePullRequestRefreshTask = Task { [weak self] in
    let repoResults = await Self.fetchWorkspacePullRequestRepoResults(
        repoDirectoriesBySlug: repoDirectoriesBySlug,
        candidateBranchesByRepo: candidateBranchesByRepo,
        cacheBySlug: cacheBySlug,
        now: now,
        allowCachedResults: allowCachedResults
    )
    var results = Self.resolveWorkspacePullRequestRefreshResults(
        candidates: candidates,
        repoResults: repoResults
    )
    guard !Task.isCancelled else { return }

    // Fetch review status for open PRs
    let configuration = URLSessionConfiguration.ephemeral
    configuration.timeoutIntervalForRequest = max(Self.workspacePullRequestProbeTimeout, 8)
    configuration.timeoutIntervalForResource = max(Self.workspacePullRequestProbeTimeout, 8)
    let reviewSession = URLSession(configuration: configuration)
    let reviewAuthHeader = workspacePullRequestAuthHeaderValue()

    results = await Self.enrichResultsWithReviewStatus(
        results: results,
        candidates: candidates,
        repoDirectoriesBySlug: repoDirectoriesBySlug,
        session: reviewSession,
        authHeader: reviewAuthHeader
    )
    guard !Task.isCancelled else { return }

    await MainActor.run { [weak self] in
        guard let self else { return }
        guard !Task.isCancelled else { return }
        self.workspacePullRequestRefreshTask = nil
        self.applyWorkspacePullRequestRefreshResults(
            results,
            repoResults: repoResults,
            requestedKeys: requestedKeys,
            now: Date(),
            reason: reason
        )
    }
}
```

- [ ] **Step 5: Add `enrichResultsWithReviewStatus` function**

```swift
private nonisolated static func enrichResultsWithReviewStatus(
    results: [WorkspacePullRequestRefreshResult],
    candidates: [WorkspacePullRequestCandidate],
    repoDirectoriesBySlug: [String: String],
    session: URLSession,
    authHeader: String?
) async -> [WorkspacePullRequestRefreshResult] {
    let enriched = await withTaskGroup(
        of: (Int, WorkspacePullRequestRefreshResult).self,
        returning: [WorkspacePullRequestRefreshResult].self
    ) { group in
        for (index, result) in results.enumerated() {
            guard case .resolved(let item) = result.resolution,
                  item.statusRawValue == SidebarPullRequestStatus.open.rawValue,
                  item.reviewStatusRawValue == nil else {
                continue
            }

            let candidate = candidates[index]
            guard let repoSlug = candidate.repoSlugs.first else { continue }

            group.addTask {
                let reviewStatus = await fetchPullRequestReviewStatus(
                    repoSlug: repoSlug,
                    prNumber: item.number,
                    session: session,
                    authHeader: authHeader
                )
                let enrichedItem = WorkspacePullRequestResolvedItem(
                    number: item.number,
                    urlString: item.urlString,
                    statusRawValue: item.statusRawValue,
                    branch: item.branch,
                    reviewStatusRawValue: reviewStatus?.rawValue
                )
                return (index, WorkspacePullRequestRefreshResult(
                    workspaceId: result.workspaceId,
                    panelId: result.panelId,
                    resolution: .resolved(enrichedItem),
                    usedCachedRepoData: result.usedCachedRepoData
                ))
            }
        }

        var updatedResults = results
        for await (index, enrichedResult) in group {
            updatedResults[index] = enrichedResult
        }
        return updatedResults
    }
    return enriched
}
```

- [ ] **Step 6: Update `resolveWorkspacePullRequestRefreshResults` to pass `nil` review status**

At line 2503, update the `WorkspacePullRequestResolvedItem` construction:

```swift
resolution = .resolved(
    WorkspacePullRequestResolvedItem(
        number: matchedPullRequest.number,
        urlString: matchedPullRequest.url,
        statusRawValue: status.rawValue,
        branch: candidate.branch,
        reviewStatusRawValue: nil
    )
)
```

- [ ] **Step 7: Update `applyWorkspacePullRequestRefreshResults` to pass `reviewStatus`**

At line 1414, in the `.resolved` case, parse and pass review status:

```swift
case .resolved(let resolvedPullRequest):
    workspacePullRequestTransientFailureCountByKey[key] = 0
    guard let status = SidebarPullRequestStatus(rawValue: resolvedPullRequest.statusRawValue),
          let url = URL(string: resolvedPullRequest.urlString) else {
        continue
    }
    let reviewStatus = resolvedPullRequest.reviewStatusRawValue
        .flatMap(SidebarPullRequestReviewStatus.init(rawValue:))
    workspace.updatePanelPullRequest(
        panelId: result.panelId,
        number: resolvedPullRequest.number,
        label: "PR",
        url: url,
        status: status,
        branch: resolvedPullRequest.branch,
        isStale: false,
        reviewStatus: reviewStatus
    )
```

- [ ] **Step 8: Build to verify compilation**

```bash
xcodebuild -project GhosttyTabs.xcodeproj -scheme cmux -configuration Debug -destination 'platform=macOS' -derivedDataPath /tmp/cmux-pr-review build 2>&1 | tail -5
```

Expected: BUILD SUCCEEDED.

- [ ] **Step 9: Commit**

```bash
git add Sources/TabManager.swift
git commit -m "feat: fetch PR review status from GitHub REST API for open PRs"
```

---

### Task 4: Update UI to Display Review Status (`ContentView.swift`)

**Files:**
- Modify: `Sources/ContentView.swift:13979-13998` (`PullRequestDisplay`, `pullRequestDisplays`)
- Modify: `Sources/ContentView.swift:13146-13174` (sidebar PR rendering)
- Modify: `Sources/ContentView.swift:14038-14044` (`pullRequestStatusLabel`)

- [ ] **Step 1: Add `reviewStatus` to `PullRequestDisplay`**

At line 13979:

```swift
private struct PullRequestDisplay: Identifiable {
    let id: String
    let number: Int
    let label: String
    let url: URL
    let status: SidebarPullRequestStatus
    let isStale: Bool
    let reviewStatus: SidebarPullRequestReviewStatus?
}
```

- [ ] **Step 2: Update `pullRequestDisplays` to pass `reviewStatus`**

At line 13988:

```swift
private func pullRequestDisplays(orderedPanelIds: [UUID]) -> [PullRequestDisplay] {
    tab.sidebarPullRequestsInDisplayOrder(orderedPanelIds: orderedPanelIds).map { pullRequest in
        PullRequestDisplay(
            id: "\(pullRequest.label.lowercased())#\(pullRequest.number)|\(pullRequest.url.absoluteString)",
            number: pullRequest.number,
            label: pullRequest.label,
            url: pullRequest.url,
            status: pullRequest.status,
            isStale: pullRequest.isStale,
            reviewStatus: pullRequest.reviewStatus
        )
    }
}
```

- [ ] **Step 3: Update `pullRequestStatusLabel` to include review suffix**

Replace the function at line 14038:

```swift
private func pullRequestStatusLabel(
    _ status: SidebarPullRequestStatus,
    reviewStatus: SidebarPullRequestReviewStatus?
) -> String {
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

- [ ] **Step 4: Update the call site in sidebar rendering**

At line 13162, update the `Text` call to pass `reviewStatus`:

```swift
Text(pullRequestStatusLabel(pullRequest.status, reviewStatus: pullRequest.reviewStatus))
```

- [ ] **Step 5: Build to verify compilation**

```bash
xcodebuild -project GhosttyTabs.xcodeproj -scheme cmux -configuration Debug -destination 'platform=macOS' -derivedDataPath /tmp/cmux-pr-review build 2>&1 | tail -5
```

Expected: BUILD SUCCEEDED.

- [ ] **Step 6: Commit**

```bash
git add Sources/ContentView.swift
git commit -m "feat: display PR review status suffix in sidebar"
```

---

### Task 5: Update Shell Integration (`cmux-zsh-integration.zsh`)

**Files:**
- Modify: `Resources/shell-integration/cmux-zsh-integration.zsh:779-851`

- [ ] **Step 1: Add `reviewDecision` to `gh pr view` JSON fields**

At line 783, change the `--json` and `--jq` arguments:

From:
```bash
&& gh pr view "$branch" \
    "${gh_repo_args[@]}" \
    --json number,state,url \
    --jq '[.number, .state, .url] | @tsv' \
```

To:
```bash
&& gh pr view "$branch" \
    "${gh_repo_args[@]}" \
    --json number,state,url,reviewDecision \
    --jq '[.number, .state, .url, .reviewDecision] | @tsv' \
```

- [ ] **Step 2: Parse the `reviewDecision` field from output**

At line 828, update the read to capture the fourth field:

From:
```bash
local IFS=$'\t'
read -r number state url <<< "$gh_output"
if [[ -z "$number" ]] || [[ -z "$url" ]]; then
    return 1
fi
```

To:
```bash
local IFS=$'\t'
read -r number state url review_decision <<< "$gh_output"
if [[ -z "$number" ]] || [[ -z "$url" ]]; then
    return 1
fi
```

- [ ] **Step 3: Build the `--review` option**

After the existing `case "$state"` block (after line 838), add review mapping:

```bash
local review_opt=""
case "$review_decision" in
    APPROVED) review_opt="--review=approved" ;;
    CHANGES_REQUESTED) review_opt="--review=changes_requested" ;;
esac
```

- [ ] **Step 4: Include `--review` in the `report_pr` socket command**

At line 851, update the `_cmux_send` call to include `$review_opt`:

From:
```bash
_cmux_send "report_pr $number $url $status_opt --branch=\"$quoted_branch\" --tab=$CMUX_TAB_ID --panel=$CMUX_PANEL_ID"
```

To:
```bash
_cmux_send "report_pr $number $url $status_opt $review_opt --branch=\"$quoted_branch\" --tab=$CMUX_TAB_ID --panel=$CMUX_PANEL_ID"
```

- [ ] **Step 5: Update the cache format to include review decision**

At line 844, update the cache write to store the review decision:

From:
```bash
printf '%s\t%s\t%s\t%s\n' "pr" "$number" "$state" "$url" >| "$result_file"
```

To:
```bash
printf '%s\t%s\t%s\t%s\t%s\n' "pr" "$number" "$state" "$url" "$review_decision" >| "$result_file"
```

- [ ] **Step 6: Commit**

```bash
git add Resources/shell-integration/cmux-zsh-integration.zsh
git commit -m "feat: emit --review flag from shell integration PR detection"
```

---

### Task 6: Add Localization Keys (`Localizable.xcstrings`)

**Files:**
- Modify: `Resources/Localizable.xcstrings`

- [ ] **Step 1: Add `sidebar.pullRequest.statusOpenApproved` key**

In the `strings` object of `Localizable.xcstrings`, add the following entry (alphabetically near the other `sidebar.pullRequest.status*` keys):

```json
"sidebar.pullRequest.statusOpenApproved": {
  "extractionState": "manual",
  "localizations": {
    "en": {
      "stringUnit": {
        "state": "translated",
        "value": "open \u00B7 approved"
      }
    },
    "ja": {
      "stringUnit": {
        "state": "translated",
        "value": "\u30AA\u30FC\u30D7\u30F3 \u00B7 \u627F\u8A8D\u6E08\u307F"
      }
    }
  }
}
```

- [ ] **Step 2: Add `sidebar.pullRequest.statusOpenChangesRequested` key**

```json
"sidebar.pullRequest.statusOpenChangesRequested": {
  "extractionState": "manual",
  "localizations": {
    "en": {
      "stringUnit": {
        "state": "translated",
        "value": "open \u00B7 changes requested"
      }
    },
    "ja": {
      "stringUnit": {
        "state": "translated",
        "value": "\u30AA\u30FC\u30D7\u30F3 \u00B7 \u5909\u66F4\u8981\u6C42"
      }
    }
  }
}
```

Note: Add translations for all other languages present in the existing `sidebar.pullRequest.statusOpen` entry (zh-Hans, zh-Hant, ko, de, es, fr, it, da, tr, uk, etc.) following the same pattern. The English value serves as the `defaultValue` fallback for untranslated locales.

- [ ] **Step 3: Commit**

```bash
git add Resources/Localizable.xcstrings
git commit -m "feat: add localization keys for PR review status labels"
```

---

### Task 7: Final Build and Smoke Test

**Files:** None (verification only)

- [ ] **Step 1: Full build**

```bash
./scripts/reload.sh --tag pr-review-status
```

Expected: Build succeeds, app path printed.

- [ ] **Step 2: Verify with debug log**

Open the tagged app, navigate to a repo with an open PR that has reviews. Tail the debug log:

```bash
tail -f /tmp/cmux-debug-pr-review-status.log
```

Verify:
- Shell integration sends `report_pr` with `--review=approved` or `--review=changes_requested`
- Sidebar shows "open · approved" or "open · changes requested"
- Merged/closed PRs still show only "merged" / "closed"
- A PR with no reviews shows just "open"

- [ ] **Step 3: Commit all remaining changes (if any)**

```bash
git add -A
git commit -m "feat: sidebar PR review status display"
```
