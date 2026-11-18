---
name: sumo-investigator
description: Sumo Logic investigation skill for the Sumo Logic MCP server. Answers questions about the user's systems and security with evidence from their own Sumo Logic org: logs, metrics, alerts, Cloud SIEM insights, detection rules and dashboards. Load it before calling any Sumo Logic tool.
when_to_use: When the user asks about errors, incidents, outages, latency, root cause, suspicious activity, alerts, monitors, metrics or ingestion in systems that send data to Sumo Logic; asks to search logs, query metrics, or create or change a dashboard or detection rule; or mentions Sumo Logic or Sumo.
allowed-tools: runLogSearch getLogSearchEstimatedUsageByMeteringType listPartitions listCustomFields listExtractionRules runMetricsQueries searchDimensions searchValuesWithinDimension alertsSearch alertsReadById getInsights getInsight getAllInsights getRules getRule listDashboards getDashboard
---

# Sumo Logic Investigation Agent
This skill helps answer operational and security questions with evidence from the user's Sumo Logic platform: logs, metrics, alerts, Cloud SIEM insights, detection rules and dashboards. It works with the Sumo Logic MCP server, so the tools below come from the user's own deployment and org, and what they can see depends on the user's permissions.

Use the direct tools below (log search, metrics, alerts, insights, rules, dashboards). They give exact control, and each has a workflow.

Two rules apply throughout:
- **Explore before declining.** If a question might be answerable from platform data, run the discovery workflow first, and only say no once you've looked and confirmed the data isn't there.
- **Back every claim with a tool result.** Don't state anything you can't point to in what the tools returned. No Speculation — if the tools don't have the answer, say so. 

## Connect the logs to the code

Sumo Logic holds what the code does in production, and the repository holds why. Linking the two is what makes an investigation useful: it turns a log line or alert into the code path behind it, and lets each log search and metrics query build on what the code already showed.

The logs you search often come from code in the user's current working directory. When the question is about a service that lives in that code, use it on both ends of the investigation.

- **Before querying:** look in the relevant code for the log messages, logger names, error and exception classes, service names and source categories it emits. Use them to build a precise query.
- **After results come back:** map each finding to where it comes from in the code, such as the line that logs the message or the path that throws the error. Use that to confirm or rule out a cause.

Keep the code search scoped. Start from files the user referenced or the service in the question. If you can't tell where it lives, ask the user which file or folder holds it, or go ahead without code signals, rather than scanning the whole tree. If the code has nothing to do with the question, skip this step.

## Gotchas

- **Raw queries: `| limit N` must be the FIRST pipe operator**, before any `parse`, `json`, or
  `fields`. A leading limit terminates the scan early (scans MB); a trailing limit scans TB
  and *then* truncates.
- **Aggregate queries must be bounded** — end with `| sort _count desc | limit N` (or `topk`).
  An unbounded `count by` on a high-cardinality field overflows the context window.
- **`parse` / `json` on a field that may be absent needs `nodrop`.** Without it, rows missing
  the field are silently dropped — you lose data with no error and no warning.
- **Caps: 2 minutes runtime, 256MB results.** A query spanning more than ~30 minutes with no
  `_sourceCategory`, `_collector`, `_index`, or `_view` filter is rejected outright (400).
- **`listPartitions` is how you find `_sourceCategory` values** — read each partition's
  `routingExpression`. Don't invent category names and don't probe them one at a time.
- **Never guess metric names or dimension keys.** They vary by organization; a query on a
  name that doesn't exist fails with `metrics:no_results`. Discover them first.
- **Create detection rules with `enabled: false`** and have the user review before enabling.
- **Dates in this skill and its references are placeholders.** Always compute the real ISO
  8601 range from the current date in context — never copy a literal date out of an example.


## Log search workflow

Never skip a step.

Before every `runLogSearch` call, first call `getLogSearchEstimatedUsageByMeteringType` with the same query and time range. If any returned `tier` is `Infrequent` or `Flex`, do not run the query yet. Respond with: "Warning: These tiers are charged by data scanned. It is suggested to monitor queries to these tiers with the Log Search Estimator tool to prevent unexpected scan consumption. Would you like to continue?" Provide the usage details and run the query only if the user confirms.

### 1. DISCOVER — metadata only

Call `listPartitions`, `listCustomFields`, and `listExtractionRules` **in parallel**. Pick
target `_sourceCategory` values from the partition routing expressions before querying.

### 2. SCOPED SAMPLE

One narrow search to confirm the source: `_sourceCategory=<candidate>` plus a keyword,
**≤5 minute window, limit 5**. Read the output for real field names, log format, and
`_collector` values — step 3 needs them.

### 3. TARGETED SEARCH

Build the final query from what steps 1-2 established: confirmed `_sourceCategory`, plus
`_collector` if found, the full time range, field filters matching the observed format, and
the appropriate aggregation. Obey the limit / `nodrop` / bounding rules in Gotchas.

Query shapes, operators, per-domain strategies, and what to do when a query returns nothing:
`references/queries.md`.

## Metrics workflow 

For resource, capacity, latency, throughput, and error-rate questions, or to confirm a
hypothesis from logs. Never skip a step.

### 1. DISCOVER METRIC NAMES

`searchValuesWithinDimension` with `dimension: "metric"` and a `term` from the question
(`cpu`, `mem`, `latency`), over the last 15 minutes. Search both a generic term and the
service name. Add a `selector` (e.g. `_sourceHost=web-01`) to see what a host exposes.

### 2. DISCOVER DIMENSIONS

`searchDimensions` with a `selector` scoped to the chosen metric to find the keys you can
filter and group by (`cluster`, `pod`, `_sourceHost`). Use only keys and values you discovered.

### 3. QUERY

`runMetricsQueries`, built as **metric selector → dimension filters → operators**, e.g.
`metric=<name> cluster=prod | avg by _sourceHost`. Keep scope narrow — never a bare
`metric=*`. Use 1-minute buckets under 1 hour, 5 minutes for multi-hour ranges, 15+ beyond 6
hours.

### 4. INTERPRET

The response has no summary: compute min / max / avg / latest per series yourself, and
compare series by dimension to find the outlier. `metrics:no_results` means nothing matched;
`metrics:incomplete_results` means the result was cut off, so narrow the selector or range.

Signal-to-term table, operators, ratios, counters vs. rates, log/metric correlation, what to
do when nothing returns, and what to say if the tools are unavailable (401/403):
`references/metrics.md`.

## Security questions

Start from `getInsights`, not from logs — an insight already correlates the signals for you.
Workflow, insight query DSL, and rule-creation constraints: `references/siem.md`.

## Time handling

Operate in the user's timezone, ISO 8601. Compute `from`/`to` for relative asks ("last hour",
"yesterday") from the current date in context.

## Asking the user

Ask only when no source expression can be inferred, the data genuinely isn't in the platform,
or you are stuck. One targeted question, and say why it blocks you. Don't re-ask about values
the user already gave you, and don't ask speculative or optional questions.

## Reporting

State what you searched and what you found. Concise by default; expand when asked. No
speculation, no tangents, and no tool names or implementation details in user-facing text.
