# OSLO Sample Dashboard Onboarding Guide

This guide walks you through importing sample SLO dashboards into Yahoo's Grafana Cloud and creating dashboards with real metrics for each supported datasource.

## Prerequisites

- Access to [yahooinc.grafana.net](https://yahooinc.grafana.net) (request via [yo/iiq](http://yo/iiq) if needed)
- Editor access to the [SRE OpenSLO](https://yahooinc.grafana.net/dashboards/f/sre-openslo-folder/sre-openslo) folder
- For CloudWatch: an AWS account [onboarded to Grafana Cloud](https://git.ouryahoo.com/o11y/management/blob/main/grafana-cloud/datasources/cloudwatch/README.md)

## Available Sample Dashboards

Located in `config/grafana/samples/`:

| File | Datasource | Notes |
|------|-----------|-------|
| `sample-chronosphere.json` | Chronosphere | PromQL queries, minimal diff from Prometheus |
| `sample-opentsdb.json` | Yamas-OpenTSDB | OpenTSDB query structure (metric/aggregator/filters) |
| `sample-cloudwatch.json` | CloudWatch | AWS metrics structure (namespace/metricName/dimensions) |

See [`config/grafana/samples/README.md`](config/grafana/samples/README.md) for a detailed comparison of JSON target structures across all three datasources.

## Step 1: Import a Sample Dashboard

1. Log into [yahooinc.grafana.net](https://yahooinc.grafana.net)
2. Navigate to **Dashboards** > **SRE OpenSLO** folder
3. Click **New** > **Import**
4. Open the desired sample JSON file and copy its entire contents
5. Paste into the import dialog and click **Load**
6. Select the matching datasource from the dropdown (e.g., `chronosphere`, `Yamas-OpenTSDB`)
7. Click **Import**

## Step 2: Replace Placeholder Metrics with Real Data

After importing, each panel will have placeholder metrics. You need to edit the queries:

### Chronosphere
- Click any panel > **Edit**
- The datasource should show **chronosphere**
- Replace placeholder queries with real PromQL queries for your service
- Use `{athenz_domain="your.domain"}` to explore available metrics

### OpenTSDB (Yamas)
- Click any panel > **Edit**
- Select **Yamas-OpenTSDB** as the datasource
- Use the query builder to find available metrics (type in the Metric field)
- Set appropriate aggregators, downsample intervals, and tag filters

### CloudWatch
- Click any panel > **Edit**
- Select your **CloudWatch** datasource (named `CloudWatch - <account_id>`)
- Replace placeholder dimensions (`LoadBalancer`, `ClusterName`) with your actual AWS resources
- Set the correct region

## Step 3: Export the Final JSON

This is the key deliverable -- the exported JSON shows the exact query structure the Python generator needs to produce.

1. Open the dashboard you edited
2. Click the **gear icon** (Dashboard settings)
3. Click **JSON Model** in the left sidebar
4. Copy the JSON and save it locally
5. Share with the team working on [SRE-26277](https://ouryahoo.atlassian.net/browse/SRE-26277) (Python script enhancement)

## Key Datasource Differences

| Aspect | Chronosphere | OpenTSDB | CloudWatch |
|--------|-------------|----------|------------|
| Query field | `expr` (PromQL) | `metric` + `aggregator` + `filters` | `namespace` + `metricName` + `dimensions` |
| Ratio math | Native PromQL | Grafana transforms needed | Grafana math expressions needed |
| Percentiles | `histogram_quantile()` | `percentile` aggregator | `p50`/`p95`/`p99` in `statistic` |
| Datasource type | `prometheus` | `opentsdb` | `cloudwatch` |

## Related Resources

- [Project OSLO TDD](https://docs.google.com/document/d/1A6PrCvsC6sQWmGwWr0RRWTeDblAC2dtDOGBTHJWjPow) -- Full technical design
- [SRE-26277](https://ouryahoo.atlassian.net/browse/SRE-26277) -- Python script enhancement ticket
- [SRE-25826](https://ouryahoo.atlassian.net/browse/SRE-25826) -- Collab/Build SLO Dashboards epic
- [CloudWatch onboarding](https://git.ouryahoo.com/o11y/management/blob/main/grafana-cloud/datasources/cloudwatch/README.md)
- [Grafana Cloud access](https://git.ouryahoo.com/pages/o11y/otel-guide/platforms/grafana-cloud/)
