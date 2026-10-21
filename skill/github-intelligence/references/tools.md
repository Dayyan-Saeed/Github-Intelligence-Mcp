# Tool reference: GitHub Intelligence MCP

All tools are read-only and take `owner` + `repo` as plain strings (e.g.
`facebook` / `react`). Outputs are validated structured JSON.

## Analysis tools

### `analyze_repository`

Deterministic health assessment — no LLM judgement.

| Param | Type | Default | Notes |
|---|---|---|---|
| owner | string | required | Repository owner |
| repo | string | required | Repository name |

Returns `RepositoryHealthResponse`:

- `overall_score` (0–100), `grade` (A/B/C/D/F)
- `components[]`: each of activity, issue_health, pr_health, contributor_health, release_activity, documentation with `score`, `weight`, and `evidence`
- `repository`: stars, forks, open issues, language, license, timestamps

Grades: A ≥85, B ≥70, C ≥55, D ≥40, F <40.
Weights: activity .25, issue_health .20, pr_health .20, contributor_health .15, release_activity .10, documentation .10.

### `find_maintenance_risks`

| Param | Type | Default |
|---|---|---|
| owner | string | required |
| repo | string | required |

Returns `MaintenanceRiskResponse` with `risks[]`, each containing
`category`, `severity` (low/medium/high), `title`, `description`, and
`evidence`. Categories: `stale_issues`, `stale_prs`, `inactive`,
`bus_factor`, `concentration`, `release_gap`.

### `compare_repositories`

| Param | Type | Default |
|---|---|---|
| owner_a | string | required |
| repo_a | string | required |
| owner_b | string | required |
| repo_b | string | required |

Returns `RepositoryComparisonResponse`: `dimensions[]` (each with a metric
comparison and a `winner` or tie), plus both repos' health scores. Use for
"which library should we adopt" questions.

## Data tools

### `get_repository`

| Param | Type | Default |
|---|---|---|
| owner | string | required |
| repo | string | required |

Returns stars, forks, watchers, open issues, language, license, default
branch, created/updated timestamps, description, homepage URL.

### `search_repositories`

| Param | Type | Default |
|---|---|---|
| query | string | required | Free-form GitHub search syntax (`language:rust stars:>1000`) |
| sort | string \| null | null | `stars`, `forks`, `help-wanted-issues`, `updated` |
| order | string | `desc` | `asc` or `desc` |
| limit | int | 30 | |

### `get_recent_commits`

| Param | Type | Default |
|---|---|---|
| owner | string | required |
| repo | string | required |
| sha | string \| null | null | Branch or commit SHA (default branch if null) |
| limit | int | 30 | |

Returns sha, message, author, author/committer timestamps, HTML URL.

### `get_issues`

| Param | Type | Default |
|---|---|---|
| owner | string | required |
| repo | string | required |
| state | string | `open` | `open`, `closed`, `all` |
| labels | string[] \| null | null | Filter by label names |
| sort | string \| null | null | `created`, `updated`, `comments` |
| direction | string | `desc` | `asc` or `desc` |
| limit | int | 30 | |

Pull requests are automatically excluded from results.

### `get_pull_requests`

| Param | Type | Default |
|---|---|---|
| owner | string | required |
| repo | string | required |
| state | string | `open` | `open`, `closed`, `all` |
| sort | string \| null | null | `created`, `updated`, `popularity`, `long-running` |
| direction | string | `desc` | `asc` or `desc` |
| limit | int | 30 | |

Returns number, title, state, author, timestamps, draft flag, labels.

### `get_contributors`

| Param | Type | Default |
|---|---|---|
| owner | string | required |
| repo | string | required |
| limit | int | 30 | |

Returns username and contribution count, ordered by contributions. High
concentration in the top entry indicates bus-factor risk.

### `get_releases`

| Param | Type | Default |
|---|---|---|
| owner | string | required |
| repo | string | required |
| limit | int | 30 | |

Returns tag name, release name, published timestamp, draft/prerelease flags.

## Example calls

```
analyze_repository(owner="facebook", repo="react")
find_maintenance_risks(owner="owner", repo="legacy-app")
compare_repositories(owner_a="expressjs", repo_a="express", owner_b="honojs", repo_b="hono")
search_repositories(query="mcp server language:python stars:>100", sort="stars", limit=10)
get_issues(owner="owner", repo="repo", state="open", labels=["bug"], limit=50)
```

## Resources (URI templates)

The server also exposes MCP resources for direct reads:

- `github-intelligence://repo/{owner}/{repo}`
- `github-intelligence://issues/{owner}/{repo}`
- `github-intelligence://pulls/{owner}/{repo}`
- `github-intelligence://commits/{owner}/{repo}`
- `github-intelligence://contributors/{owner}/{repo}`
- `github-intelligence://releases/{owner}/{repo}`

## Prompts

Five prompt templates ship with the server for health review, risk triage,
adoption due diligence, repo comparison, and contribution triage.

## Error messages

| Error | Meaning |
|---|---|
| `Repository 'x/y' was not found or is inaccessible` | Wrong name or private repo without access |
| `GitHub rejected the configured credentials` | Invalid/expired `GITHUB_TOKEN` |
| `does not have permission to access this resource` | Token lacks Contents/Metadata read access |
| Rate limit message | Too many requests — wait and retry later |
