---
name: "sumo-logic"
displayName: "Sumo Logic Observability"
description: "Query logs, alerts, dashboards, SIEM insights, and detection rules from Sumo Logic for production debugging, incident response, and security investigation"
keywords:
  - "sumo logic"
  - "observability"
  - "logs"
  - "log search"
  - "monitoring"
  - "alerts"
  - "siem"
  - "insights"
  - "incidents"
  - "dashboards"
  - "detection rules"
  - "cloud operations"
  - "root cause analysis"
  - "security"
  - "devops"
  - "error"
  - "latency"
  - "troubleshoot"
  - "investigate"
author: "Sumo Logic"
---

# Sumo Logic Observability — Kiro Power

You are enhanced with the **Sumo Logic MCP Server**, providing direct access to the Sumo Logic observability and security platform. Query production logs, analyze alerts, investigate security insights, and manage dashboards without leaving your IDE.

---

## Onboarding

### Prerequisites

- A Sumo Logic account (any role — no admin required)
- Your deployment's MCP host (see table below)

### Step 1 — Pick the deployment (agent: do this first)

If the `sumo-logic` MCP server is not connected and `SUMO_LOGIC_MCP_HOST` is not set, ask the user which Sumo Logic deployment they use. Resolve their answer (region name, deployment code, or a Sumo Logic URL) to an MCP host from this table. Ask for clarification if ambiguous.

| Region | Deployment | `SUMO_LOGIC_MCP_HOST` |
|--------|------------|-----------------------|
| US East (N. Virginia) | us1 | `mcp.sumologic.com` |
| US West (Oregon) | us2 | `mcp.us2.sumologic.com` |
| Europe (Ireland) | dub | `mcp.eu.sumologic.com` |
| Europe (Frankfurt) | fra | `mcp.de.sumologic.com` |
| Asia Pacific (Tokyo) | tky | `mcp.jp.sumologic.com` |
| Asia Pacific (Sydney) | syd | `mcp.au.sumologic.com` |
| Asia Pacific (Seoul) | kor | `mcp.kr.sumologic.com` |
| Canada (Central) | mon | `mcp.ca.sumologic.com` |
| FedRAMP (US East) | fed | `mcp.fed.sumologic.com` |

### Step 2 — Apply the host (agent: guide the user, do not edit the file yourself)

Kiro does not let the agent edit its MCP settings, so the user makes one small edit. The power registers the server in `~/.kiro/settings/mcp.json` (Windows: `%USERPROFILE%\.kiro\settings\mcp.json`) as `power-kiro-power-sumo-logic`, with `${SUMO_LOGIC_MCP_HOST}` in two places. Open that file for the user if you can, and tell them to replace **both** occurrences of `${SUMO_LOGIC_MCP_HOST}` with the host from Step 1 and save. For example, for Europe (Ireland):

```json
"url": "https://mcp.eu.sumologic.com/mcp",
"oauth": {
  "clientMetadataUrl": "https://mcp.eu.sumologic.com/oauth/kiro-client-metadata",
  "redirectUri": "http://localhost:8890/oauth/callback"
}
```

Leave `redirectUri` unchanged. Kiro reconnects automatically when the file is saved; no restart is needed.

**Changing deployment later:** Kiro keeps the stored login under the same server name regardless of host, so a login from one deployment can be replayed to another and fail with `invalid_grant`. Before switching, clear the stored authentication for the `sumo-logic` server in Kiro's MCP panel (or remove and re-import the power), then change the host.

**Alternative (environment variable):** if the user prefers not to edit the file, have them set `SUMO_LOGIC_MCP_HOST=<host>` (`export` in a shell and launch Kiro from it, for example `open -a Kiro` on macOS; `setx` on Windows), restart Kiro, and approve the variable when prompted (also under "Mcp Approved Env Vars" in settings).

Then stop. Do not attempt any MCP calls until the server is connected.

### Step 3 — Authenticate

The power authenticates with OAuth 2.0 (authorization code + PKCE) using a Sumo Logic-hosted Client ID Metadata Document (CIMD). No client ID, secret, or admin setup is needed. Kiro does not yet host its own CIMD document ([kirodotdev/Kiro#11076](https://github.com/kirodotdev/Kiro/issues/11076)), so `mcp.json` points `oauth.clientMetadataUrl` at `https://<host>/oauth/kiro-client-metadata`.

Connect the `sumo-logic` server from Kiro's MCP Servers panel. A browser opens to log in with your Sumo Logic credentials. If your org uses an identity provider, click "Sign in with your identity provider".

`oauth.redirectUri` is pinned to `http://localhost:8890/oauth/callback`. It must stay on `localhost` (not `127.0.0.1`).

---------|-------|
| `url` | `https://<mcp-host>/mcp` |
| `oauth.clientMetadataUrl` | `https://<mcp-host>/oauth/kiro-client-metadata` |
| `oauth.redirectUri` | `http://localhost:8890/oauth/callback` (keep as-is; must use `localhost`, not `127.0.0.1`) |

On first use, Kiro opens a browser to log in with your Sumo Logic credentials. If your org uses an identity provider, click "Sign in with your identity provider".

---

## MCP Server

| Server | Transport | Role |
|--------|-----------|------|
| `sumo-logic` | Streamable HTTP | Primary — all log search, alerts, SIEM, dashboards |

---

## Tools

### Log Search (primary investigation tool)

| Tool | Purpose |
|------|---------|
| `runLogSearch` | Execute Sumo Logic search queries over logs. Returns aggregated records or raw messages. Max runtime: 2 min. Max result size: 256MB. |

### Metadata Discovery

| Tool | Purpose |
|------|---------|
| `listPartitions` | Find `_sourceCategory` values from partition routing expressions |
| `listCustomFields` | List extracted fields available for filtering |
| `listExtractionRules` | List field extraction patterns and parsing rules |

### Alerts

| Tool | Purpose |
|------|---------|
| `alertsSearch` | Search alerts by name, status, severity, monitor |
| `alertsReadById` | Get full details of a specific alert |

### SIEM Insights

| Tool | Purpose |
|------|---------|
| `getInsights` / `getAllInsights` | Search and filter security insights |
| `getInsight` | Get full details of a specific insight |
| `updateInsightAssignee` | Reassign an insight |
| `updateInsightStatus` | Update insight status (new/inprogress/closed) |

### Detection Rules

| Tool | Purpose |
|------|---------|
| `getRules` | Search and filter detection rules |
| `getRule` | Get details of a specific rule |
| `createTemplatedMatchRule` | Create a Match Rule |
| `createThresholdRule` | Create a Threshold Rule |

### Dashboards

| Tool | Purpose |
|------|---------|
| `listDashboards` | List available dashboards |
| `getDashboard` | Get dashboard details and panel definitions |
| `createDashboard` | Create a new dashboard |
| `updateDashboard` | Update an existing dashboard |

---

## Intent Detection — Auto-Route Without Asking

When the user describes a problem, automatically choose the right workflow:

### Log Investigation (most common)
**Triggers**: error, logs, latency, timeout, 5xx, 4xx, 500, 503, slow, failing, crash, OOM, debug, troubleshoot, investigate, search logs, log search, what happened

**Action**: Follow the 3-step log search workflow in `steering/log-investigation.md`.

### Alert Triage
**Triggers**: alert, triggered, monitor, alarm, firing, notification, critical alert, warning

**Action**: Follow `steering/alert-analysis.md`.

### Security Investigation
**Triggers**: security, insight, signal, threat, MITRE, entity, suspicious, breach, compromise, detection rule

**Action**: Follow `steering/alert-analysis.md` (SIEM section).

### Dashboard Operations
**Triggers**: dashboard, visualization, panel, chart, create dashboard, show dashboard

**Action**: Follow `steering/dashboard-management.md`.

---

## Available Steering Files

| File | When to Use |
|------|-------------|
| `steering/log-investigation.md` | Log search — mandatory 3-step workflow, query syntax, investigation strategies |
| `steering/alert-analysis.md` | Alerts and SIEM insights — triage, security investigation, detection rules |
| `steering/dashboard-management.md` | Dashboard CRUD operations |

---

## Quick Start

1. Call `listPartitions` to confirm connectivity and see available data sources.
2. Run a scoped sample: `runLogSearch` with `_sourceCategory=<value>` over a 5-minute window, limit 5.
3. If both succeed, the power is working.

---

## Core Principles

1. **Never skip discovery** — always call `listPartitions` + `listCustomFields` + `listExtractionRules` before your first log search.
2. **Start narrow** — 5-minute time ranges for samples, then expand for targeted queries.
3. **Scope every query** — always include `_sourceCategory` (and `_collector` when known). Unscoped queries are rejected for time ranges over 30 minutes.
4. **Bound results** — raw queries need `| limit N` on the first line; aggregate queries need `| sort _count desc | limit N` at the end.
5. **Use `nodrop`** — when parsing fields that may be absent (`| json "field" as f nodrop`), always add `nodrop` to avoid silently dropping rows.

---

## Support & Legal

- **Documentation**: [Sumo Logic MCP Server Docs](https://www.sumologic.com/help/docs/api/mcp-server/)
- **Support**: [Sumo Logic Support](https://support.sumologic.com/)
- **License**: Apache-2.0
- **Privacy**: [Sumo Logic Privacy Statement](https://www.sumologic.com/privacy-statement/)
