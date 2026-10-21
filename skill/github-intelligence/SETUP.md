# Setup: GitHub Intelligence MCP Server

The skill works with the open-source **GitHub Intelligence MCP server**
(read-only GitHub repository intelligence: health scores, maintenance risks,
comparisons).

Repository: https://github.com/Dayyan-Saeed/Github-Intelligence-Mcp
Live dashboard: https://github-intelligence-mcp-ashy.vercel.app/

## 1. Install the server

With [uv](https://docs.astral.sh/uv/) (recommended):

```bash
uv tool install github-intelligence-mcp
```

Or with pip:

```bash
pip install github-intelligence-mcp
```

Or run from source:

```bash
git clone https://github.com/Dayyan-Saeed/Github-Intelligence-Mcp.git
cd Github-Intelligence-Mcp
uv sync
```

## 2. Create a GitHub token

1. Open https://github.com/settings/tokens (Settings → Developer settings → Fine-grained tokens)
2. Generate a new fine-grained token:
   - **Repository access:** All repositories (public is enough for analyzing any public repo)
   - **Permissions:** Contents: Read-only, Metadata: Read-only
   - **Expiration:** No expiration (or long-lived) so it does not silently break
3. Copy the token — you will need it in the next step

The token is only used for read-only GitHub REST API calls. It is stored in
your environment, never in this skill, and never appears in responses.

## 3. Register the MCP server

Copy the matching block from `mcp-config.json` into your agent's MCP config.

**Claude Code** (`claude mcp add`):

```bash
claude mcp add github-intelligence \
  --env GITHUB_TOKEN=your-token-here \
  -- github-intelligence-mcp
```

Or add to `.mcp.json` in your project root:

```json
{
  "mcpServers": {
    "github-intelligence": {
      "command": "github-intelligence-mcp",
      "env": { "GITHUB_TOKEN": "your-token-here" }
    }
  }
}
```

**Cursor** (`.cursor/mcp.json` in your project, or `~/.cursor/mcp.json` global):

```json
{
  "mcpServers": {
    "github-intelligence": {
      "command": "github-intelligence-mcp",
      "env": { "GITHUB_TOKEN": "your-token-here" }
    }
  }
}
```

**Codex CLI / OpenCode / other MCP clients:** same JSON shape — a `command`
entry running `github-intelligence-mcp` with `GITHUB_TOKEN` in `env`.

If you installed from source with uv, replace the command with:

```json
{ "command": "uv", "args": ["run", "--directory", "/path/to/Github-Intelligence-Mcp", "github-intelligence-mcp"] }
```

## 4. Verify

Restart the agent, then ask:

> Analyze facebook/react

The agent should call `analyze_repository` and return a health score with a
letter grade. If tools are missing, re-check the config path and that the
`GITHUB_TOKEN` environment variable is set where the server process starts.

## 5. Optional: REST API and dashboard

The project also ships a FastAPI wrapper and a Next.js dashboard:

- API: `uv run python -m github_intelligence_mcp.api` → http://localhost:8000
- Dashboard: see the repo README for local dev and the hosted demo

## Troubleshooting

| Symptom | Fix |
|---|---|
| `GitHub rejected the configured credentials` | Token invalid or revoked — regenerate (step 2) |
| `does not have permission` | Token lacks Contents/Metadata read permissions |
| Rate limit errors | Token missing → unauthenticated 60 req/hr; add a token |
| Tools not listed | Server not registered or agent not restarted |
| `command not found` | Install failed — check `uv tool`/`pip` PATH |
