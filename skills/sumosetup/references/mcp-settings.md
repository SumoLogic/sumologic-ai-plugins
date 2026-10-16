# MCP JSON Registration Reference

The MCP registration file is shared across all plugin skills. Use the flows below to check the server state, locate the registration file, edit a value, or map a Sumo Logic deployment to its MCP domain.

### Stay on script

Describe state and actions in plain language ("the Sumo Logic MCP server is not set up", "the deployment has been updated"). Never reveal:

- File paths, file names, or directory layout.
- The sentinel value `not-setup` or related terms such as "domain placeholder".
- Variable names, values, environment variables, or shell syntax.
- API keys, tokens, client secrets, or credentials of any kind.

Emit only what the current step instructs. Do not add setup tips, follow-ups, or "helpful" notes from general knowledge of the AI client. When the user needs to reload or re-authenticate, the skill emits that instruction at the correct step.

**Exception — Kiro IDE:** The Kiro first-time setup branch requires showing the user specific file paths (`~/.kiro/settings/mcp.json` and the workspace-scoped fallback) so they can edit the file manually, since Kiro protects it from agent writes. Showing these paths in the Kiro branch is intentional and does not violate this rule.

## Determine `sumo-server-state`

Silently determine the state of the `sumo-logic` MCP server using **only** the steps below. Do not use any other source of information (status files, cached state, error messages from previous calls):

1. Try a lightweight MCP call on `sumo-logic` (e.g. list tools or read a resource).
2. If the server returns actual Sumo Logic data (tools, resources, or content) -> state is **working**.
3. If the call fails or returns empty/generic content:
   - **In Kiro IDE**: silently read `~/.kiro/settings/mcp.json`. If `sumo-logic` is present under the top-level `mcpServers` key -> state is **not-working**. If absent, also check `<workspace>/.kiro/settings/mcp.json` (the workspace-scoped fallback). If `sumo-logic` is present there -> state is **not-working**. If absent from both files -> state is **not-setup**.
   - **In other clients**: silently read the registration file (see below). Check for the literal string `not-setup`: if found -> state is **not-setup**. Otherwise -> state is **not-working**.

Do not tell the user which state was determined, what was checked, or what was found. Follow the skill's instructions for that state.

## Kiro IDE: registration file

In Kiro, the active MCP configuration lives in `~/.kiro/settings/mcp.json` under the top-level `mcpServers` key. Kiro also reads the power's own `mcp.json` and **registers and attempts to connect** the server defined there — this is why a second, power-managed entry (named `power-<power-name>-sumo-logic`) appears in the MCP Servers panel alongside the user-level `sumo-logic` entry. That power-managed entry will always fail to authenticate due to missing OAuth config and a known Kiro OAuth limitation (CIMD not supported). It can be ignored.

**Important:** Kiro protects `~/.kiro/settings/mcp.json` from agent writes. If the agent cannot edit the file, show the user **only** the `sumo-logic` server entry (not the full file) and ask them to merge it into the `mcpServers` object in their existing file. Never dump the full file contents — the user may have other servers configured that must not be overwritten.

A workspace-scoped fallback is also valid: `<workspace>/.kiro/settings/mcp.json`. If setup used this path instead of the user-level file, the `sumo-server-state` check must look for `sumo-logic` in **both** files — user-level first, then workspace-scoped. If found in either, the server is considered configured.

The correct entry format for Kiro (no `"type"` field):

```json
{
  "mcpServers": {
    "sumo-logic": {
      "url": "https://<mcp-domain>/mcp",
      "oauth": {
        "clientMetadataUrl": "https://<mcp-domain>/oauth/kiro-client-metadata",
        "redirectUri": "http://localhost:8890/oauth/callback"
      }
    }
  }
}
```

## Other clients: registration file `mcp.json`

The registration file is at `<plugin-root>/mcp.json`. If `<plugin-root>` is not already known, derive it from this markdown file's path by removing `skills/sumosetup/references/mcp-settings.md` from the end.

The file contains a `url` field in the `"sumo-logic"` server entry:

```json
"url": "https://<current domain>/mcp"
```

### Editing rule

Edit the `url` string in the `"sumo-logic"` server entry. Leave everything else in the file untouched.

To change the domain: replace the hostname — the part of the URL between `https://` and `/mcp`.

Example:

```json
"url": "https://not-setup/mcp"  ->  "url": "https://mcp.sumologic.com/mcp"
```

### The `not-setup` sentinel

A fresh installation has `not-setup` as the URL hostname:

```json
"url": "https://not-setup/mcp"
```

This prevents the MCP server from connecting. It exists only before first-time setup and is replaced by `/sumosetup` with a real MCP domain. Once replaced, it never returns to `not-setup`.

## Deployment-to-domain mapping

| Deployment | Region | MCP domain |
| --- | --- | --- |
| us1 | US East (N. Virginia) | mcp.sumologic.com |
| us2 | US West (Oregon) | mcp.us2.sumologic.com |
| eu | Europe (Ireland) | mcp.eu.sumologic.com |
| de | Europe (Frankfurt) | mcp.de.sumologic.com |
| jp | Asia Pacific (Tokyo) | mcp.jp.sumologic.com |
| au | Asia Pacific (Sydney) | mcp.au.sumologic.com |
| kr | Asia Pacific (Seoul) | mcp.kr.sumologic.com |
| ca | Canada (Central) | mcp.ca.sumologic.com |
| fed | FedRAMP (US East) | mcp.fed.sumologic.com |

Present all available deployments and their MCP domains, then ask the user which one they use.

When mapping user input:

- **Deployment code** (e.g. "us1", "eu", "de") — use the matching MCP domain directly. Codes are case-insensitive.
- **URL** (e.g. "https://service.sumologic.com", "https://service.us2.sumologic.com") — identify the deployment from the URL subdomain, then use the matching MCP domain. Note: `sumologic.com` with no deployment prefix maps to `us1`.
- **Domain not in the table** — confirm with the user, warning that an invalid domain will prevent connection.

If the user is unsure which deployment they use, suggest checking the URL in their Sumo Logic browser session or asking their Sumo Logic administrator.
