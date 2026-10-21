---
name: github-intelligence
description: Investigates and analyzes GitHub repositories with deterministic health scores, maintenance-risk detection, and repo comparisons via the GitHub Intelligence MCP server. Use when the user asks to analyze a repo, check repository health, find stale issues, assess maintenance risk, evaluate a library before adopting it, compare two repos, investigate commit activity, or audit contributor/bus-factor risk.
when_to_use: |
  - User asks to "analyze <owner>/<repo>", "how healthy is this repo", or "is this project maintained"
  - User asks to find stale issues, stale pull requests, or maintenance risks in a repo
  - User wants to compare two repositories before choosing a dependency
  - User asks about a repo's contributors, bus factor, release cadence, or commit activity
  - User says "due diligence on <repo>" or "should we adopt this library"
  - Do NOT use for: searching code inside a repo, cloning repos, or GitHub write operations (this skill is read-only)
---

# GitHub Intelligence

Deterministic GitHub repository intelligence for AI agents. Scores, risks, and
comparisons are computed in code from the GitHub REST API — not guessed by an
LLM. Never fabricate metrics; always pull them from the tools below.

## Prerequisite

This skill requires the **GitHub Intelligence MCP server** to be connected.
Check for its tools (`get_repository`, `analyze_repository`, etc.) before
following this guide.

If the tools are not available, ask the user to install the server by
following `SETUP.md` in this skill's folder. The short version:

1. Install: `uv tool install github-intelligence-mcp` (or `pip install github-intelligence-mcp`)
2. Create a GitHub fine-grained PAT with **Contents + Metadata: Read-only**
3. Register the server in the agent's MCP config (template: `mcp-config.json`)
4. Set `GITHUB_TOKEN` in the environment, then restart the agent

## Tool routing

Match the user's intent to the right tool. Never call all tools when one
suffices.

| User intent | Tool(s) | Notes |
|---|---|---|
| "Analyze <repo>", "is it healthy" | `analyze_repository` | One call returns full health report |
| "Find maintenance risks", "stale issues" | `find_maintenance_risks` | Fixed severity thresholds (see below) |
| "Compare A vs B", "which library" | `compare_repositories` | Both repos in one call |
| "Repo basics", stars, license, description | `get_repository` | Lightweight metadata |
| "Recent activity", "is it active" | `get_recent_commits` + `analyze_repository` | Commits feed the activity score |
| "Who maintains this", "bus factor" | `get_contributors` + `analyze_repository` | Concentration is scored automatically |
| "Open issues/PRs", triage | `get_issues` / `get_pull_requests` | Use `state`, `labels`, `limit` params |
| "Releases", "version cadence" | `get_releases` | Release activity is one health component |
| "Find a repo for X" | `search_repositories` | GitHub search syntax in the `query` param |
| Full deep-dive report | use all relevant tools above | Then synthesize into one report |

Full parameter and output schemas: `references/tools.md`.

## Interpreting scores

`analyze_repository` returns an overall score (0–100), a letter grade, and six
weighted components:

| Component | Weight | What it measures |
|---|---|---|
| activity | 25% | Commit volume and active authors (last 30 days) |
| issue_health | 20% | Share of stale open issues (default: >90 days) |
| pr_health | 20% | Share of stale open PRs (default: >30 days) |
| contributor_health | 15% | Number of contributors (bus-factor proxy) |
| release_activity | 10% | Releases in the last 90 days |
| documentation | 10% | License presence and docs signals |

Grades: **A** ≥85, **B** ≥70, **C** ≥55, **D** ≥40, **F** <40.

When reporting, always cite the numbers from the tool output. If a component
score is missing or an API call failed, say so — do not invent a value.

## Risk severity

`find_maintenance_risks` classifies each risk as low / medium / high:

- **stale_issues**: low <5, medium ≥5, high ≥10 and ≥50% of open issues
- **stale_prs**: low <5, medium ≥5, high ≥15
- **inactive**: medium ≥180 days since last activity, high ≥365 days
- **bus_factor**: high if 1 dominant contributor, medium if 2
- **concentration**: medium if one contributor has ≥80% of commits
- **release_gap**: medium ≥365 days since last release, high ≥730 days

## Output conventions

When the user asks for an analysis, structure your answer as:

1. **Verdict** — one sentence: grade + whether the repo is worth adopting/contributing to
2. **Key numbers** — overall score, stars, open issues, last commit date, contributor count
3. **Risks** — bulleted, severity-tagged (only medium/high unless asked)
4. **Recommendation** — concrete next action (adopt, avoid, contribute where, ask maintainer about X)

Keep it concise. Link to the live dashboard (`https://github-intelligence-mcp-ashy.vercel.app/repo/<owner>/<repo>`) when the user wants a visual view.

## Boundaries

- **Read-only.** Never create issues, PRs, comments, or any write operation.
- **No secrets.** Never print, log, or include the `GITHUB_TOKEN` value in responses.
- **No fabrication.** Every metric must come from tool output. If a tool errors, report the error.
- **Rate-limit aware.** If the API reports rate limiting, tell the user and wait rather than retry-storming.
