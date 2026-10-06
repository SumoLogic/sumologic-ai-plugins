# Alert and Security Analysis

Guide for triaging alerts, investigating SIEM insights, and managing detection rules in Sumo Logic.

---

## Alert Triage Workflow

### Step 1 — Search alerts

Use `alertsSearch` to find relevant alerts:

```
# Find triggered alerts
status:Triggered

# Find critical alerts
severity:Critical

# Find alerts by monitor name
monitorName:api-error-rate

# Combine filters
status:Triggered severity:Critical
```

### Step 2 — Read alert details

Use `alertsReadById` with the alert ID from search results to get full context: monitor query, thresholds, trigger conditions.

### Step 3 — Investigate underlying logs

Use the monitor's query pattern from the alert details as a starting point for log investigation. Follow the 3-step log search workflow from `steering/log-investigation.md` to gather supporting evidence.

---

## Alert Search Filters

| Filter | Example |
|--------|---------|
| `status` | `status:Triggered`, `status:Resolved` |
| `severity` | `severity:Critical`, `severity:Warning`, `severity:MissingData` |
| `monitorName` | `monitorName:api-error-rate` |
| `monitorId` | `monitorId:000000000123ABCD` |
| `isMuted` | `isMuted:true`, `isMuted:false` |
| `subscribedMonitors` | `subscribedMonitors:true` |
| `createdAfter` | `createdAfter:1457997111` (timestamp in ms) |
| `createdBefore` | `createdBefore:1457997222` (timestamp in ms) |
| `type` | `type:folder` |

Filters can be combined: `status:Triggered severity:Critical isMuted:false`

---

## SIEM Investigation Workflow

When investigating security concerns, follow this order:

### Step 1 — Check Insights first

Use `getInsights` with relevant entity/severity filters:

```
# Open insights
status:"new"

# High severity
severity:>7

# By entity IP
entity.ip:"10.0.1.5"

# By time
timestamp:>2026-07-01T00:00:00+00:00

# By MITRE ATT&CK tag
tag:"MITRE_ATT&CK"

# Combine
status:"new"+severity:>7
```

### Step 2 — Review Detection Rules

Use `getRules` to understand what's being monitored. Check if existing rules cover the threat vector.

### Step 3 — Correlate with logs

Use the log search workflow (`steering/log-investigation.md`) to gather supporting evidence from raw logs.

### Step 4 — Take action

- `updateInsightStatus` — update status (new → inprogress → closed)
- `updateInsightAssignee` — reassign to the right team/user
- Create new detection rules if a gap is identified

---

## Insight Query DSL

The `q` parameter uses a custom DSL with `+` to concatenate filters:

### Operators

| Operator | Example |
|----------|---------|
| Equals | `status:"new"` |
| In | `status:in("new", "inprogress")` |
| Contains | `description:contains("brute force")` |
| Greater than | `severity:>7` |
| Range | `severity:5..10` |
| Date after | `timestamp:>2026-07-01T00:00:00+00:00` |
| Date range | `timestamp:2026-07-01T00:00:00+00:00..2026-07-02T00:00:00+00:00` |

### Available fields

`assignee`, `closed`, `confidence`, `created`, `description`, `enrichment`, `entity.hostname`, `entity.id`, `entity.ip`, `entity.sensorZone`, `entity.type`, `entity.username`, `entity.value`, `id`, `insightId`, `involvedEntities.id`, `involvedEntities.type`, `involvedEntities.value`, `name`, `readableId`, `records`, `resolution`, `ruleId`, `sensorZone`, `serialId`, `severity`, `status`, `statusId`, `subResolution`, `tag`, `timestamp`, `triageVerdict`

---

## Creating Detection Rules

When creating rules, always:

1. Set `enabled: false` initially — inform the user to review before enabling
2. Provide a clear `descriptionExpression` explaining what the rule detects
3. Use appropriate entity selectors (`_ip`, `_hostname`, `_username`)
4. Set a reasonable `score` (1-10) based on severity
5. Choose the right rule type:
   - **Match Rule** (`createTemplatedMatchRule`) — fires when a single record matches an expression
   - **Threshold Rule** (`createThresholdRule`) — fires when count of matching records exceeds a limit within a time window
