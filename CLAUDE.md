# Ollie MCP Server

WordPress-native MCP (Model Context Protocol) server for AI-assisted page building with the Ollie theme.

## Project Vision

Use natural language via Claude to build WordPress pages using Ollie's block patterns and design system. The primary interface is Claude CLI (or any MCP-compatible client) talking directly to WordPress via the WordPress Abilities API and MCP Adapter.

### Key Design Decisions

- **WordPress Abilities API**: Ollie-specific abilities registered via WordPress 6.9's Abilities API
- **MCP Adapter Integration**: Uses the official WordPress MCP Adapter to expose abilities as MCP tools
- **Ollie-focused**: Only provides Ollie-specific abilities (patterns, design tokens); generic WordPress CRUD uses core abilities
- **Draft-first editing**: Changes are made to drafts visible in WP Admin, not live published content

## Architecture

```
server/                          # Bedrock WordPress installation
├── composer.json                # Includes wordpress/abilities-api and wordpress/mcp-adapter
├── web/app/plugins/
│   └── ollie-mcp/              # Ollie abilities plugin (git submodule)
│       ├── ollie-mcp.php       # Plugin entry point with dependency check
│       ├── src/
│       │   ├── Plugin.php      # Registers ability category and abilities
│       │   ├── Discovery.php   # Catalogs patterns, blocks, design tokens
│       │   └── Abilities/      # Ability implementations
│       │       ├── GetPatterns.php
│       │       └── GetDesignTokens.php
│       └── views/admin.php     # Admin status page
└── web/app/themes/ollie/       # Ollie theme (via Composer)
```

## Dependencies

- **WordPress 6.9+** - Includes Abilities API in core
- **wordpress/mcp-adapter** - MCP protocol bridge (exposes abilities as MCP tools)

## Local Development

### Prerequisites

- PHP 8.1+
- MySQL database: `olliewpmcp` (user: `root`, password: `password`)
- Herd (or similar local PHP server)
- WP-CLI

### URLs

- **Site**: https://olliewpmcp.test
- **Admin**: https://olliewpmcp.test/wp/wp-admin/
- **Credentials**: `admin` / `password`

## Ollie Abilities

| Ability | Description |
|---------|-------------|
| `ollie-mcp/get-patterns` | List Ollie block patterns, optionally with full markup |
| `ollie-mcp/get-design-tokens` | Get theme colors, typography, spacing, and shadows |

Generic WordPress abilities (post CRUD, etc.) are provided by WordPress core and other plugins.

## Testing MCP

```bash
# List all registered abilities
wp mcp-adapter list

# Test abilities via STDIO transport
echo '{"jsonrpc":"2.0","id":1,"method":"tools/list","params":{}}' | \
  wp mcp-adapter serve --user=admin --server=mcp-adapter-default-server
```

## MCP Transport Options

### STDIO Transport (Local)

For local development where WP-CLI is available:

```json
{
  "mcpServers": {
    "wordpress": {
      "command": "wp",
      "args": [
        "--path=/path/to/server/web/wp",
        "mcp-adapter",
        "serve",
        "--server=mcp-adapter-default-server",
        "--user=admin"
      ]
    }
  }
}
```

### HTTP Transport (Remote)

For remote access via REST API:

- **Endpoint:** `POST /wp-json/mcp/mcp-adapter-default-server`
- **Auth:** Application Password via Basic Auth
- **Session:** Include `Mcp-Session-Id` header from initialize response

## Claude CLI Setup

### Option 1: Local with STDIO

Best for local development. Claude CLI spawns WP-CLI as a subprocess.

1. Ensure WP-CLI is installed and in your PATH
2. Add to `~/.claude.json`:

```json
{
  "mcpServers": {
    "olliewpmcp": {
      "command": "wp",
      "args": [
        "--path=/Users/aaron/repos/OllieMCP/server/web/wp",
        "mcp-adapter",
        "serve",
        "--server=mcp-adapter-default-server",
        "--user=admin"
      ]
    }
  }
}
```

3. Restart Claude CLI - the `olliewpmcp` server will be available

### Option 2: Remote with HTTP

Best for connecting to remote WordPress sites.

1. Create an Application Password at `https://yoursite.com/wp-admin/profile.php`

2. Generate Base64 credentials:
```bash
echo -n "admin:YOUR_APP_PASSWORD" | base64
```

3. Add to `~/.claude.json`:

```json
{
  "mcpServers": {
    "olliewpmcp": {
      "url": "https://olliewpmcp.test/wp-json/mcp/mcp-adapter-default-server",
      "headers": {
        "Authorization": "Basic YOUR_BASE64_CREDENTIALS"
      }
    }
  }
}
```

4. Restart Claude CLI

### Verifying Connection

Once configured, ask Claude CLI:

```
What MCP tools are available from WordPress?
```

Claude should list the available tools including `mcp-adapter-discover-abilities`, `mcp-adapter-execute-ability`, etc.

## Submodule Workflow

The `ollie-mcp` plugin is a git submodule. To work on it:

```bash
cd web/app/plugins/ollie-mcp
git checkout main
# make changes, commit, push
```

After cloning the server repo fresh:

```bash
git submodule update --init --recursive
```

## Future Plans

- **Enhanced abilities**: Style variation switching, block-level manipulation
- **Embedded chat UI**: Integrate BotManChatSDK for in-admin chat interface
- **Resource exposure**: Expose patterns as MCP resources for better context
