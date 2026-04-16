# Sidebar CI Status + GraphQL Migration Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Add CI status to sidebar PR display and migrate enrichment from N REST calls to 1 GraphQL call per repo.

**Architecture:** Extend the existing review status pipeline with CI status. Replace the REST-based `enrichResultsWithReviewStatus` with a GraphQL enrichment that fetches both `reviewDecision` and `statusCheckRollup` in a single batched query per repo. Refactor localization from combo keys to composable parts.

**Tech Stack:** Swift, SwiftUI, zsh, GitHub GraphQL API

---

### Task 1: Add CI Status Data Model (`Workspace.swift`)

**Files:**
- Modify: `Sources/Workspace.swift` (near existing `SidebarPullRequestReviewStatus` enum ~line 6044, `SidebarPullRequestState` struct ~line 6055, `updatePanelPullRequest` ~line 7663)

- [ ] **Step 1: Add `SidebarPullRequestCIStatus` enum**

Insert after `SidebarPullRequestReviewStatus` enum:

```swift
enum SidebarPullRequestCIStatus: String {
    case passing
    case failing
    case pending
}
```

- [ ] **Step 2: Add `ciStatus` field to `SidebarPullRequestState`**

Add `let ciStatus: SidebarPullRequestCIStatus?` after `reviewStatus`, and add `ciStatus: SidebarPullRequestCIStatus? = nil` to the init.

- [ ] **Step 3: Update `updatePanelPullRequest` to accept `ciStatus`**

Add `ciStatus: SidebarPullRequestCIStatus? = nil` parameter, pass through to `SidebarPullRequestState` init.

- [ ] **Step 4: Commit**

```bash
git add Sources/Workspace.swift
git commit -m "feat: add SidebarPullRequestCIStatus model"
```

---

### Task 2: Parse `--ci` in Socket Command (`TerminalController.swift`)

**Files:**
- Modify: `Sources/TerminalController.swift` (~line 15140 in `reportPullRequest`, ~line 386 in `shouldReplacePullRequest`)

- [ ] **Step 1: Parse `--ci` option in `reportPullRequest`**

After the existing `reviewStatus` parsing block, add:

```swift
let ciStatusRaw = normalizedOptionValue(parsed.options["ci"])
let ciStatus: SidebarPullRequestCIStatus?
if let ciStatusRaw {
    guard let parsedCI = SidebarPullRequestCIStatus(rawValue: ciStatusRaw.lowercased()) else {
        return "ERROR: Invalid CI status '\(ciStatusRaw)' — use: passing, failing, pending"
    }
    ciStatus = status == .open ? parsedCI : nil
} else {
    ciStatus = nil
}
```

- [ ] **Step 2: Update usage strings**

Update all usage strings to include `[--ci=passing|failing|pending]` after `[--review=...]`.

- [ ] **Step 3: Thread `ciStatus` into `shouldReplacePullRequest` and `updatePanelPullRequest`**

Add `ciStatus` parameter to both calls. Add `|| current.ciStatus != ciStatus` to the return expression in `shouldReplacePullRequest`.

- [ ] **Step 4: Update `report_review` help string**

Add `[--ci=passing|failing|pending]` to the help text at ~line 11425.

- [ ] **Step 5: Commit**

```bash
git add Sources/TerminalController.swift
git commit -m "feat: parse --ci option in report_pr socket command"
```

---

### Task 3: Migrate Enrichment to GraphQL (`TabManager.swift`)

**Files:**
- Modify: `Sources/TabManager.swift`

This is the core migration task. Replace REST review fetching with batched GraphQL queries that return both review and CI status.

- [ ] **Step 1: Add `ciStatusRawValue` to `WorkspacePullRequestResolvedItem`**

At ~line 757, add `let ciStatusRawValue: String?` field.

- [ ] **Step 2: Add GraphQL response structs**

Replace `WorkspacePullRequestReviewRESTItem` with GraphQL response types:

```swift
private struct GraphQLResponse<T: Decodable>: Decodable, Sendable where T: Sendable {
    let data: T?
    let errors: [GraphQLError]?
}

private struct GraphQLError: Decodable, Sendable {
    let message: String
}

private struct GraphQLRepositoryResponse: Decodable, Sendable {
    let repository: [String: GraphQLPullRequestNode]?

    init(from decoder: Decoder) throws {
        let container = try decoder.container(keyedBy: DynamicCodingKey.self)
        guard let repoContainer = try? container.nestedContainer(
            keyedBy: DynamicCodingKey.self,
            forKey: DynamicCodingKey(stringValue: "repository")!
        ) else {
            repository = nil
            return
        }
        var result: [String: GraphQLPullRequestNode] = [:]
        for key in repoContainer.allKeys {
            if let node = try? repoContainer.decode(GraphQLPullRequestNode.self, forKey: key) {
                result[key.stringValue] = node
            }
        }
        repository = result
    }
}

private struct GraphQLPullRequestNode: Decodable, Sendable {
    let reviewDecision: String?
    let commits: GraphQLCommitConnection?
}

private struct GraphQLCommitConnection: Decodable, Sendable {
    let nodes: [GraphQLCommitNode]?
}

private struct GraphQLCommitNode: Decodable, Sendable {
    let commit: GraphQLCommit?
}

private struct GraphQLCommit: Decodable, Sendable {
    let statusCheckRollup: GraphQLStatusCheckRollup?
}

private struct GraphQLStatusCheckRollup: Decodable, Sendable {
    let state: String?
}

private struct DynamicCodingKey: CodingKey {
    var stringValue: String
    var intValue: Int?
    init?(stringValue: String) { self.stringValue = stringValue }
    init?(intValue: Int) { self.stringValue = "\(intValue)"; self.intValue = intValue }
}
```

- [ ] **Step 3: Add GraphQL query builder**

```swift
private nonisolated static func buildPullRequestEnrichmentQuery(
    owner: String,
    repo: String,
    prNumbers: [Int]
) -> String {
    let fragment = """
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
    """
    let fields = prNumbers.map { number in
        "pr_\(number): pullRequest(number: \(number)) { \(fragment) }"
    }.joined(separator: "\n    ")
    return """
    query {
      repository(owner: "\(owner)", name: "\(repo)") {
        \(fields)
      }
    }
    """
}
```

- [ ] **Step 4: Add GraphQL fetch function**

```swift
private nonisolated static func fetchPullRequestEnrichmentViaGraphQL(
    repoSlug: String,
    prNumbers: [Int],
    session: URLSession,
    authHeader: String?
) async -> [Int: (reviewStatus: SidebarPullRequestReviewStatus?, ciStatus: SidebarPullRequestCIStatus?)] {
    guard !prNumbers.isEmpty else { return [:] }
    let components = repoSlug.split(separator: "/")
    guard components.count == 2 else { return [:] }
    let owner = String(components[0])
    let repo = String(components[1])

    let query = buildPullRequestEnrichmentQuery(owner: owner, repo: repo, prNumbers: prNumbers)
    let body: [String: Any] = ["query": query]
    guard let bodyData = try? JSONSerialization.data(withJSONObject: body) else { return [:] }

    guard let url = URL(string: "https://api.github.com/graphql") else { return [:] }
    var request = URLRequest(url: url)
    request.httpMethod = "POST"
    request.setValue("application/json", forHTTPHeaderField: "Content-Type")
    request.setValue("application/json", forHTTPHeaderField: "Accept")
    request.setValue("cmux-workspace-pr-poller", forHTTPHeaderField: "User-Agent")
    if let authHeader, !authHeader.isEmpty {
        request.setValue(authHeader, forHTTPHeaderField: "Authorization")
    }
    request.httpBody = bodyData

    guard let (data, response) = try? await session.data(for: request),
          let httpResponse = response as? HTTPURLResponse,
          httpResponse.statusCode == 200 else {
        return [:]
    }

    guard let graphQLResponse = decodeJSON(GraphQLResponse<GraphQLRepositoryResponse>.self, from: data),
          let repository = graphQLResponse.data?.repository else {
        return [:]
    }

    var results: [Int: (reviewStatus: SidebarPullRequestReviewStatus?, ciStatus: SidebarPullRequestCIStatus?)] = [:]
    for prNumber in prNumbers {
        let key = "pr_\(prNumber)"
        guard let node = repository[key] else { continue }

        let reviewStatus: SidebarPullRequestReviewStatus? = {
            switch node.reviewDecision?.uppercased() {
            case "APPROVED": return .approved
            case "CHANGES_REQUESTED": return .changesRequested
            default: return nil
            }
        }()

        let ciStatus: SidebarPullRequestCIStatus? = {
            let state = node.commits?.nodes?.first?.commit?.statusCheckRollup?.state?.uppercased()
            switch state {
            case "SUCCESS": return .passing
            case "FAILURE", "ERROR": return .failing
            case "PENDING", "EXPECTED": return .pending
            default: return nil
            }
        }()

        results[prNumber] = (reviewStatus: reviewStatus, ciStatus: ciStatus)
    }
    return results
}
```

- [ ] **Step 5: Replace `enrichResultsWithReviewStatus` with GraphQL-based enrichment**

Replace the function at ~line 2910:

```swift
private nonisolated static func enrichResultsWithGraphQL(
    results: [WorkspacePullRequestRefreshResult],
    candidates: [WorkspacePullRequestCandidate],
    session: URLSession,
    authHeader: String?
) async -> [WorkspacePullRequestRefreshResult] {
    // Group open PRs by repo slug
    var prNumbersByRepo: [String: [(index: Int, item: WorkspacePullRequestResolvedItem)]] = [:]
    for (index, result) in results.enumerated() {
        guard case .resolved(let item) = result.resolution,
              item.statusRawValue == SidebarPullRequestStatus.open.rawValue else {
            continue
        }
        let candidate = candidates[index]
        guard let repoSlug = candidate.repoSlugs.first else { continue }
        prNumbersByRepo[repoSlug, default: []].append((index: index, item: item))
    }

    guard !prNumbersByRepo.isEmpty else { return results }

    // One GraphQL call per repo (concurrent across repos)
    let enrichments = await withTaskGroup(
        of: (String, [Int: (reviewStatus: SidebarPullRequestReviewStatus?, ciStatus: SidebarPullRequestCIStatus?)]).self,
        returning: [String: [Int: (reviewStatus: SidebarPullRequestReviewStatus?, ciStatus: SidebarPullRequestCIStatus?)]].self
    ) { group in
        for (repoSlug, _) in prNumbersByRepo {
            let prNumbers = prNumbersByRepo[repoSlug]!.map(\.item.number)
            group.addTask {
                let result = await fetchPullRequestEnrichmentViaGraphQL(
                    repoSlug: repoSlug,
                    prNumbers: prNumbers,
                    session: session,
                    authHeader: authHeader
                )
                return (repoSlug, result)
            }
        }
        var collected: [String: [Int: (reviewStatus: SidebarPullRequestReviewStatus?, ciStatus: SidebarPullRequestCIStatus?)]] = [:]
        for await (repoSlug, result) in group {
            collected[repoSlug] = result
        }
        return collected
    }

    // Apply enrichments to results
    var updatedResults = results
    for (repoSlug, entries) in prNumbersByRepo {
        guard let repoEnrichments = enrichments[repoSlug] else { continue }
        for (index, item) in entries {
            let enrichment = repoEnrichments[item.number]
            let enrichedItem = WorkspacePullRequestResolvedItem(
                number: item.number,
                urlString: item.urlString,
                statusRawValue: item.statusRawValue,
                branch: item.branch,
                reviewStatusRawValue: enrichment?.reviewStatus?.rawValue ?? item.reviewStatusRawValue,
                ciStatusRawValue: enrichment?.ciStatus?.rawValue
            )
            updatedResults[index] = WorkspacePullRequestRefreshResult(
                workspaceId: results[index].workspaceId,
                panelId: results[index].panelId,
                resolution: .resolved(enrichedItem),
                usedCachedRepoData: results[index].usedCachedRepoData
            )
        }
    }
    return updatedResults
}
```

- [ ] **Step 6: Remove old REST review functions**

Delete:
- `WorkspacePullRequestReviewRESTItem` struct (~line 842)
- `fetchPullRequestReviewStatus` function (~line 2866)
- `aggregateReviewStatus` function (~line 2887)

- [ ] **Step 7: Update refresh task to call `enrichResultsWithGraphQL`**

At ~line 1294, change:
```swift
results = await Self.enrichResultsWithReviewStatus(
    results: results,
    candidates: candidates,
    session: reviewSession,
    authHeader: reviewAuthHeader
)
```
To:
```swift
results = await Self.enrichResultsWithGraphQL(
    results: results,
    candidates: candidates,
    session: reviewSession,
    authHeader: reviewAuthHeader
)
```

- [ ] **Step 8: Update `resolveWorkspacePullRequestRefreshResults`**

At the `WorkspacePullRequestResolvedItem` construction (~line 2537), add `ciStatusRawValue: nil`.

- [ ] **Step 9: Update `applyWorkspacePullRequestRefreshResults`**

In the `.resolved` case (~line 1440), parse and pass CI status:

```swift
let reviewStatus = resolvedPullRequest.reviewStatusRawValue
    .flatMap(SidebarPullRequestReviewStatus.init(rawValue:))
let ciStatus = resolvedPullRequest.ciStatusRawValue
    .flatMap(SidebarPullRequestCIStatus.init(rawValue:))
workspace.updatePanelPullRequest(
    panelId: result.panelId,
    number: resolvedPullRequest.number,
    label: "PR",
    url: url,
    status: status,
    branch: resolvedPullRequest.branch,
    isStale: false,
    reviewStatus: reviewStatus,
    ciStatus: ciStatus
)
```

- [ ] **Step 10: Commit**

```bash
git add Sources/TabManager.swift
git commit -m "feat: migrate PR enrichment from REST to GraphQL, add CI status"
```

---

### Task 4: Update UI to Display CI Status (`ContentView.swift`)

**Files:**
- Modify: `Sources/ContentView.swift`

- [ ] **Step 1: Add `ciStatus` to `PullRequestDisplay`**

Add `let ciStatus: SidebarPullRequestCIStatus?` to the struct.

- [ ] **Step 2: Update `pullRequestDisplays` to pass `ciStatus`**

Add `ciStatus: pullRequest.ciStatus` to the `PullRequestDisplay` construction.

- [ ] **Step 3: Add helper functions for review and CI labels**

```swift
private func pullRequestReviewStatusLabel(_ reviewStatus: SidebarPullRequestReviewStatus) -> String {
    switch reviewStatus {
    case .approved:
        return String(localized: "sidebar.pullRequest.reviewApproved", defaultValue: "approved")
    case .changesRequested:
        return String(localized: "sidebar.pullRequest.reviewChangesRequested", defaultValue: "changes requested")
    }
}

private func pullRequestCIStatusLabel(_ ciStatus: SidebarPullRequestCIStatus) -> String {
    switch ciStatus {
    case .passing:
        return String(localized: "sidebar.pullRequest.ciPassing", defaultValue: "passing")
    case .failing:
        return String(localized: "sidebar.pullRequest.ciFailing", defaultValue: "failing")
    case .pending:
        return String(localized: "sidebar.pullRequest.ciPending", defaultValue: "pending")
    }
}
```

- [ ] **Step 4: Rewrite `pullRequestStatusLabel` to build from parts**

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

- [ ] **Step 5: Update call site**

```swift
Text(pullRequestStatusLabel(pullRequest.status, reviewStatus: pullRequest.reviewStatus, ciStatus: pullRequest.ciStatus))
```

- [ ] **Step 6: Commit**

```bash
git add Sources/ContentView.swift
git commit -m "feat: display CI status suffix in sidebar PR label"
```

---

### Task 5: Update Shell Integration (`cmux-zsh-integration.zsh`)

**Files:**
- Modify: `Resources/shell-integration/cmux-zsh-integration.zsh`

- [ ] **Step 1: Add `statusCheckRollup` to `gh pr view` JSON fields**

Change:
```bash
--json number,state,url,reviewDecision \
--jq '[.number, .state, .url, .reviewDecision] | @tsv' \
```
To:
```bash
--json number,state,url,reviewDecision,statusCheckRollup \
--jq '[.number, .state, .url, .reviewDecision, .statusCheckRollup] | @tsv' \
```

- [ ] **Step 2: Parse the fifth field**

Change:
```bash
read -r number state url review_decision <<< "$gh_output"
```
To:
```bash
read -r number state url review_decision ci_status <<< "$gh_output"
```

- [ ] **Step 3: Build `--ci` option**

After the `review_opt` case block, add:

```bash
local ci_opt=""
case "$ci_status" in
    SUCCESS) ci_opt="--ci=passing" ;;
    FAILURE|ERROR) ci_opt="--ci=failing" ;;
    PENDING|EXPECTED) ci_opt="--ci=pending" ;;
esac
```

- [ ] **Step 4: Include `$ci_opt` in `_cmux_send`**

Change:
```bash
_cmux_send "report_pr $number $url $status_opt $review_opt --branch=\"$quoted_branch\" --tab=$CMUX_TAB_ID --panel=$CMUX_PANEL_ID"
```
To:
```bash
_cmux_send "report_pr $number $url $status_opt $review_opt $ci_opt --branch=\"$quoted_branch\" --tab=$CMUX_TAB_ID --panel=$CMUX_PANEL_ID"
```

- [ ] **Step 5: Update cache format**

Change:
```bash
printf '%s\t%s\t%s\t%s\t%s\n' "pr" "$number" "$state" "$url" "$review_decision" >| "$result_file"
```
To:
```bash
printf '%s\t%s\t%s\t%s\t%s\t%s\n' "pr" "$number" "$state" "$url" "$review_decision" "$ci_status" >| "$result_file"
```

- [ ] **Step 6: Commit**

```bash
git add Resources/shell-integration/cmux-zsh-integration.zsh
git commit -m "feat: emit --ci flag from shell integration PR detection"
```

---

### Task 6: Update Localization (`Localizable.xcstrings`)

**Files:**
- Modify: `Resources/Localizable.xcstrings`

- [ ] **Step 1: Remove old combo keys**

Delete `sidebar.pullRequest.statusOpenApproved` and `sidebar.pullRequest.statusOpenChangesRequested` entries.

- [ ] **Step 2: Add individual review part keys**

Add:
- `sidebar.pullRequest.reviewApproved` — en: `"approved"`, ja: `"承認済み"`
- `sidebar.pullRequest.reviewChangesRequested` — en: `"changes requested"`, ja: `"変更要求"`

Include all languages present in the existing `sidebar.pullRequest.statusOpen` entry.

- [ ] **Step 3: Add CI status keys**

Add:
- `sidebar.pullRequest.ciPassing` — en: `"passing"`, ja: `"成功"`
- `sidebar.pullRequest.ciFailing` — en: `"failing"`, ja: `"失敗"`
- `sidebar.pullRequest.ciPending` — en: `"pending"`, ja: `"保留中"`

Include all languages present in the existing `sidebar.pullRequest.statusOpen` entry.

- [ ] **Step 4: Commit**

```bash
git add Resources/Localizable.xcstrings
git commit -m "feat: add localization keys for review and CI status parts"
```

---

### Task 7: Build and Verify

- [ ] **Step 1: Full build**

```bash
./scripts/reload.sh --tag pr-ci-status
```

Expected: BUILD SUCCEEDED.

- [ ] **Step 2: Verify**

Open the tagged app, navigate to a repo with an open PR that has reviews and CI checks. Verify:
- "open · approved · passing" for approved PR with green CI
- "open · changes requested · failing" for requested changes with red CI
- "open · approved" for approved PR with no CI
- "open · passing" for PR with CI but no reviews
- "open" for PR with neither
- Merged/closed PRs show only "merged" / "closed"
