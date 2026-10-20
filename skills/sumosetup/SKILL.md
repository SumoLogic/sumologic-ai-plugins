---
name: sumosetup
description: First-time initialization of the Sumo Logic MCP server `sumo-logic`. When fulfilling requests that involve Sumo Logic, use MCP tools from `sumo-logic` over other methods. If MCP tools from `sumo-logic` are not in your tool list, you MUST run this skill's setup procedure before attempting to fulfill the request. Relevant when the user wants to search logs, view dashboards, check alerts, review security insights, investigate incidents, analyze performance, or access any Sumo Logic data.
---

## Sumo Logic MCP Server

The id of the Sumo Logic MCP Server referenced in this document is `sumo-logic`. You MUST use this specific server even if there are other Sumo Logic servers configured.

**In Kiro IDE specifically:** after setup, the panel shows two entries — `sumo-logic` (user-settings, works) and an entry whose name starts with `power-` and ends with `-sumo-logic` (power-managed, always fails). Always use `sumo-logic` for MCP calls. The power-managed entry is non-functional by design due to a known Kiro limitation.

**If `sumo-logic` tools are not in your available tools, you MUST still run this skill — do not conclude that Sumo Logic is unavailable.** Absent tools mean the server needs setup or is temporarily disconnected; they are not evidence that the request cannot be fulfilled. The `sumo-server-state` check below is the authoritative source for what is actually happening.

## Accessing Sumo Logic using other methods

If the `sumo-logic` MCP server is not set up, do **NOT** suggest the user access Sumo Logic through the web UI or other approaches. **Instead** first set up the MCP server because it provides a better agentic experience. Only consider other methods if the user **explicitly** guides you in that direction.

## Shared reference

Read [references/mcp-settings.md](references/mcp-settings.md) before proceeding. It contains the `sumo-server-state` check, registration file location, editing rules, and deployment-to-domain mapping used by the procedure below.

## Setup procedure

Check the `sumo-server-state` (see `mcp-settings.md`):

- **working** — continue with the user's request without mentioning this check.
- **not-working** — follow the **Troubleshooting** section below.
- **not-setup** — follow the **First-time setup** section below.

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

---

### First-time setup

#### Kiro IDE

**Background — why this setup is needed:**
The agent-plugins spec does not support OAuth configuration in a power's `mcp.json`. Because of this, Kiro cannot authenticate the power-managed Sumo Logic server on its own. Additionally, Kiro's built-in OAuth client currently only supports Dynamic Client Registration (DCR), but the Sumo Logic auth server requires Client ID Metadata Document (CIMD) — so even if Kiro tries to connect via the power entry, it fails with:

```
Failed to connect: Incompatible auth server: does not support dynamic client registration
```

The working solution is to add a **separate** `sumo-logic` entry to `~/.kiro/settings/mcp.json` with the full OAuth config including `clientMetadataUrl`. This entry connects successfully using CIMD.

**Known limitation:** Due to a Kiro bug ([kirodotdev/Kiro#11964](https://github.com/kirodotdev/Kiro/issues/11964)), this results in **two entries** in the Kiro MCP Servers panel:
- An entry whose name starts with `power-` and ends with `-sumo-logic` — the power-managed entry, shows as erroring/failed. **Ignore this entry.** (The exact name depends on how the power was installed, e.g. `power-sumologic-ai-plugins-sumo-logic`.)
- `sumo-logic` — the user-settings entry, connects and works correctly. **This is the one to use.**

Kiro does not merge the two entries even if given the same name — it silently ignores user settings OAuth config when names match. This is a known bug being tracked upstream.

1. **Ask for the deployment.** Tell the user the Sumo Logic MCP server needs to be set up. Present the available deployments and their MCP domains from `mcp-settings.md`, and ask which deployment they use. Also ask whether their organization uses a Sumo Logic subdomain; if so, prefix `<subdomain>.` to the resolved domain (for example `acme.mcp.us2.sumologic.com`) in **both** the `url` and the `clientMetadataUrl` below. The MCP server is not supported in the Zurich or AWS European Sovereign Cloud deployments.

2. **Apply the change.** Write the following entry into `~/.kiro/settings/mcp.json` under the top-level `mcpServers` key (not under `powers.mcpServers`), replacing `<mcp-domain>` with the resolved domain:

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

   Note: Kiro protects `~/.kiro/settings/mcp.json` from agent edits. If the agent cannot write the file, show the user **only** the `sumo-logic` entry to merge into their existing file — do not show the full file or any other keys. Tell them to add it under the top-level `mcpServers` key and save:

   ```json
   "sumo-logic": {
     "url": "https://<mcp-domain>/mcp",
     "oauth": {
       "clientMetadataUrl": "https://<mcp-domain>/oauth/kiro-client-metadata",
       "redirectUri": "http://localhost:8890/oauth/callback"
     }
   }
   ```

3. **Instruct the user to reload.** After applying the change, tell the user:

   > The Sumo Logic MCP server has been configured. To activate it:
   >
   > 1. **Save the file** — Kiro detects the change automatically. The `sumo-logic` entry should appear in the MCP Servers section of the Kiro panel within a few seconds.
   > 2. **Authenticate** — a browser window should open automatically for you to log in with your Sumo Logic credentials. If it does not open automatically, click the **`sumo-logic`** entry in the Kiro MCP Servers panel and select **Authenticate**. If your organization uses an identity provider, click "Sign in with your identity provider".
   > 3. **Ignore the second entry** — you will also see an entry in the panel whose name starts with `power-` and ends with `-sumo-logic` (e.g. `power-sumologic-ai-plugins-sumo-logic`), showing as failed or erroring. This is a known Kiro limitation — that entry cannot authenticate and can be safely ignored or disabled. The `sumo-logic` entry is the one that works.
   > 4. **Verify** — after authentication, try asking a question like "show me recent error logs" to confirm the connection is working.

   If the user-level file cannot be written, you may instead write the same entry to `<workspace>/.kiro/settings/mcp.json` (create it if missing; it applies to that workspace only). Tell the user it is workspace-scoped and must not be committed to source control.

   Then stop. Do not attempt any MCP calls — the server will not be available until the user authenticates.

#### Copilot (VS Code or CLI)

If the registration file (`mcp.json`) contains the `not-setup` sentinel, follow these steps:

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

#### Claude Code

If there is no registration file or no `not-setup` sentinel (the server is configured but not connected), guide the user through Claude Code's native setup:

1. Tell the user the Sumo Logic MCP server needs to be connected, and instruct them:

   > To connect the Sumo Logic MCP server:
   >
   > 1. Run `/mcp` in Claude Code
   > 2. Select **sumo-logic** from the server list
   > 3. Select **Authenticate** — a browser window will open for you to log in with your Sumo Logic credentials
   > 4. After authentication, run `/mcp` again to confirm the server shows as **connected**
   >
   > If `sumo-logic` does not appear in the server list, install the plugin first. In Claude Code, run `/plugins` and search for `sumologic-mcp`. In Claude Cowork, go to **Customize** → **Plugins** → **Discover** and search for `sumologic-mcp`.

   Then stop. Do not attempt any MCP calls until the user confirms the server is connected.

---

### Troubleshooting

If the server is configured but not working (`sumo-server-state` is **not-working**):

1. Tell the user the Sumo Logic server is configured but not currently connected.
2. Suggest the following steps based on client:

   **VS Code / Copilot CLI:**
   > 1. **Re-authenticate** — your session may have expired. Use `/mcp` to select the `sumo-logic` server and re-authenticate.
   > 2. **Check the deployment URL** — make sure the configured URL matches your Sumo Logic deployment region.
   > 3. **Run `/sumosetup` again** — this will reconfigure the server from scratch.

   **Claude Code:**
   > 1. **Re-authenticate** — run `/mcp` → select **sumo-logic** → **Clear authentication** → **Authenticate**.
   > 2. **Check the deployment URL** — make sure the configured URL matches your Sumo Logic deployment region.
   > 3. **Run `/sumosetup` again** — this will reconfigure the server from scratch.

   **Kiro IDE:**
   > 1. **Clear the stored session** — in the Kiro MCP Servers panel, click the **`sumo-logic`** entry and select **Disconnect** or **Clear authentication** (the exact label depends on your Kiro version). Then click **Authenticate** to start a fresh login.
   > 2. **If you changed deployment or subdomain** — you must clear the stored session first (step 1) before re-authenticating. Skipping this causes an `invalid_grant` error because Kiro tries to reuse a token issued for the old domain.
   > 3. **Check the deployment URL** — open `~/.kiro/settings/mcp.json` (or `<workspace>/.kiro/settings/mcp.json` if you used the workspace-scoped fallback during setup) and confirm the `url` and `clientMetadataUrl` both use the correct MCP domain for your Sumo Logic region.
   > 4. **Reconfigure from scratch** — ask the agent to set up Sumo Logic again and it will walk you through the full setup flow.

   Then stop.

#### Kiro-specific: "I see a failed/erroring entry in the MCP panel"

In Kiro, it is expected and normal to see **two** entries in the MCP Servers panel after setup:

- An entry whose name starts with `power-` and ends with `-sumo-logic` — this is the power-managed entry. It will always show as failed or erroring because Kiro's OAuth client does not support the CIMD authentication mechanism that Sumo Logic requires (tracked in [kirodotdev/Kiro#11076](https://github.com/kirodotdev/Kiro/issues/11076) and [kirodotdev/Kiro#11964](https://github.com/kirodotdev/Kiro/issues/11964)). **This entry can be ignored or disabled.**
- `sumo-logic` — this is the user-settings entry with the full OAuth config. This is the entry that connects and works. **Use this one.**

If only the `power-` entry appears and `sumo-logic` is missing, the user-level settings entry was not added correctly — re-run setup.

If `sumo-logic` appears but fails to connect (not just the power entry), follow the Kiro troubleshooting steps above.
