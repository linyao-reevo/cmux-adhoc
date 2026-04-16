# Sidebar CI Status + GraphQL Enrichment Migration

Add CI status to the sidebar PR display and migrate the per-open-PR enrichment from REST to GraphQL.

## Goal

Open PRs in the sidebar show review status ("open · approved"). This change adds CI/check status and migrates the background poller's enrichment pass from N REST calls to 1 GraphQL call per repo, fetching both review and CI status together.

## Display Format

The status text for open PRs gains an optional CI suffix:

| Review | CI | Sidebar Text |
|---|---|---|
| none | none | `open` |
| approved | none | `open · approved` |
| changes requested | none | `open · changes requested` |
| none | passing | `open · passing` |
| none | failing | `open · failing` |
| none | pending | `open · pending` |
| approved | passing | `open · approved · passing` |
| approved | failing | `open · approved · failing` |
| changes requested | failing | `open · changes requested · failing` |
| any | any | `open [· review] [· ci]` |

Merged and closed PRs never show review or CI suffix. Fallback is always `open`.

## Data Model

### New enum (`Workspace.swift`)

```swift
enum SidebarPullRequestCIStatus: String {
    case passing
    case failing
    case pending
}
```

### Updated struct (`Workspace.swift`)

Add `ciStatus` to `SidebarPullRequestState`:

```swift
struct SidebarPullRequestState {
    // ... existing fields ...
    let reviewStatus: SidebarPullRequestReviewStatus?
    let ciStatus: SidebarPullRequestCIStatus?      // new
}
```

## Data Sources

### 1. Shell integration (`cmux-zsh-integration.zsh`)

The existing `gh pr view --json` call already fetches `reviewDecision`. Add `statusCheckRollup`:

```bash
gh pr view "$branch" --json number,state,url,reviewDecision,statusCheckRollup \
    --jq '[.number, .state, .url, .reviewDecision, .statusCheckRollup] | @tsv'
```

Map `statusCheckRollup`:
- `SUCCESS` → `--ci=passing`
- `FAILURE` / `ERROR` → `--ci=failing`
- `PENDING` / `EXPECTED` → `--ci=pending`
- Empty/null → omit `--ci` flag

This is zero additional cost — same single `gh pr view` call.

### 2. Background poller — GraphQL migration (`TabManager.swift`)

**Replace** the current REST-based `enrichResultsWithReviewStatus` (which makes one `GET /pulls/{number}/reviews` per open PR) with a single GraphQL query per repo.

#### GraphQL query structure

For each repo that has open PRs visible in the sidebar, issue one `POST https://api.github.com/graphql` call:

```graphql
query {
  repository(owner: "owner", name: "repo") {
    pr_42: pullRequest(number: 42) {
      reviewDecision
      commits(last: 1) {
        nodes {
          commit {
            statusCheckRollup {
              state
            }
          }
        }
      }
    }
    pr_55: pullRequest(number: 55) {
      reviewDecision
      commits(last: 1) {
        nodes {
          commit {
            statusCheckRollup {
              state
            }
          }
        }
      }
    }
  }
}
```

Each PR is aliased as `pr_{number}` to batch them in one query.

#### GraphQL response mapping

- `reviewDecision`: `APPROVED` → `.approved`, `CHANGES_REQUESTED` → `.changesRequested`, null/other → `nil`
- `statusCheckRollup.state`: `SUCCESS` → `.passing`, `FAILURE`/`ERROR` → `.failing`, `PENDING`/`EXPECTED` → `.pending`, null → `nil`

#### Migration scope

| Component | Before | After |
|---|---|---|
| `enrichResultsWithReviewStatus` | N REST calls (1 per open PR) | 1 GraphQL call per repo |
| `fetchPullRequestReviewStatus` | REST `/pulls/{n}/reviews` | Removed |
| `aggregateReviewStatus` | Manual per-reviewer aggregation | Removed (GraphQL gives `reviewDecision` directly) |
| `WorkspacePullRequestReviewRESTItem` | REST review struct | Removed |
| New: GraphQL request function | — | `POST /graphql` with query + JSON response parsing |
| New: GraphQL response structs | — | Decodable structs for the response shape |

#### Auth and infrastructure

- Same auth header (`Bearer` token from `GH_TOKEN`/`GITHUB_TOKEN`/`gh auth token`)
- Same timeout and URLSession configuration
- Same ephemeral session pattern
- GraphQL endpoint: `https://api.github.com/graphql` (POST, not GET)
- On failure: preserve last known review/CI status (existing stale pattern)

### 3. Socket command (`TerminalController.swift`)

Update `report_pr` to accept `--ci=passing|failing|pending`:

```
report_pr <number> <url> [--label=PR] [--state=open|merged|closed] [--review=approved|changes_requested] [--ci=passing|failing|pending] [--branch=<name>] [--tab=X] [--panel=Y]
```

- `--ci` is optional. When omitted, `ciStatus` is `nil`.
- Invalid values produce an error.
- Ignored for merged/closed PRs.

## UI Changes (`ContentView.swift`)

### Status label

Build the label programmatically from parts rather than one localization key per combination:

```swift
private func pullRequestStatusLabel(
    _ status: SidebarPullRequestStatus,
    reviewStatus: SidebarPullRequestReviewStatus?,
    ciStatus: SidebarPullRequestCIStatus?
) -> String {
    switch status {
    case .open:
        var parts = [String(localized: "sidebar.pullRequest.statusOpen", defaultValue: "open")]
        if let reviewStatus {
            parts.append(pullRequestReviewStatusLabel(reviewStatus))
        }
        if let ciStatus {
            parts.append(pullRequestCIStatusLabel(ciStatus))
        }
        return parts.joined(separator: " \u{00B7} ")
    case .merged:
        return String(localized: "sidebar.pullRequest.statusMerged", defaultValue: "merged")
    case .closed:
        return String(localized: "sidebar.pullRequest.statusClosed", defaultValue: "closed")
    }
}
```

This avoids combinatorial localization keys. Individual review and CI labels are localized separately.

## Localization

Replace the two combo keys (`statusOpenApproved`, `statusOpenChangesRequested`) with individual part keys:

- `sidebar.pullRequest.reviewApproved` (en: `"approved"`)
- `sidebar.pullRequest.reviewChangesRequested` (en: `"changes requested"`)
- `sidebar.pullRequest.ciPassing` (en: `"passing"`)
- `sidebar.pullRequest.ciFailing` (en: `"failing"`)
- `sidebar.pullRequest.ciPending` (en: `"pending"`)

Remove old combo keys: `statusOpenApproved`, `statusOpenChangesRequested`.

## Files Changed

| File | Change |
|---|---|
| `Sources/Workspace.swift` | Add `SidebarPullRequestCIStatus` enum, add `ciStatus` field |
| `Sources/TerminalController.swift` | Parse `--ci` option, thread through |
| `Sources/TabManager.swift` | Replace REST enrichment with GraphQL; add query builder, response parsing; remove REST review structs/functions |
| `Sources/ContentView.swift` | Update `pullRequestStatusLabel` to build from parts with CI suffix |
| `Resources/shell-integration/cmux-zsh-integration.zsh` | Add `statusCheckRollup` to `gh pr view`, emit `--ci` flag |
| `Resources/Localizable.xcstrings` | Add 5 new part keys, remove 2 old combo keys |

## Non-Goals

- Showing individual check run names or details
- CI status for merged/closed PRs
- Changing the bulk `/pulls` REST fetch to GraphQL (only the enrichment pass migrates)
