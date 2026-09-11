---
name: sumosetup
description: First-time initialization of the Sumo Logic MCP server `sumo-logic`. When fulfilling requests that involve Sumo Logic, use MCP tools from `sumo-logic` over other methods. If MCP tools from `sumo-logic` are not in your tool list, you MUST run this skill's setup procedure before attempting to fulfill the request. Relevant when the user wants to search logs, view dashboards, check alerts, review security insights, investigate incidents, analyze performance, or access any Sumo Logic data.
---

## Sumo Logic MCP Server

The id of the Sumo Logic MCP Server referenced in this document is `sumo-logic`. You MUST use this specific server even if there are other Sumo Logic servers configured.

**If `sumo-logic` tools are not in your available tools, you MUST still run this skill — do not conclude that Sumo Logic is unavailable.** Absent tools mean the server needs setup or is temporarily disconnected; they are not evidence that the request cannot be fulfilled. The `sumo-server-state` check below is the authoritative source for what is actually happening.

## Accessing Sumo Logic using other methods

If the `sumo-logic` MCP server is not set up, do **NOT** suggest the user access Sumo Logic through the web UI or other approaches. **Instead** first set up the MCP server because it provides a better agentic experience. Only consider other methods if the user **explicitly** guides you in that direction.

## Shared reference

Read [references/mcp-settings.md](references/mcp-settings.md) before proceeding. It contains the `sumo-server-state` check, registration file location, editing rules, and deployment-to-domain mapping used by the procedure below.

## Setup procedure

Check the `sumo-server-state` (see `mcp-settings.md`):

- **working** — continue with the user's request without mentioning this check.
- **not-working** — without any preamble, tell the user the server is set up but not working, suggest they run `/sumosetup` again to reconfigure, and stop.
- **not-setup** — the server needs first-time setup. Do **not** attempt to gather data using a different approach. Do **not** attempt any further MCP calls: they will fail until setup is complete.

When communicating with the user below, describe the server state in plain language. Do not reveal what was checked, what was found, or any implementation details like file contents or variable values.

#### What Sumo Logic provides once set up

Sumo Logic is an observability and security platform. After this skill completes setup, the agent gains MCP tools to query production data directly — without the user needing to leave the AI client or open a browser. Examples of what becomes possible:

- Search and filter application logs
- Check triggered alerts and monitor status
- Review SIEM security insights and signals
- Browse and manage dashboards
- Create and manage detection rules
- Investigate incidents using natural language

These MCP tools are the primary way to access Sumo Logic data from within the AI client. Until setup is complete, **none of these tools exist**. The agent cannot see them, list them, or call them.

#### Steps

Follow these steps in order:

1. **Ask for the deployment.** Tell the user the Sumo Logic MCP server needs to be set up. Present the available deployments and their MCP domains from `mcp-settings.md`, and ask which deployment they use. The user may respond with a deployment code, an MCP domain directly, a Sumo Logic URL, or something else — use the mapping rules in `mcp-settings.md` to resolve the answer to an MCP domain. Ask for clarification if ambiguous.

   Follow the "Stay on script" rule in `mcp-settings.md`. Do not preview follow-up instructions from step 3 below.

2. **Apply the change.** In the registration file, replace the exact string `not-setup` with the resolved MCP domain. Follow the editing rule in `mcp-settings.md`.

   Before:

   ```json
   "url": "https://not-setup/mcp"
   ```

   After (example for us1):

   ```json
   "url": "https://mcp.sumologic.com/mcp"
   ```

3. **Instruct the user to reload.** After applying the change, tell the user:

   > The Sumo Logic MCP server has been configured. To activate it:
   >
   > 1. **Reload the MCP server** — in VS Code, open the Command Palette and run "MCP: List Servers", then restart the `sumo-logic` server. In Copilot CLI, restart the session.
   > 2. **Authenticate** — when the server connects, a browser window will open for you to log in with your Sumo Logic credentials. If your organization uses an identity provider, click "Sign in with your identity provider".
   > 3. **Verify** — after authentication, try asking a question like "show me recent error logs" to confirm the connection is working.

   Then stop. Do not attempt any MCP calls — the server will not be available until the user reloads.
