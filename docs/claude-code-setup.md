# Hermes + Claude Code Integration

This document describes how Hermes can be set up to work with Claude Code as an MCP (Model Context Protocol) server.

## Overview

Hermes is a Rust-native knowledge graph engine that provides pointer-based RAG (Retrieval-Augmented Generation) for AI-assisted development. Instead of returning full file contents, it returns lightweight pointers that reference specific code locations, fetching full content only when needed.

## Architecture

```
┌─────────────────┐     stdio/JSON-RPC      ┌─────────────────┐
│   Claude Code   │ ◄─────────────────────► │  Hermes Engine  │
│   (MCP Client)  │                         │  (MCP Server)   │
└─────────────────┘                         └────────┬────────┘
                                                     │
                                            ┌────────▼────────┐
                                            │   SQLite DB     │
                                            │  (.hermes.db)   │
                                            └─────────────────┘
```

### Why MCP?

MCP (Model Context Protocol) is a standardized protocol for AI assistants to communicate with external tools. Benefits:

- **Standardized interface**: Claude Code natively supports MCP servers
- **Process isolation**: Hermes runs as a separate process, preventing crashes from affecting Claude Code
- **Language agnostic**: Any language can implement an MCP server (Hermes uses Rust for performance)
- **Stdio transport**: Simple, reliable communication over stdin/stdout

### Why Pointer-Based RAG?

Traditional RAG returns full file contents, which:
- Consumes many tokens (expensive)
- Fills context window quickly
- Often includes irrelevant code

Pointer-based RAG returns:
- File path + line numbers
- Brief summary/preview
- Relevance score
- Node ID for fetching full content

**Typical savings: 85-95% fewer tokens** compared to full-file retrieval.

## Implementation

### 1. Build the Binary

The Hermes engine is written in Rust. Building produces a single static binary:

```bash
cd /path/to/hermes
cargo build --release
```

Output: `target/release/Hermes`

### 2. MCP Server Configuration

Claude Code uses the `claude mcp` CLI commands to manage MCP servers. There are three scopes:

| Scope | Storage | Use Case |
|-------|---------|----------|
| `local` (default) | `~/.claude.json` per project | Personal servers for a specific project |
| `project` | `.mcp.json` in project root | Team-shared servers (checked into git) |
| `user` | `~/.claude.json` global | Personal servers across all projects |

#### Option A: User Scope (Recommended)

Make hermes available across all your projects:

```bash
claude mcp add hermes --scope user --transport stdio \
  -e HERMES_PROJECT_ROOT=. \
  -- /path/to/hermes/target/release/Hermes --stdio
```

#### Option B: Project Scope (Team Sharing)

Add to a specific project for team use (creates `.mcp.json`):

```bash
claude mcp add hermes --scope project --transport stdio \
  -e HERMES_PROJECT_ROOT=. \
  -- /path/to/hermes/target/release/Hermes --stdio
```

Or create `.mcp.json` manually in your project root:

```json
{
  "mcpServers": {
    "hermes": {
      "command": "/path/to/hermes/target/release/Hermes",
      "args": ["--stdio"],
      "env": {
        "HERMES_PROJECT_ROOT": "."
      }
    }
  }
}
```

#### Option C: Local Scope (Single Project, Personal)

```bash
claude mcp add hermes --transport stdio \
  -e HERMES_PROJECT_ROOT=. \
  -- /path/to/hermes/target/release/Hermes --stdio
```

### 3. Verify Configuration

```bash
# List all configured MCP servers
claude mcp list

# Check specific server details
claude mcp get hermes
```

Expected output:
```
hermes: /path/to/hermes/target/release/Hermes --stdio - ✓ Connected
```

### 4. Database Location

Hermes stores its knowledge graph in SQLite:
- Default location: `<project_root>/.hermes.db`
- Can be overridden via `HERMES_DB_PATH` environment variable

## How to Use

### Initial Setup (Per Project)

1. **Start Claude Code** in your project directory
2. **Index the project** by asking Claude to use `hermes_index`:
   ```
   "Index this project with hermes"
   ```
3. Hermes will crawl supported files and build the knowledge graph

### Searching Code

Ask Claude to search using hermes:
```
"Search hermes for authentication logic"
```

Claude will use `hermes_search` which returns pointers like:
```json
{
  "pointers": [
    {
      "node_id": "abc123",
      "file_path": "src/auth.rs",
      "start_line": 45,
      "end_line": 82,
      "summary": "fn authenticate_user(...) -> Result<User>",
      "relevance": 0.95
    }
  ]
}
```

### Fetching Full Content

When more detail is needed, Claude uses `hermes_fetch` with a node ID:
```
"Fetch the full content of that authentication function"
```

### Recording Facts

Store persistent knowledge about the codebase:
```
"Record a hermes fact: decision - We use JWT tokens for API authentication"
```

Fact types: `architecture`, `decision`, `learning`, `constraint`, `error_pattern`, `api_contract`

### Viewing Statistics

Check token savings:
```
"Show hermes stats"
```

## Available MCP Tools

| Tool | Description | Parameters |
|------|-------------|------------|
| `hermes_search` | Search knowledge graph, returns pointers | `query` (string) |
| `hermes_fetch` | Fetch full content by node ID | `node_id` (string) |
| `hermes_index` | Re-index project files | none |
| `hermes_stats` | Show token savings statistics | none |
| `hermes_fact` | Record a persistent fact | `fact_type`, `content` |
| `hermes_facts` | List active facts | `fact_type` (optional) |

## Supported File Types

Hermes indexes these file extensions:
- **Code**: `.rs`, `.ts`, `.tsx`, `.js`, `.jsx`, `.py`, `.sh`, `.ps1`, `.tf`
- **Config**: `.toml`, `.json`, `.yml`, `.yaml`
- **Docs**: `.md`, `.css`

Ignored directories: `target/`, `node_modules/`, `.git/`, `.venv/`, `dist/`, `.next/`, `.vite/`

## Environment Variables

| Variable | Default | Description |
|----------|---------|-------------|
| `HERMES_PROJECT_ROOT` | Current directory | Root directory to index |
| `HERMES_DB_PATH` | `<root>/.hermes.db` | SQLite database path |
| `HERMES_AUTO_INDEX_INTERVAL_SECS` | `300` | Auto-reindex interval (0 to disable) |
| `GEMINI_API_KEY` | unset | Optional: enables vector search |

## Search Tiers

Hermes uses a three-tier hybrid search:

1. **L0 - Literal**: Fast regex pattern matching (exact matches)
2. **L1 - FTS**: SQLite full-text search (semantic keywords)
3. **L2 - Vector**: Embedding similarity (requires `GEMINI_API_KEY`)

Search strategy:
- If L0 finds high-confidence matches (score >= 0.9), returns immediately
- Otherwise, runs L1 and L2 in parallel
- Results are deduplicated and ranked

## Troubleshooting

### MCP Server Not Loading

1. Verify the binary exists:
   ```bash
   ls -la /path/to/hermes/target/release/Hermes
   ```

2. Check MCP server status:
   ```bash
   claude mcp list
   ```

3. Test the server manually:
   ```bash
   echo '{"jsonrpc":"2.0","id":1,"method":"initialize","params":{"protocolVersion":"2024-11-05","capabilities":{},"clientInfo":{"name":"test","version":"1.0.0"}}}' | \
   HERMES_PROJECT_ROOT=/path/to/project ./target/release/Hermes --stdio
   ```

4. Remove and re-add the server:
   ```bash
   claude mcp remove hermes
   claude mcp add hermes --scope user --transport stdio \
     -e HERMES_PROJECT_ROOT=. \
     -- /path/to/hermes/target/release/Hermes --stdio
   ```

### No Search Results

1. Ensure the project is indexed: use `hermes_index`
2. Check that files have supported extensions
3. Verify `HERMES_PROJECT_ROOT` points to the correct directory

### Database Issues

Delete and re-index:
```bash
rm /path/to/project/.hermes.db
# Then run hermes_index in Claude Code
```

## Token Accounting

Hermes tracks token savings automatically:
- **Pointer tokens**: Tokens used by search result pointers (~30-50 per result)
- **Fetched tokens**: Tokens for full content when fetched
- **Traditional estimate**: What full-file RAG would have cost

Session ID is the current date (YYYY-MM-DD), so stats accumulate daily and reset at midnight.

## File Locations

```
~/.claude.json               # User/local MCP server configurations
<project>/.mcp.json          # Project-scoped MCP servers (team-shared)
<project>/.hermes.db         # Knowledge graph database (per project)
```

## Managing MCP Servers

```bash
# List all servers
claude mcp list

# Get details for a server
claude mcp get hermes

# Remove a server
claude mcp remove hermes

# Remove from specific scope
claude mcp remove hermes --scope user
```
