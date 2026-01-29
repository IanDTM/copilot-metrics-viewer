# GitHub API Architecture Documentation

## Overview

This document provides a comprehensive deep dive into how the GitHub Copilot Metrics Viewer interacts with the GitHub API, including:
1. Complete list of GitHub API endpoints called and the data returned
2. Sequence of API calls for different user request scenarios
3. How data is joined, transformed, and displayed throughout the system

---

## Table of Contents

- [Part 1: GitHub API Endpoints](#part-1-github-api-endpoints)
- [Part 2: API Call Sequences](#part-2-api-call-sequences)
- [Part 3: Data Flow and Transformations](#part-3-data-flow-and-transformations)

---

## Part 1: GitHub API Endpoints

### 1.1 Copilot Metrics Endpoints

These endpoints return usage statistics for GitHub Copilot including suggestions, acceptances, active users, and feature breakdown.

#### Organization Metrics
```
GET https://api.github.com/orgs/{org}/copilot/metrics
```

**Query Parameters:**
- `since` (optional): Start date (ISO 8601 format, e.g., "2024-01-01")
- `until` (optional): End date (ISO 8601 format, e.g., "2024-01-28")

**Response Format:**
```json
[
  {
    "date": "2024-01-15",
    "total_active_users": 45,
    "total_engaged_users": 42,
    "copilot_ide_code_completions": {
      "total_engaged_users": 38,
      "editors": [
        {
          "name": "vscode",
          "total_engaged_users": 30,
          "models": [
            {
              "name": "default",
              "is_custom_model": false,
              "custom_model_training_date": null,
              "total_engaged_users": 30,
              "languages": [
                {
                  "name": "python",
                  "total_engaged_users": 25,
                  "total_code_suggestions": 5432,
                  "total_code_acceptances": 3210,
                  "total_code_lines_suggested": 8765,
                  "total_code_lines_accepted": 5432
                }
              ]
            }
          ]
        }
      ]
    },
    "copilot_ide_chat": {
      "total_engaged_users": 28,
      "editors": [
        {
          "name": "vscode",
          "total_engaged_users": 25,
          "models": [
            {
              "name": "gpt-4",
              "is_custom_model": false,
              "custom_model_training_date": null,
              "total_engaged_users": 25,
              "total_chats": 1234,
              "total_chat_insertion_events": 567,
              "total_chat_copy_events": 234
            }
          ]
        }
      ]
    },
    "copilot_dotcom_chat": {
      "total_engaged_users": 15,
      "models": [
        {
          "name": "gpt-4",
          "is_custom_model": false,
          "custom_model_training_date": null,
          "total_engaged_users": 15,
          "total_chats": 456
        }
      ]
    },
    "copilot_dotcom_pull_requests": {
      "total_engaged_users": 12,
      "repositories": [
        {
          "name": "my-repo",
          "total_engaged_users": 10,
          "models": [
            {
              "name": "gpt-4",
              "is_custom_model": false,
              "custom_model_training_date": null,
              "total_engaged_users": 10,
              "total_pr_summaries_created": 45
            }
          ]
        }
      ]
    }
  }
]
```

**Data Returned:**
- Daily metrics array (one object per day in the date range)
- Code completion statistics (suggestions, acceptances, lines, languages)
- IDE chat usage (conversations, insertions, copy events)
- GitHub.com chat usage
- Pull request summary statistics
- Breakdown by editor (VS Code, IntelliJ, etc.)
- Breakdown by AI model (GPT-4, Claude, custom models)
- Breakdown by programming language

#### Enterprise Metrics
```
GET https://api.github.com/enterprises/{enterprise}/copilot/metrics
```

**Same format as organization metrics** but aggregated across all organizations in the enterprise.

#### Team Metrics (Organization)
```
GET https://api.github.com/orgs/{org}/team/{team_slug}/copilot/metrics
```

**Same format as organization metrics** but filtered to team members only.

#### Team Metrics (Enterprise)
```
GET https://api.github.com/enterprises/{enterprise}/team/{team_slug}/copilot/metrics
```

**Same format as organization metrics** but filtered to enterprise team members only.

---

### 1.2 Copilot Seat Endpoints

These endpoints return information about Copilot seat assignments and usage status.

#### Organization Seats
```
GET https://api.github.com/orgs/{org}/copilot/billing/seats
```

**Query Parameters:**
- `page` (optional): Page number for pagination (default: 1)
- `per_page` (optional): Results per page (default: 30, max: 100)

**Response Format:**
```json
{
  "total_seats": 150,
  "seats": [
    {
      "created_at": "2023-10-01T12:00:00Z",
      "updated_at": "2024-01-15T08:30:00Z",
      "pending_cancellation_date": null,
      "last_activity_at": "2024-01-14T16:45:00Z",
      "last_activity_editor": "vscode",
      "assignee": {
        "login": "johndoe",
        "id": 12345678,
        "node_id": "MDQ6VXNlcjEyMzQ1Njc4",
        "avatar_url": "https://avatars.githubusercontent.com/u/12345678?v=4",
        "type": "User"
      },
      "assigning_team": null,
      "organization": {
        "login": "acme-corp",
        "id": 87654321
      }
    }
  ]
}
```

**Data Returned:**
- Total seat count
- Array of seat assignments with:
  - User information (login, ID, avatar)
  - Seat creation and update timestamps
  - Last activity timestamp and editor used
  - Organization assignment
  - Team assignment (if applicable)
  - Pending cancellation status

**Pagination:** Response includes pagination via query parameters. The application automatically fetches all pages.

#### Enterprise Seats
```
GET https://api.github.com/enterprises/{enterprise}/copilot/billing/seats
```

**Same format as organization seats** but includes users across all organizations in the enterprise. May include duplicates (same user in multiple orgs) which are deduplicated by the application.

---

### 1.3 Teams Endpoints

These endpoints list teams within an organization or enterprise.

#### Organization Teams
```
GET https://api.github.com/orgs/{org}/teams
```

**Query Parameters:**
- `per_page` (optional): Results per page (default: 30, max: 100)

**Response Format:**
```json
[
  {
    "id": 123456,
    "node_id": "MDQ6VGVhbTEyMzQ1Ng==",
    "name": "Engineering Team",
    "slug": "engineering-team",
    "description": "Core engineering team responsible for product development",
    "privacy": "closed",
    "permission": "pull",
    "members_count": 25,
    "repos_count": 42
  }
]
```

**Data Returned:**
- Array of team objects with:
  - Team ID and slug
  - Team name and description
  - Privacy and permission settings
  - Member and repository counts

**Pagination:** Response includes `Link` header for pagination. The application follows the `next` link until no more pages exist.

#### Enterprise Teams
```
GET https://api.github.com/enterprises/{enterprise}/teams
```

**Same format as organization teams** but includes teams across all organizations in the enterprise.

---

### 1.4 Team Members Endpoint

This endpoint lists members of a specific team.

#### Organization Team Members
```
GET https://api.github.com/orgs/{org}/teams/{team_slug}/members
```

**Query Parameters:**
- `page` (optional): Page number for pagination (default: 1)
- `per_page` (optional): Results per page (default: 30, max: 100)

**Response Format:**
```json
[
  {
    "login": "johndoe",
    "id": 12345678,
    "node_id": "MDQ6VXNlcjEyMzQ1Njc4",
    "avatar_url": "https://avatars.githubusercontent.com/u/12345678?v=4",
    "type": "User",
    "site_admin": false
  }
]
```

**Data Returned:**
- Array of user objects (team members) with:
  - User login and ID
  - Avatar URL
  - User type and admin status

**Usage:** The application fetches team members when filtering seats by team scope, then matches seat assignments to team member IDs.

---

### 1.5 Authentication Headers

All GitHub API requests include these headers:

```http
Accept: application/vnd.github+json
Authorization: token {github_token}
X-GitHub-Api-Version: 2022-11-28
```

**Authentication Methods:**
1. **GitHub Personal Access Token (PAT):** Via `NUXT_GITHUB_TOKEN` environment variable
2. **GitHub OAuth:** Via GitHub App authentication (access token from user session)
3. **Mock Mode:** Uses dummy token for development/testing

**Required Token Scopes:**
- `copilot` - Access Copilot usage metrics
- `read:org` - Read organization data
- `read:enterprise` - Read enterprise data (for enterprise scope)
- `manage_billing:copilot` - Access billing/seat information
- `manage_billing:enterprise` - Enterprise billing (for enterprise scope)

---

## Part 2: API Call Sequences

### 2.1 Initial Page Load - Organization View

**User Action:** Visit `http://localhost:3000/orgs/acme-corp`

**Sequence:**
```
1. Browser requests page
   └─→ Nuxt server renders index.vue with MainComponent.vue

2. MainComponent.vue mounts (client-side)
   │
   ├─→ API Call #1 (parallel): GET /api/metrics
   │   Query: ?scope=organization&githubOrg=acme-corp&since=2024-01-01&until=2024-01-28
   │   └─→ Server endpoint: /server/api/metrics.ts
   │       └─→ Calls getMetricsData() from shared/utils/metrics-util.ts
   │           └─→ Builds URL: https://api.github.com/orgs/acme-corp/copilot/metrics?since=2024-01-01&until=2024-01-28
   │               └─→ Returns: CopilotMetrics[] (28 days of data)
   │
   └─→ API Call #2 (parallel): GET /api/seats
       Query: ?scope=organization&githubOrg=acme-corp
       └─→ Server endpoint: /server/api/seats.ts
           └─→ Builds URL: https://api.github.com/orgs/acme-corp/copilot/billing/seats
               ├─→ Page 1: ?per_page=100&page=1 (returns 100 seats)
               ├─→ Page 2: ?per_page=100&page=2 (returns 50 seats)
               └─→ Returns: Seat[] (150 total seats, deduplicated)

3. MetricsViewer.vue renders with metrics data
   └─→ Displays: Acceptance rate, active users, language breakdown charts

4. User sees initial dashboard with "languages/editors" tab active
```

**Total GitHub API Calls:** 3 (1 metrics + 2 seats pagination)

**Response Time:** ~2-3 seconds (depends on data size and GitHub API latency)

---

### 2.2 Switching to GitHub.com Statistics Tab

**User Action:** Click "github.com" tab

**Sequence:**
```
1. Tab switch triggers AgentModeViewer.vue component mount

2. Component watch triggers fetchStats() (debounced 150ms)
   └─→ API Call: GET /api/github-stats
       Query: ?scope=organization&githubOrg=acme-corp&since=2024-01-01&until=2024-01-28
       └─→ Server endpoint: /server/api/github-stats.ts
           └─→ Calls getMetricsData() [uses cached data from initial load]
               └─→ Transforms metrics into aggregated statistics
                   ├─→ Counts unique models across all days
                   ├─→ Sums engaged users across date range
                   ├─→ Builds chart datasets for visualization
                   └─→ Returns: GitHubStats object

3. AgentModeViewer.vue renders 4 expansion panels + 2 charts
   ├─→ IDE Code Completions (models, users, editors)
   ├─→ IDE Chat (models, users, conversations)
   ├─→ GitHub.com Chat (models, users, chats)
   └─→ GitHub.com PR Summaries (models, users, summaries)
```

**Total New GitHub API Calls:** 0 (reuses cached metrics)

**Server Processing:** Aggregates and transforms existing metrics data

---

### 2.3 Viewing Seat Analysis

**User Action:** Click "seat analysis" tab

**Sequence:**
```
1. Tab switch renders SeatsAnalysisViewer.vue

2. Component processes seats data (client-side, no API calls)
   ├─→ Filters: Never used seats (last_activity_at === null)
   ├─→ Filters: No activity in 7 days
   ├─→ Filters: No activity in 30 days
   └─→ Sorts: By last_activity_at (most recent first)

3. Displays tables:
   ├─→ Active seats (with last activity date/editor)
   ├─→ Inactive seats (with recommendations)
   └─→ Summary statistics (total, active, inactive counts)
```

**Total New GitHub API Calls:** 0 (uses seats data from initial load)

---

### 2.4 Loading Teams Comparison

**User Action:** Click "teams" tab (organization or enterprise scope)

**Sequence:**
```
1. Tab switch renders TeamsComponent.vue

2. Component mounts and loads teams
   └─→ API Call #1: GET /api/teams
       Query: ?scope=organization&githubOrg=acme-corp
       └─→ Server endpoint: /server/api/teams.ts
           └─→ Builds URL: https://api.github.com/orgs/acme-corp/teams?per_page=100
               ├─→ Page 1: Returns 100 teams
               ├─→ Page 2: Returns 25 teams (follows Link header)
               └─→ Returns: Team[] (125 total teams)

3. User selects 3 teams for comparison: "frontend-team", "backend-team", "qa-team"

4. Component fetches metrics for each team IN PARALLEL
   ├─→ API Call #2: GET /api/metrics?scope=team-organization&githubOrg=acme-corp&githubTeam=frontend-team&since=...&until=...
   │   └─→ Returns: CopilotMetrics[] for frontend team
   │
   ├─→ API Call #3: GET /api/metrics?scope=team-organization&githubOrg=acme-corp&githubTeam=backend-team&since=...&until=...
   │   └─→ Returns: CopilotMetrics[] for backend team
   │
   └─→ API Call #4: GET /api/metrics?scope=team-organization&githubOrg=acme-corp&githubTeam=qa-team&since=...&until=...
       └─→ Returns: CopilotMetrics[] for qa team

5. Component aggregates metrics from all teams
   └─→ Builds 8 line charts + 2 bar charts comparing:
       ├─→ Acceptance rates over time
       ├─→ Active users per team
       ├─→ Feature usage (completions, chat, PR summaries)
       └─→ Language breakdown by team
```

**Total GitHub API Calls:** 5 (2 teams pagination + 3 team metrics)

**Note:** If user selects 10 teams, this becomes 12 API calls (2 teams + 10 metrics)

---

### 2.5 Changing Date Range

**User Action:** Change date range from "Last 28 days" to "Last 7 days"

**Sequence:**
```
1. DateRangeSelector emits change event
   └─→ MainComponent.vue handles event
       └─→ Updates: since = "2024-01-21", until = "2024-01-28"

2. Calls fetchMetrics() with new dates
   └─→ API Call: GET /api/metrics
       Query: ?scope=organization&githubOrg=acme-corp&since=2024-01-21&until=2024-01-28
       └─→ Server checks cache (different date range = cache miss)
           └─→ GitHub API Call: https://api.github.com/orgs/acme-corp/copilot/metrics?since=2024-01-21&until=2024-01-28
               └─→ Returns: CopilotMetrics[] (7 days of data)
               └─→ Caches with 5-minute TTL

3. All components re-render with new metrics
   ├─→ MetricsViewer: Updates charts with 7 days of data
   ├─→ AgentModeViewer: Re-fetches /api/github-stats with new date range
   └─→ TeamsComponent: Re-fetches team metrics with new date range
```

**Total GitHub API Calls:** Varies (1-5+ depending on active tab and team selections)

---

### 2.6 Enterprise View with Team Filtering

**User Action:** Visit `http://localhost:3000/enterprises/acme-ent/teams/platform-team`

**Sequence:**
```
1. Browser requests page
   └─→ Nuxt extracts params: { ent: "acme-ent", team: "platform-team" }

2. MainComponent.vue mounts with scope = "team-enterprise"

3. API Call #1 (parallel): GET /api/metrics
   Query: ?scope=team-enterprise&githubEnt=acme-ent&githubTeam=platform-team&since=...&until=...
   └─→ Server endpoint: /server/api/metrics.ts
       └─→ GitHub API: https://api.github.com/enterprises/acme-ent/team/platform-team/copilot/metrics
           └─→ Returns: CopilotMetrics[] (filtered to team members)

4. API Call #2 (parallel): GET /api/seats
   Query: ?scope=team-enterprise&githubEnt=acme-ent&githubTeam=platform-team
   └─→ Server endpoint: /server/api/seats.ts
       ├─→ Fetches team members first:
       │   └─→ GitHub API: https://api.github.com/enterprises/acme-ent/teams/platform-team/members?per_page=100
       │       └─→ Page 1: Returns 35 members
       │       └─→ Returns: TeamMember[] (35 members)
       │
       └─→ Fetches all enterprise seats:
           └─→ GitHub API: https://api.github.com/enterprises/acme-ent/copilot/billing/seats?per_page=100
               ├─→ Page 1-10: Returns 1000 total seats (deduplicated to 850 unique users)
               └─→ Filters seats: Keeps only 35 seats matching team member IDs
                   └─→ Returns: Seat[] (35 team member seats)

5. Dashboard displays metrics and seats for platform team only
```

**Total GitHub API Calls:** 13 (1 metrics + 1 team members + 10 seats pagination + 1 teams if tab clicked)

---

## Part 3: Data Flow and Transformations

### 3.1 Architecture Overview

```
┌─────────────────────────────────────────────────────────────────────┐
│                        Browser (Client-Side)                         │
│  ┌────────────────────┐                                              │
│  │   index.vue        │  Route params: /orgs/:org or /enterprises/:ent│
│  │   (Nuxt Page)      │                                              │
│  └─────────┬──────────┘                                              │
│            │ mounts                                                   │
│  ┌─────────▼──────────────────────────────────────────────────────┐ │
│  │              MainComponent.vue (Orchestrator)                   │ │
│  │  - Manages date range state                                     │ │
│  │  - Fetches metrics and seats data                               │ │
│  │  - Passes data to child components                              │ │
│  └─────┬────────────────────────────────────┬─────────────────────┘ │
│        │ $fetch('/api/metrics')             │ useFetch('/api/seats') │
└────────┼────────────────────────────────────┼───────────────────────┘
         │                                    │
         │ HTTP                               │ HTTP
         │                                    │
┌────────▼────────────────────────────────────▼───────────────────────┐
│                      Nuxt Server (Server-Side)                       │
│  ┌─────────────────────────────────────────────────────────────────┐│
│  │        Middleware: /server/middleware/github.ts                 ││
│  │  - Validates scope/org/enterprise in URL                        ││
│  │  - Injects Authorization headers into event.context             ││
│  └─────────────────────────────────────────────────────────────────┘│
│                                                                       │
│  ┌──────────────────┐  ┌──────────────────┐  ┌──────────────────┐  │
│  │ /api/metrics.ts  │  │ /api/seats.ts    │  │ /api/teams.ts    │  │
│  │ (Endpoint)       │  │ (Endpoint)       │  │ (Endpoint)       │  │
│  └────────┬─────────┘  └────────┬─────────┘  └────────┬─────────┘  │
│           │                     │                      │             │
│  ┌────────▼──────────────────────▼──────────────────────▼─────────┐ │
│  │      /server/modules/authentication.ts                          │ │
│  │  - Retrieves GitHub token (PAT or OAuth)                        │ │
│  │  - Builds headers with Authorization + API version              │ │
│  └─────────────────────────────────────────────────────────────────┘ │
│                                                                       │
│  ┌─────────────────────────────────────────────────────────────────┐│
│  │        /shared/utils/metrics-util.ts                            ││
│  │  - Builds GitHub API URL from Options                           ││
│  │  - Implements 5-minute caching with auth-bound keys             ││
│  │  - Handles mock data mode                                       ││
│  │  - Filters holidays from metrics                                ││
│  └─────────────────────────────────────────────────────────────────┘│
│           │ $fetch()                                                 │
└───────────┼──────────────────────────────────────────────────────────┘
            │
            │ HTTPS
            │
┌───────────▼──────────────────────────────────────────────────────────┐
│                    GitHub REST API (External)                         │
│  https://api.github.com                                               │
│  ┌────────────────────┐  ┌────────────────────┐  ┌────────────────┐ │
│  │ /orgs/:org/        │  │ /enterprises/:ent/ │  │ /orgs/:org/    │ │
│  │   copilot/metrics  │  │   copilot/metrics  │  │   teams        │ │
│  └────────────────────┘  └────────────────────┘  └────────────────┘ │
│  ┌────────────────────┐  ┌────────────────────┐  ┌────────────────┐ │
│  │ /orgs/:org/        │  │ /enterprises/:ent/ │  │ /orgs/:org/    │ │
│  │   copilot/billing/ │  │   copilot/billing/ │  │   teams/:slug/ │ │
│  │   seats            │  │   seats            │  │   members      │ │
│  └────────────────────┘  └────────────────────┘  └────────────────┘ │
└───────────────────────────────────────────────────────────────────────┘
```

---

### 3.2 Data Transformation Pipeline

#### Stage 1: GitHub API Response → Server Processing

**Input:** Raw GitHub API response (CopilotMetrics[])

**Server-Side Transformations (`/server/api/metrics.ts`):**

1. **Holiday Filtering** (`filterHolidaysFromMetrics`)
   - If `excludeHolidays=true`, removes metrics for holidays based on locale
   - Uses `@formkit/tempo` with locale-specific holiday calendar
   - Preserves date continuity in charts

2. **Data Normalization** (`ensureCopilotMetrics`)
   - Ensures all expected nested structures exist (prevents null errors)
   - Adds empty `languages` arrays if missing
   - Guarantees `copilot_ide_code_completions.editors` exists

3. **Legacy Format Conversion** (`convertToMetrics`)
   - Converts new usage API format to old metrics format
   - Flattens nested structure for backwards compatibility
   - Returns both formats: `{ metrics: [...], usage: [...] }`

4. **Caching** (`buildMetricsCacheKey`)
   - Creates auth-bound cache key: `{auth_hash}:{path}?{sorted_params}`
   - SHA-256 hash of Authorization header (prevents data leakage)
   - 5-minute TTL (300 seconds)
   - Example key: `a1b2c3d4e5f6g7h8:/api/metrics?githubOrg=acme&scope=organization&since=2024-01-01&until=2024-01-28`

**Example Transformation:**
```typescript
// GitHub API returns:
{
  date: "2024-01-15",
  copilot_ide_code_completions: {
    editors: [
      {
        name: "vscode",
        models: [
          {
            name: "default",
            languages: [/* data */]
          }
        ]
      }
    ]
  }
}

// After ensureCopilotMetrics():
{
  date: "2024-01-15",
  copilot_ide_code_completions: {
    total_engaged_users: 0,  // Added
    editors: [/* ... */],
    languages: []  // Added if missing
  }
}

// After convertToMetrics() (legacy format):
{
  date: "2024-01-15",
  total_suggestions_count: 12345,
  total_acceptances_count: 6789,
  total_lines_suggested: 23456,
  total_lines_accepted: 12345,
  total_active_users: 45,
  total_chat_acceptances: 234,
  total_chat_turns: 567,
  total_active_chat_users: 28,
  // Flattened language data
  breakdown: [/* ... */]
}
```

---

#### Stage 2: Server Response → Client Processing

**Input:** Server API response (MetricsApiResponse)

**Client-Side Transformations (Vue Components):**

##### MetricsViewer.vue - Language & Editor Charts

**Transformation Function:** `extractLanguageMetrics()`

```typescript
// Input: CopilotMetrics[] (28 days)
// Output: Language aggregation for charts

{
  languages: [
    {
      name: "python",
      totalSuggestions: 45678,
      totalAcceptances: 23456,
      acceptanceRate: 51.4,
      activeUsers: 35
    },
    // ... more languages
  ],
  chartData: {
    labels: ["python", "javascript", "typescript", ...],
    datasets: [
      {
        label: "Acceptance Rate",
        data: [51.4, 48.2, 62.1, ...],
        backgroundColor: ["#3776ab", "#f7df1e", "#3178c6", ...]
      }
    ]
  }
}
```

**Process:**
1. Iterate through all 28 days of metrics
2. For each day, iterate through editors → models → languages
3. Aggregate suggestions/acceptances by language name
4. Calculate acceptance rate: `(acceptances / suggestions) * 100`
5. Count unique users per language
6. Build Chart.js compatible dataset

##### AgentModeViewer.vue - GitHub.com Statistics

**Transformation Function:** `calculateGitHubStats()` (server-side in `/api/github-stats.ts`)

```typescript
// Input: CopilotMetrics[] (28 days)
// Output: Aggregated statistics across all days

{
  totalIdeCodeCompletionUsers: 485,  // Sum across 28 days
  totalIdeChatUsers: 328,
  totalDotcomChatUsers: 156,
  totalDotcomPRUsers: 89,
  totalPRSummariesCreated: 234,
  
  // Unique models (Set across all days)
  totalIdeCodeCompletionModels: 2,  // ["default", "custom-model"]
  totalIdeChatModels: 3,            // ["gpt-4", "claude-3", "custom"]
  
  // Detailed model breakdowns
  ideCodeCompletionModels: [
    {
      name: "default",
      editor: "vscode",
      model_type: "Default",
      total_engaged_users: 380
    },
    {
      name: "custom-model",
      editor: "vscode",
      model_type: "Custom",
      total_engaged_users: 105
    }
  ],
  
  // Chart data for visualization
  agentModeChartData: {
    labels: ["2024-01-01", "2024-01-02", ...],
    datasets: [
      {
        label: "IDE Code Completions",
        data: [38, 42, 45, ...],  // Users per day
        borderColor: "rgb(75, 192, 192)"
      },
      // ... 3 more datasets for chat/PR
    ]
  }
}
```

**Process:**
1. **Count Users:** Sum `total_engaged_users` across all days for each feature
2. **Find Unique Models:** Use `Set` to track unique model names across date range
3. **Aggregate by Model:** Group users by model name + editor combination
4. **Build Time-Series:** Create array of daily values for line charts
5. **Calculate Summaries:** Sum PR summaries created across all repositories

##### SeatsAnalysisViewer.vue - Seat Activity Analysis

**Transformation Function:** `processSeatData()` (client-side)

```typescript
// Input: Seat[] (150 seats)
// Output: Categorized seats with activity analysis

{
  activeSeats: [
    {
      login: "johndoe",
      last_activity_at: "2024-01-28T10:30:00Z",
      last_activity_editor: "vscode",
      daysSinceActivity: 0,
      status: "active"
    }
  ],
  inactiveSeats: [
    {
      login: "janedoe",
      last_activity_at: "2024-01-01T14:20:00Z",
      last_activity_editor: "intellij",
      daysSinceActivity: 27,
      status: "inactive-30days"
    }
  ],
  neverUsedSeats: [
    {
      login: "newuser",
      last_activity_at: null,
      created_at: "2024-01-15T09:00:00Z",
      status: "never-used"
    }
  ],
  summary: {
    total: 150,
    active: 120,
    inactive7days: 15,
    inactive30days: 10,
    neverUsed: 5
  }
}
```

**Process:**
1. **Calculate Days Since Activity:** `(now - last_activity_at) / (1000 * 60 * 60 * 24)`
2. **Categorize Seats:**
   - Active: Activity within 7 days
   - Inactive 7-30 days: Activity between 7-30 days ago
   - Inactive 30+ days: Activity more than 30 days ago
   - Never used: `last_activity_at === null`
3. **Sort by Activity:** Most recent first
4. **Calculate Percentages:** `(category_count / total_seats) * 100`

##### TeamsComponent.vue - Multi-Team Comparison

**Transformation Function:** `aggregateTeamMetrics()` (client-side)

```typescript
// Input: Map<team_slug, CopilotMetrics[]>
// Output: Comparative analysis across teams

{
  teams: ["frontend-team", "backend-team", "qa-team"],
  
  comparisons: {
    acceptanceRates: {
      labels: ["2024-01-01", "2024-01-02", ...],
      datasets: [
        {
          label: "frontend-team",
          data: [48.5, 52.1, 49.8, ...],  // Acceptance rate per day
          borderColor: "#FF6384"
        },
        {
          label: "backend-team",
          data: [55.2, 58.3, 54.1, ...],
          borderColor: "#36A2EB"
        },
        {
          label: "qa-team",
          data: [42.1, 45.7, 43.2, ...],
          borderColor: "#FFCE56"
        }
      ]
    },
    
    activeUsersComparison: {
      labels: ["frontend-team", "backend-team", "qa-team"],
      datasets: [
        {
          label: "Average Active Users",
          data: [32, 28, 15],  // Average across date range
          backgroundColor: ["#FF6384", "#36A2EB", "#FFCE56"]
        }
      ]
    },
    
    featureAdoption: {
      teamName: "frontend-team",
      codeCompletionUsers: 30,
      chatUsers: 25,
      dotcomChatUsers: 12,
      prSummaryUsers: 8
    }
  }
}
```

**Process:**
1. **Load Multiple Teams:** Fetch `/api/metrics` for each selected team in parallel
2. **Normalize Date Ranges:** Ensure all teams have same date array
3. **Calculate Per-Team Metrics:**
   - Daily acceptance rate: `(acceptances / suggestions) * 100`
   - Daily active users: `total_engaged_users`
   - Feature breakdown: Count users for each feature (completions, chat, PR)
4. **Create Comparative Datasets:** Build Chart.js datasets with team-specific colors
5. **Aggregate Statistics:** Calculate averages, totals, and percentages per team

---

### 3.3 Data Joining and Correlation

#### Seats + Metrics Correlation

**Scenario:** Identify users with seats but no activity

**Join Logic:**
```typescript
// Step 1: Get all seats from /api/seats
const seats = await $fetch('/api/seats'); // 150 seats

// Step 2: Get metrics from /api/metrics
const metrics = await $fetch('/api/metrics'); // 28 days

// Step 3: Extract active users from metrics
const activeUserIds = new Set();
metrics.forEach(day => {
  day.copilot_ide_code_completions?.editors?.forEach(editor => {
    editor.models?.forEach(model => {
      // Note: Metrics don't include user IDs, only counts
      // This correlation is done via last_activity_at on seats
    });
  });
});

// Step 4: Identify inactive seats
const inactiveSeats = seats.filter(seat => {
  if (!seat.last_activity_at) return true; // Never used
  const daysSince = (Date.now() - new Date(seat.last_activity_at)) / (1000*60*60*24);
  return daysSince > 30; // Inactive for 30+ days
});

// Result: List of seats to potentially reclaim
```

**Key Point:** Metrics aggregate user counts but don't expose individual user IDs. Seat activity is tracked separately via `last_activity_at` timestamps.

---

#### Team Members + Seats Filtering

**Scenario:** Show seats for a specific team only

**Join Logic:**
```typescript
// Step 1: Get team members from /api/teams/{team_slug}/members
const teamMembers = await fetchAllTeamMembers(); 
// Returns: [{ id: 123, login: "johndoe" }, ...]

// Step 2: Get all organization seats
const allSeats = await $fetch('/api/seats'); // 500 seats (entire org)

// Step 3: Filter seats by team member IDs
const teamSeats = allSeats.filter(seat => 
  teamMembers.some(member => member.id === seat.assignee.id)
);

// Result: Only seats assigned to team members
// Example: 500 org seats → 35 team member seats
```

**Implementation:** `/server/api/seats.ts` lines 115-161

---

#### Metrics Aggregation Across Teams

**Scenario:** Compare metrics for 3 teams side-by-side

**Join Logic:**
```typescript
// Step 1: Fetch metrics for each team in parallel
const [team1Metrics, team2Metrics, team3Metrics] = await Promise.all([
  $fetch('/api/metrics?scope=team-organization&githubTeam=team1'),
  $fetch('/api/metrics?scope=team-organization&githubTeam=team2'),
  $fetch('/api/metrics?scope=team-organization&githubTeam=team3')
]);

// Step 2: Normalize dates (ensure all have same date array)
const allDates = new Set([
  ...team1Metrics.map(m => m.date),
  ...team2Metrics.map(m => m.date),
  ...team3Metrics.map(m => m.date)
]);

// Step 3: Build comparison datasets
const comparisonData = Array.from(allDates).map(date => {
  const team1Day = team1Metrics.find(m => m.date === date);
  const team2Day = team2Metrics.find(m => m.date === date);
  const team3Day = team3Metrics.find(m => m.date === date);
  
  return {
    date,
    team1ActiveUsers: team1Day?.total_active_users || 0,
    team2ActiveUsers: team2Day?.total_active_users || 0,
    team3ActiveUsers: team3Day?.total_active_users || 0
  };
});

// Result: Unified dataset showing all teams per day
```

**Implementation:** `/app/components/TeamsComponent.vue`

---

### 3.4 Caching Strategy

#### Server-Side Caching (5-minute TTL)

**Location:** `/shared/utils/metrics-util.ts`

**Cache Key Structure:**
```
{auth_fingerprint}:{path}?{sorted_query_params}

Example:
a1b2c3d4e5f6g7h8:/api/metrics?githubOrg=acme&scope=organization&since=2024-01-01&until=2024-01-28
```

**Why Auth-Bound?**
- Prevents data leakage between users/tokens
- Different tokens may have different access levels (org vs. enterprise)
- Ensures users only see data they're authorized to access

**Cache Invalidation:**
- Time-based: 5 minutes (300 seconds)
- On error: Cleared immediately to prevent stale data on retry
- On date change: Different query params = different cache key

**Implementation:**
```typescript
const cache = new Map<string, CacheData>();

interface CacheData {
  data: CopilotMetrics[];
  valid_until: number; // Unix timestamp
}

// Check cache
const cachedData = cache.get(cacheKey);
if (cachedData && cachedData.valid_until > Date.now() / 1000) {
  return cachedData.data; // Cache hit
}

// Fetch from GitHub API
const freshData = await $fetch(githubApiUrl);

// Store in cache
cache.set(cacheKey, {
  data: freshData,
  valid_until: Math.floor(Date.now() / 1000) + 300 // 5 min TTL
});
```

---

#### Client-Side Caching (Component Level)

**Location:** `/app/components/AgentModeViewer.vue`

**Cache Strategy:**
```typescript
// Cache previous request to prevent duplicate calls
const previousMetrics = ref<string>('');

const fetchStats = debounce(async () => {
  const metricsHash = JSON.stringify(options);
  
  // Check if already fetched
  if (metricsHash === previousMetrics.value) {
    return; // Skip duplicate request
  }
  
  // Fetch new data
  const stats = await $fetch('/api/github-stats', { params: options });
  previousMetrics.value = metricsHash;
}, 150); // 150ms debounce
```

**Why Debounce?**
- Prevents rapid-fire API calls during date range changes
- Waits for user to finish selecting dates before fetching
- Reduces server load and improves UX

---

### 3.5 Error Handling and Fallbacks

#### Authentication Errors

```typescript
// If no auth provided
if (!authHeader) {
  throw new MetricsError('No Authentication provided', 401);
}

// Frontend receives 401 → shows login prompt or token error
```

#### GitHub API Errors

```typescript
try {
  const response = await $fetch(apiUrl, { headers });
} catch (error) {
  // Extract status code from error
  const statusCode = error.statusCode || 500;
  
  // Common errors:
  // 401: Invalid or expired token
  // 403: Rate limit exceeded or insufficient permissions
  // 404: Organization/enterprise not found
  // 422: Invalid date format
  
  throw new MetricsError(`Error fetching metrics: ${error.message}`, statusCode);
}
```

#### Mock Data Fallback

```typescript
// For development/testing without GitHub access
if (options.isDataMocked) {
  const mockData = JSON.parse(fs.readFileSync('public/mock-data/sample.json'));
  
  // Make mock data dynamic based on date range
  const adjustedData = updateMockDataDates(mockData, since, until);
  
  return adjustedData;
}
```

---

## Summary

### Key Architectural Patterns

1. **Parallel Data Loading:** Metrics and seats fetched simultaneously on page load
2. **Progressive Enhancement:** Components load additional data only when tabs are activated
3. **Smart Caching:** Auth-bound caching prevents data leakage while improving performance
4. **Client-Side Aggregation:** Heavy transformations (charts, statistics) done in browser
5. **Pagination Handling:** Automatic page fetching for seats and teams (up to 100 per page)
6. **Team Filtering:** Members fetched first, then seats filtered by member IDs
7. **Mock Mode:** Complete mocked data support for development without GitHub access

### Performance Characteristics

- **Initial Load:** 2-3 seconds (1 metrics call + 1-10 seats pagination calls)
- **Cache Hit:** <100ms (served from memory)
- **Tab Switch:** 0-2 seconds (depends on whether data is cached)
- **Date Range Change:** 1-2 seconds (new metrics fetch + cache store)
- **Team Comparison (3 teams):** 2-4 seconds (parallel metrics fetches)

### Data Volume

- **Metrics Response:** ~50-200 KB per 28 days (depends on org size)
- **Seats Response:** ~5-10 KB per 100 seats
- **Teams Response:** ~2-5 KB per 100 teams
- **Typical Full Load:** 100-500 KB total (compressed over HTTP/2)

---

## Appendix: Code References

### Server Endpoints
- `/server/api/metrics.ts` - Main metrics endpoint (lines 1-29)
- `/server/api/seats.ts` - Seats endpoint with pagination (lines 87-164)
- `/server/api/teams.ts` - Teams listing with Link header pagination (lines 30-102)
- `/server/api/github-stats.ts` - Aggregated statistics endpoint (lines 22-237)

### Data Utilities
- `/shared/utils/metrics-util.ts` - Metrics fetching and caching (lines 72-148)
- `/server/modules/authentication.ts` - Token and header management (lines 19-63)
- `/app/model/Options.ts` - URL building and configuration (lines 267-389)

### Vue Components
- `/app/components/MainComponent.vue` - Orchestrator component
- `/app/components/MetricsViewer.vue` - Language/editor charts
- `/app/components/AgentModeViewer.vue` - GitHub.com statistics
- `/app/components/SeatsAnalysisViewer.vue` - Seat activity analysis
- `/app/components/TeamsComponent.vue` - Multi-team comparison

### Data Models
- `/app/model/Copilot_Metrics.ts` - TypeScript interfaces for metrics
- `/app/model/Seat.ts` - Seat assignment model
- `/app/types/metricsApiResponse.ts` - API response types

---

*Generated: 2024-01-29*  
*Version: Based on copilot-metrics-viewer v2.1.0*
