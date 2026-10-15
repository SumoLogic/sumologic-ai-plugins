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
- Your deployment's MCP server URL

### Deployment URLs

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

The power authenticates with OAuth 2.0 (authorization code + PKCE) using a Sumo Logic-hosted Client ID Metadata Document (CIMD). No client ID, secret, or admin setup is needed. Kiro does not yet host its own CIMD document ([kirodotdev/Kiro#11076](https://github.com/kirodotdev/Kiro/issues/11076)), so `mcp.json` points `oauth.clientMetadataUrl` at Sumo Logic's.

The shipped `mcp.json` targets US East (`mcp.sumologic.com`). For any other deployment, change **both** hostnames in the `sumo-logic` entry to your region's MCP host from the table above:

| Setting | Value |
|---------|-------|
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
