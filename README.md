# Sumo Logic AI Plugins

AI coding tool plugins for [Sumo Logic](https://www.sumologic.com) — connecting Claude Code, Cursor, and other AI development tools to the Sumo Logic observability platform.

## Claude Code Plugin

Connects Claude Code to Sumo Logic via MCP (Model Context Protocol). Investigate logs, analyze incidents, check alerts, review security insights, create detection rules, and manage dashboards — all from your terminal.

### What You Get

- **MCP tools** — `runLogSearch`, `listPartitions`, `listCustomFields`, `listExtractionRules`, dashboards, alerts, SIEM insights, and detection rules
- **`/sumologic-mcp:sumo-investigator`** — Senior investigation agent that answers natural language operational questions using Sumo Logic platform data

### Prerequisites

- [Claude Code](https://code.claude.com/docs/en/quickstart) installed and authenticated
- Sumo Logic account with credentials (any role — no admin required for CIMD)
- Your deployment's MCP server URL (see table below)

### Installation

```
/plugin install sumologic-mcp@claude-community
```

On first enable, Claude Code prompts for your MCP Server URL:

| Region | MCP Server URL |
|--------|----------------|
| US East (N. Virginia) | `https://mcp.sumologic.com/mcp` |
| US West (Oregon) | `https://mcp.us2.sumologic.com/mcp` |
| Europe (Ireland) | `https://mcp.eu.sumologic.com/mcp` |
| Europe (Frankfurt) | `https://mcp.de.sumologic.com/mcp` |
| Asia Pacific (Tokyo) | `https://mcp.jp.sumologic.com/mcp` |
| Asia Pacific (Sydney) | `https://mcp.au.sumologic.com/mcp` |
| Asia Pacific (Seoul) | `https://mcp.kr.sumologic.com/mcp` |
| Canada (Central) | `https://mcp.ca.sumologic.com/mcp` |
| FedRAMP (US East) | `https://mcp.fed.sumologic.com/mcp` |

### Authentication

The plugin uses **OAuth 2.0 with CIMD** (Client ID Metadata Documents) — no pre-created credentials or admin setup required. Claude Code handles authentication and token refresh automatically.

#### First-time setup

1. Launch Claude Code with the plugin installed
2. Run `/mcp`
3. Select **sumo-logic** → **Authenticate**
4. A browser window opens — log in with your Sumo Logic credentials
   - If your org uses an identity provider, click **Sign in with your identity provider**
5. Run `/mcp` again — the server should show as **connected**

#### Switching organizations

To connect to a different Sumo Logic org:
1. Run `/mcp`
2. Select **sumo-logic** → **Clear authentication**
3. Select **Authenticate** and log in to the new org

> **Note:** If you previously granted consent for an org, you will not be prompted again. To revoke consent, go to your Sumo Logic user settings and remove the app under **Personal Authorized Apps**.

#### Manual OAuth (pre-registered client)

For environments that don't support CIMD, an administrator can create an OAuth client manually:

1. Create an Authorization Code client in Sumo Logic with redirect URI `http://localhost:8888/callback`
2. Register manually in Claude Code:

```bash
claude mcp add --scope user --transport http \
  --client-id "<client-id>" --client-secret --callback-port 8888 \
  sumo-logic "https://mcp.sumologic.com/mcp"
```

> **Note:** Recent VS Code releases do not support explicit client credentials. Use the default CIMD setup for VS Code.

### Usage

#### Investigation Skill

```
/sumologic-mcp:sumo-investigator what errors are happening in the API gateway in the last hour?
/sumologic-mcp:sumo-investigator are there any critical security insights from today?
/sumologic-mcp:sumo-investigator show me triggered alerts for the collector service
```

The skill follows a structured 3-step workflow (Discover → Scoped Sample → Targeted Query) and covers logs, alerts, SIEM insights, detection rules, and dashboards.

#### Direct Tool Use

You can also ask naturally without invoking the skill — Claude will use the MCP tools directly:

```
Search sumo for HTTP 500 errors in the last 15 minutes
Show me critical security insights from this week
List dashboards related to kubernetes
```

## Development

Test the plugin locally:

```bash
claude --plugin-dir .
```

Validate before submission:

```bash
claude plugin validate .
```

## License

[Apache-2.0](LICENSE)
