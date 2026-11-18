# Metrics reference

Detail behind the "Metrics workflow" in SKILL.md (which holds the mandatory steps). Load this
when picking discovery terms, building a non-trivial query, correlating with logs, or when a
query returns nothing.

**Never guess metric names or dimension keys** — they vary by organization, and a query on a
name that doesn't exist fails with `metrics:no_results`. Discover first.

## Tools

- `searchValuesWithinDimension` — list the values of one dimension; pass `dimension: "metric"`
  to list metric names
- `searchDimensions` — list the dimension keys (`cluster`, `pod`, `_sourceHost`, ...) on metrics
- `runMetricsQueries` — run one or more metrics queries over a time range and return the series

Neither discovery tool has a `limit` or reports truncation, so narrow with `term` and
`selector` instead of paging.

## Step details

### 1. DISCOVER METRIC NAMES

Call `searchValuesWithinDimension` with `dimension: "metric"` and a `term` taken from the
question, over a recent window (start with the last 15 minutes).

| Signal | Discovery `term` |
|--------|------------------|
| OOM kills, memory pressure | `mem`, `memory` |
| High latency, slow requests | `latency`, `duration`, `cpu` |
| Connection refused, timeouts | `net`, `tcp` |
| Disk full | `disk`, `filesystem` |
| JVM GC pauses | `jvm`, `gc`, `heap` |
| HTTP 5xx | `http` |
| Container restarts | `container`, `kube` |
| Throttling | `throttl`, `capacity`, `limit` |

- Search BOTH a generic term (`cpu`, `memory`) AND the service name — services often expose
  their own instrumentation.
- To see which metrics exist for a host or service, add a `selector` (e.g.
  `_sourceHost=web-01`). A selector naming a dimension or value that doesn't exist returns
  empty results.

### 2. DISCOVER DIMENSIONS

Call `searchDimensions` with a `selector` scoped to the chosen metric to see which keys you
can filter and group by. To see the values of one key, call `searchValuesWithinDimension` with
that key as `dimension` and the same `selector`. Use only keys and values you discovered.

### 3. QUERY

Call `runMetricsQueries`. Build each query as **metric selector → dimension filters →
operators**, e.g. `metric=<name> cluster=prod | avg by _sourceHost`.

- **Filters** go before the first `|`: exact `cluster=prod`, wildcard `_sourceHost=web-*`,
  negation `cluster!=staging`. Multiple filters are ANDed.
- **Operators** go after the `|`: `avg`, `sum`, `min`, `max`, `count` (optionally `by <dim>`),
  `topk(5, avg)`, `bottomk(3, max)`, `rate`, `smooth(5)`, `eval _value * 100`.
- **Bucket size by range:** 1 minute for under 1 hour (and brief incidents), 5 minutes for
  multi-hour ranges, 15 minutes or more beyond 6 hours.
- **Ratios** combine rows, e.g. error ratio: row A `metric=HTTP_5xx | sum`, row B
  `metric=HTTP_Total | sum`, row C `#A / #B * 100`.
- Keep scope narrow (specific metric, specific dimensions). Never use a bare `metric=*`.

### 4. INTERPRET

The response has no precomputed summary, so compute min / max / avg / latest per series from
the returned values:

- `max` far above `avg` means spikes — find when from the timestamps
- Few points relative to the range means data gaps
- Compare series by their dimensions to find the outlier (one pod, one host)

Report what metric, what value, over what period, and what it means. `metrics:no_results`
means nothing matched; `metrics:incomplete_results` means the result was cut off, so narrow
the selector or range.

## Investigation strategies

- **Resource, capacity, or latency:** discover names → discover dimensions → query with
  `avg by` / `topk` over the incident window → break down by the dimension that stands out.
- **Correlating logs and metrics:** find the event in logs (time, host/pod/service) → take the
  shared identifier (`_sourceHost`, `pod`, `cluster`, `container`) → discover metrics for it →
  query around the event time → report whether the metric explains the log symptom.

Tips:

- **Disaggregate early.** Break down by serving infrastructure (pod, shard, node) and by
  caller or workload. A single outlier hides in averages.
- **Break down by container** when pods are involved — the failing container may be a sidecar.
  Query container-level metrics (memory usage, restart count, termination reason) from the
  first call.
- **Use 1-minute buckets** for brief disruptions.
- **Counters vs rates:** `rate` on a cumulative counter gives throughput, which drops when a
  service is saturated. To measure load, use raw counter growth or a separate inbound-request
  metric.
- Metrics and logs complement each other — don't stop at one when the other could confirm the
  finding.

## Worked example — "is CPU high on our web hosts right now?"

```
1. NAMES       searchValuesWithinDimension dimension="metric" term="cpu" (last 15 min)
2. DIMENSIONS  searchDimensions selector=["metric=CPU_Idle"] (same window)
               → host key is `_sourceHost`
3. QUERY       runMetricsQueries: metric=CPU_Idle _sourceHost=web-* | avg by _sourceHost
               (1-minute buckets, last hour)
4. INTERPRET   per-host min/max/avg/latest; report the busiest hosts (CPU busy = 100 − idle)
```

## When metrics return nothing

1. Don't retry the same guessed name. Re-run step 1 with a broader `term` or a wider window
   (15 minutes → 1 hour → 6 hours → 24 hours).
2. Re-check dimension keys and values with step 2 — a selector naming a key or value that
   doesn't exist returns nothing.
3. If metrics still don't cover the question, say so and fall back to the log search workflow.

## If the metrics tools are unavailable

If any metrics tool is missing from your tool list, or returns 401/403, stop using them and
tell the user: "Metrics queries are not available for your account. Ask your Sumo Logic admin
to add the **Run Metrics Query** capability to your role."
