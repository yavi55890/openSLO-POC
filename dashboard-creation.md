# Sample SLO Dashboards by Datasource

This directory contains sample Grafana dashboard JSON files for each supported datasource. These are **reference implementations** showing the exact JSON structure each datasource requires, intended to guide the enhancement of the Python `dashboard_generator.py` script ([SRE-26277](https://ouryahoo.atlassian.net/browse/SRE-26277)).

## Files

| File | Datasource | Query Language | Key JSON Difference |
|------|-----------|----------------|---------------------|
| `sample-chronosphere.json` | Chronosphere | PromQL | Same `expr` field as Prometheus; only datasource UID changes |
| `sample-opentsdb.json` | Yamas-OpenTSDB | OpenTSDB | Uses `metric`, `aggregator`, `downsampleInterval`, `filters` instead of `expr` |
| `sample-cloudwatch.json` | CloudWatch | CloudWatch Metrics | Uses `namespace`, `metricName`, `dimensions`, `statistic`, `period`, `region` |

## How the Target Structures Differ

### Chronosphere (PromQL)

Chronosphere is PromQL-compatible. The JSON target structure is essentially identical to Prometheus:

```json
{
  "datasource": { "type": "prometheus", "uid": "chronosphere" },
  "editorMode": "code",
  "expr": "sum(rate(http_requests_total{status_code=~\"2..\"}[5m]))",
  "legendFormat": "Successful",
  "range": true,
  "refId": "A"
}
```

### OpenTSDB (Yamas)

OpenTSDB uses a completely different structure -- no `expr` field at all:

```json
{
  "datasource": { "type": "opentsdb", "uid": "yamas-opentsdb" },
  "aggregator": "sum",
  "alias": "Total Requests",
  "downsampleAggregator": "avg",
  "downsampleFillPolicy": "none",
  "downsampleInterval": "5m",
  "filters": [
    {
      "filter": "2*",
      "groupBy": false,
      "tagk": "status_code",
      "type": "wildcard"
    }
  ],
  "metric": "http.requests.count",
  "refId": "A"
}
```

### CloudWatch

CloudWatch is structured around AWS concepts -- namespaces, metrics, dimensions:

```json
{
  "datasource": { "type": "cloudwatch", "uid": "cloudwatch-ACCOUNT_ID" },
  "alias": "Request Count",
  "dimensions": {
    "LoadBalancer": ["app/my-alb/1234567890abcdef"]
  },
  "matchExact": true,
  "metricEditorMode": 0,
  "metricName": "RequestCount",
  "metricQueryType": 0,
  "namespace": "AWS/ApplicationELB",
  "period": "300",
  "queryMode": "Metrics",
  "refId": "A",
  "region": "us-east-1",
  "statistic": "Sum"
}
```

## Importing into Yahoo Grafana Cloud

### Option A: Import JSON (Fastest)

1. Go to [yahooinc.grafana.net](https://yahooinc.grafana.net)
2. Navigate to the **SRE OpenSLO** folder
3. Click **New** > **Import**
4. Paste the JSON content from one of the sample files
5. Click **Load**, then select the correct datasource from the dropdown
6. Click **Import**

**After importing, you must:**
- Update the datasource UID to match your actual Grafana Cloud datasource
- Replace placeholder metric names / dimensions with real ones
- Save the dashboard

### Option B: Build Manually in Grafana UI

This approach is better for discovering real metrics interactively.

#### Chronosphere

1. Create a new dashboard in the SRE OpenSLO folder
2. Add a panel, select **chronosphere** as the datasource
3. Switch to **Code** mode in the query editor
4. Enter PromQL queries (e.g., `{athenz_domain="your.domain"}` to explore available metrics)
5. Use metrics like `http_server_request_duration_seconds`, `http_server_active_requests`
6. Build panels matching the SLO dashboard layout: gauge, timeseries for burn rate, RED metrics

#### OpenTSDB (Yamas)

1. Create a new dashboard in the SRE OpenSLO folder
2. Add a panel, select **Yamas-OpenTSDB** as the datasource
3. Use the visual query builder (it has dropdowns for metric, aggregator, downsample, tags)
4. Start typing metric names in the **Metric** field to discover what's available
5. Set aggregator (sum, avg, etc.), downsample interval, and tag filters
6. Build panels for request counts, latency, error rates

**Key differences from PromQL:**
- No `rate()` function -- OpenTSDB uses its own rate calculation via the `rate` option in the query
- No `histogram_quantile()` -- use pre-aggregated percentile metrics or the `percentile` aggregator
- Tag filters replace label matchers (e.g., `{status_code=~"2.."}` becomes a wildcard filter on tag `status_code`)
- Metric names use dots instead of underscores (convention: `http.requests.count` vs `http_requests_total`)

#### CloudWatch

1. Create a new dashboard in the SRE OpenSLO folder
2. Add a panel, select your **CloudWatch** datasource (named like `CloudWatch - <account_id>`)
3. Select **Region** (e.g., `us-east-1`)
4. Select **Namespace** (e.g., `AWS/ApplicationELB`, `AWS/ECS`)
5. Select **Metric name** (e.g., `RequestCount`, `TargetResponseTime`)
6. Add **Dimensions** to filter (e.g., `LoadBalancer`, `TargetGroup`)
7. Set **Statistic** (`Sum`, `Average`, `p50`, `p95`, `p99`)
8. Set **Period** (e.g., `300` for 5-minute intervals)

**Key differences from PromQL:**
- No free-form query language -- everything is selected via dropdowns
- Latency percentiles use the `statistic` field (e.g., `p95`) instead of `histogram_quantile()`
- Ratio calculations (availability %) require Grafana math expressions, not inline computation
- `dimensions` replace label matchers -- each dimension is a key-value filter on AWS resource attributes

### Exporting Dashboard JSON (After Manual Creation)

This is the most important step -- the exported JSON is what the Python script needs to replicate.

1. Open the dashboard you created
2. Click the **gear icon** (Dashboard settings) in the top-right
3. Click **JSON Model** in the left sidebar
4. Copy the entire JSON
5. Save it as a file (e.g., `exported-chronosphere.json`)
6. **Compare the `targets` arrays** across all three exports -- this shows exactly what the Python generator needs to produce per datasource

## Datasource Availability in Yahoo Grafana Cloud

| Datasource | Status | How to Access |
|-----------|--------|--------------|
| Chronosphere | Available (managed by O11y team) | Already provisioned -- just select it |
| Yamas-OpenTSDB | Available (managed by O11y team) | Already provisioned -- just select it |
| CloudWatch | Requires onboarding | Submit PR to [o11y/management](https://git.ouryahoo.com/o11y/management/blob/main/grafana-cloud/datasources/cloudwatch/README.md) |

## Limitations by Datasource

| Capability | Chronosphere | OpenTSDB | CloudWatch |
|-----------|-------------|----------|------------|
| Ratio queries (good/total) | Native (PromQL) | Requires Grafana transforms | Requires Grafana math expressions |
| Histogram percentiles | `histogram_quantile()` | Pre-aggregated or `percentile` aggregator | `p50`/`p95`/`p99` statistic |
| Burn rate calculations | Native (PromQL math) | Requires external pre-computation | Requires Grafana math expressions |
| Group-by / label filtering | PromQL label matchers | Tag filters with wildcards | Dimension key-value pairs |
| Rate calculation | `rate()` / `irate()` | Built-in rate option | Period-based aggregation |
