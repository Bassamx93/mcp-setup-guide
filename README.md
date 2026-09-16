# MCP Setup Guide for GitHub Copilot CLI

A complete, practical quick-start for setting up and managing Model Context Protocol (MCP) servers with GitHub Copilot CLI.

---

## 1) What is MCP?

**Model Context Protocol (MCP)** is an open protocol that lets tools and data sources expose capabilities to AI clients in a standard way.

With Copilot CLI, MCP servers let you:
- Connect local tools (stdio servers)
- Connect remote services (HTTP/SSE servers)
- Control which tools Copilot can call
- Share repeatable team configuration in-repo

Think of MCP as the bridge between Copilot and your external systems.

---

## 2) Quick Start (5 minutes)

This is the fastest working setup using a local stdio server.

### Prerequisites
- GitHub Copilot CLI installed and signed in
- Node.js (for `npx` examples)

### Step-by-step walkthrough

```bash
# 1) Open Copilot interactive mode
copilot
```

Inside interactive mode:
```text
/mcp add
```

Then fill:
- **Name**: `memory`
- **Transport**: `stdio`
- **Command**: `npx`
- **Args**: `-y @modelcontextprotocol/server-memory`
- **Tools**: `*`
- Save (`Ctrl+S`)

Verify:
```text
/mcp
/mcp show memory
```

You are now running your first MCP server.

---

## 3) All Configuration Methods (5 ways)

### Method A: Interactive form (`/mcp add`) — easiest

Best for first-time setup.

```text
/mcp add
```

Use form fields, save, and changes apply immediately.

### Method B: Terminal command (`copilot mcp add`) — scriptable

### Local stdio server
```bash
copilot mcp add memory -- npx -y @modelcontextprotocol/server-memory
```

### Local server with environment variables
```bash
copilot mcp add github-local \
  --env GITHUB_PERSONAL_ACCESS_TOKEN=YOUR_PAT \
  -- docker run -i --rm -e GITHUB_PERSONAL_ACCESS_TOKEN \
  ghcr.io/github/github-mcp-server
```

### Remote HTTP server
```bash
copilot mcp add --transport http notion https://mcp.notion.com/mcp
```

### Remote HTTP with auth header
```bash
copilot mcp add --transport http stripe https://mcp.stripe.com \
  --header "Authorization: Bearer <TOKEN>"
```
Use an auth scheme prefix in this header (typically `Bearer`) before your token value.

### Remote SSE server
```bash
copilot mcp add --transport sse analytics-sse https://example.com/mcp/sse \
  --header "Authorization: Bearer <TOKEN>"
```

### Method C: User config file (`~/.copilot/mcp-config.json`)

Best for bulk edits and backups.

```json
{
  "mcpServers": {
    "memory": {
      "type": "local",
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-memory"],
      "tools": ["*"],
      "timeout": 60000
    },
    "notion": {
      "type": "http",
      "url": "https://mcp.notion.com/mcp",
      "headers": {},
      "tools": ["search", "pages.read"]
    }
  }
}
```

### Method D: Repository config (`.mcp.json` or `.github/mcp.json`)

Best for team/project defaults.

```json
{
  "mcpServers": {
    "project-memory": {
      "type": "local",
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-memory"],
      "tools": ["*"]
    }
  }
}
```

**Config precedence and merge behavior (from GitHub Docs)**  
Source: https://docs.github.com/en/copilot/how-tos/copilot-cli/customize-copilot/add-mcp-servers#adding-per-repository-mcp-servers

- Copilot CLI reads project-level MCP config from `.mcp.json` and `.github/mcp.json`.
- If both files are present in the same directory, `.mcp.json` is used first.
- If server names conflict across project files in different directories, the definition closer to your current working directory wins.
- Project-level server definitions override user-level definitions in `~/.copilot/mcp-config.json`.

### Method E: MCP Registry search (`/mcp search`) — experimental

In interactive mode:
```text
/experimental on
/mcp search github
```

Use this to discover install-ready MCP servers quickly.

---

## 4) Troubleshooting Guide

### Server does not appear in `/mcp`
- Re-run `copilot mcp list`
- Validate JSON syntax if manually edited
- Ensure you saved interactive form (`Ctrl+S`)

### Command not found / local server fails to start
- Confirm runtime exists (`node -v`, `python --version`, `docker --version`)
- Use absolute command path if shell PATH differs

### Authentication failures (401/403)
- Check token is valid and unexpired
- Confirm the Authorization header includes an auth scheme prefix (such as Bearer) followed by a token.
- Ensure env variable name exactly matches server docs

### Timeouts or slow responses
- Increase timeout:
```bash
copilot mcp edit NAME --timeout 120000
```
- Reduce enabled tools to only what you need
- Prefer local network endpoints when possible

### Remote server unreachable
- Verify URL, TLS cert, and firewall/proxy rules
- Test endpoint with curl (include auth for protected endpoints):
```bash
curl -i https://mcp.notion.com/mcp \
  -H "Authorization: Bearer <TOKEN>"
```

### Monitoring and debugging tips
- Use `/mcp` (interactive) or `copilot mcp list` to confirm server status quickly
- Use `/mcp show <name>` or `copilot mcp show <name>` to verify effective config
- Run a small test prompt right after setup to validate tool availability
- Keep timeouts and tool scopes tight to reduce failure blast radius

---

## 5) Real-World Server Setups

> Replace placeholder tokens before use.

### GitHub MCP (local Docker server)
```bash
copilot mcp add github \
  --env GITHUB_PERSONAL_ACCESS_TOKEN=YOUR_GITHUB_PAT \
  -- docker run -i --rm -e GITHUB_PERSONAL_ACCESS_TOKEN \
  ghcr.io/github/github-mcp-server
```

### Stripe MCP
```bash
copilot mcp add --transport http stripe https://mcp.stripe.com \
  --header "Authorization: Bearer <TOKEN>"
```

### Notion MCP
```bash
copilot mcp add --transport http notion https://mcp.notion.com/mcp \
  --header "Authorization: Bearer <TOKEN>"
```

### Memory MCP (local stdio)
```bash
copilot mcp add memory -- npx -y @modelcontextprotocol/server-memory
```

### Generic Docker-based local server pattern
```bash
copilot mcp add my-server \
  --env API_KEY=YOUR_KEY \
  -- docker run -i --rm -e API_KEY ghcr.io/example/example-mcp-server:latest
```

---

## 6) Best Practices

### Security
- Never commit secrets in `.mcp.json` or docs
- Prefer environment variables over hard-coded tokens
- Use least-privilege tokens per server
- Rotate credentials regularly

### Performance
- Enable only required tools (avoid `*` in production if possible)
- Increase timeout only for genuinely slow backends
- Remove unused servers to reduce overhead

### Organization
- Use clear names (`github-prod`, `notion-team`, `stripe-test`)
- Keep personal servers in `~/.copilot/mcp-config.json`
- Keep team/project servers in repo config

### Reliability
- Add health checks with simple test prompts after setup
- Track server-specific limits (rate limits, payload limits)

---

## 7) CLI Reference

### Add / discover
```bash
copilot mcp add NAME -- COMMAND [ARGS...]
copilot mcp add --transport http NAME URL
copilot mcp list
```

Interactive equivalents:
```text
/mcp add
/mcp search <query>
/mcp
```

### Inspect / manage
```bash
copilot mcp show NAME
copilot mcp edit NAME
copilot mcp delete NAME
copilot mcp enable NAME
copilot mcp disable NAME
```

Interactive equivalents:
```text
/mcp show NAME
/mcp edit NAME
/mcp delete NAME
/mcp enable NAME
/mcp disable NAME
```

### Common options
- `--transport stdio|http|sse`
- `--env KEY=VALUE`
- `--header "Name: Value"`
- `--tools tool1,tool2` or `--tools *`
- `--timeout <milliseconds>`

---

## 8) FAQ

### Do I need to restart Copilot CLI after adding a server?
Usually no. Changes apply immediately after save/update.

### What is the difference between stdio and HTTP/SSE?
- **stdio**: starts a local process and communicates via stdin/stdout
- **HTTP/SSE**: connects to a remote MCP endpoint

### Should I use user-level or repo-level config?
- **User-level** for personal servers and secrets
- **Repo-level** for team-shared defaults

### Can I use multiple MCP servers at once?
Yes. Add multiple servers and selectively enable/disable them.

### Where should secrets go?
Use environment variables or secret managers. Do not commit plaintext secrets.

### How do I debug a misbehaving server quickly?
1. `copilot mcp show <name>`
2. Verify command/URL/auth values
3. Reduce tools list to minimum
4. Increase timeout and retry
5. Check network/runtime prerequisites

---

## 9) Copy-Paste Templates

### Minimal local server template
```bash
copilot mcp add <name> -- <command> <arg1> <arg2>
```

### Minimal remote server template
```bash
copilot mcp add --transport http <name> <url>
```

### Auth via header template
```bash
copilot mcp add --transport http <name> <url> \
  --header "Authorization: Bearer <TOKEN>"
```
Header values should include an auth scheme prefix (for example, `Bearer`) and then your token.

### Auth via environment variable template
```bash
copilot mcp add <name> \
  --env API_KEY=<token> \
  -- <command> [args...]
```

### JSON template
```json
{
  "mcpServers": {
    "<name>": {
      "type": "http",
      "url": "https://example.com/mcp",
      "headers": {
        "Authorization": "Bearer <TOKEN>"
      },
      "tools": ["*"],
      "timeout": 60000
    }
  }
}
```

### SSE JSON template
```json
{
  "mcpServers": {
    "<name>": {
      "type": "sse",
      "url": "https://example.com/mcp/sse",
      "headers": {
        "Authorization": "Bearer <TOKEN>"
      },
      "tools": ["*"],
      "timeout": 60000
    }
  }
}
```
