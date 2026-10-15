# Dashboard Management

Guide for listing, viewing, creating, and updating Sumo Logic dashboards.

---

## Viewing Dashboards

### List dashboards

Use `listDashboards` to see available dashboards. Supports pagination via `token`.

### Get dashboard details

Use `getDashboard` with the dashboard UUID. Returns:
- Title, description, theme
- Panel definitions (queries, visual settings)
- Variables and their source definitions
- Layout structure (Grid positions)
- Time range configuration

---

## Creating Dashboards

Use `createDashboard` with these required fields:

| Field | Required | Description |
|-------|----------|-------------|
| `title` | Yes | Dashboard title |
| `timeRange` | Yes | Default time range for all panels |
| `panels` | No | Array of panel definitions |
| `layout` | No | Grid layout positions for panels |
| `variables` | No | Template variables for dynamic filtering |
| `folderId` | No | Folder to save in (default: personal folder) |
| `description` | No | Dashboard description |

### Panel definition

Each panel needs:
- `key` — unique identifier (e.g., `panelPANE-ABC123`)
- `panelType` — typically `SumoSearchPanel`
- `title` — panel title
- `visualSettings` — JSON string defining chart type and appearance

### Layout structure

```json
{
  "layoutType": "Grid",
  "layoutStructures": [
    {
      "key": "panelPANE-ABC123",
      "structure": "{\"height\":5,\"width\":12,\"x\":0,\"y\":0}"
    }
  ]
}
```

Grid is 24 columns wide. Panels are positioned by `x`, `y`, `width`, `height`.

### Time range

```json
{
  "type": "BeginBoundedTimeRange",
  "from": {
    "type": "RelativeTimeRangeBoundary",
    "relativeTime": "-15m"
  }
}
```

### Variables

Variables allow dynamic filtering. Each variable needs:
- `name` — variable name (referenced as `{{name}}` in queries)
- `sourceDefinition` — where values come from (`MetadataVariableSourceDefinition`, `LogQueryVariableSourceDefinition`, etc.)

---

## Updating Dashboards

Use `updateDashboard` with the dashboard `id` plus the fields to change. Required fields: `id`, `title`, `timeRange`.

Fetch the existing dashboard with `getDashboard` first, then modify and send back the full definition.

---

## Best Practices

1. **Use variables** for `_sourceCategory`, environment, and service name to make dashboards reusable
2. **Set reasonable refresh intervals** — 0 (manual), 60 (1 min), 300 (5 min) are common choices
3. **Organize with folders** — use `folderId` to keep dashboards organized
4. **Include descriptions** — help others understand what the dashboard monitors
5. **Bound panel queries** — every panel query should have `| limit` or aggregation bounds to prevent slow loading
