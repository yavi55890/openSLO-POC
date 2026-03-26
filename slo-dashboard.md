# SLO Dashboard — Codebase Reference

A Python package that converts OpenSLO-style SLI/SLO definitions into Grafana dashboards. It ships two CLIs — one to generate OpenSLO YAML from a compact manifest, another to turn that YAML into Grafana JSON — plus a library class for uploading dashboards to Grafana's HTTP API.

---

## Directory Structure

```
slo-dashboard/
├── .claude/
│   └── settings.json                  # Claude Code tool allowlists
├── .github/
│   ├── dependabot.yml                 # Weekly pip + Docker dependency updates
│   └── workflows/
│       └── security-workflow.yml      # Dependency review & IAC SAST (GitHub Actions)
├── example/
│   ├── openslo.yaml                   # Sample manifest (2 indicators: ratio + threshold)
│   └── openslo/                       # Generated OpenSLO output from the example
│       ├── sli-availability.yaml
│       ├── slo-availability.yaml
│       ├── sli-latency-p99.yaml
│       └── slo-latency-p99.yaml
├── src/yahoo/sre/                     # Namespace package root (NO __init__.py by design)
│   ├── grafana/
│   │   ├── __init__.py
│   │   ├── dashboard_generator.py     # dashboard-generator CLI entry point
│   │   ├── datasource_adapters.py     # MetricSource ABC + Prometheus/Chronosphere/OpenTSDB
│   │   ├── grafana_types.py           # GrafanaDashboard / GrafanaPanel / GrafanaTarget dataclasses
│   │   └── grafana_uploader.py        # GrafanaUploader: POST dashboards to Grafana HTTP API
│   └── openslo/
│       ├── __init__.py
│       ├── manifest_generator.py      # openslo-config-generator CLI entry point
│       └── openslo_types.py           # SLI / SLO / DataSource dataclasses (from_dict parsers)
├── tests/
│   ├── __init__.py
│   ├── README.md
│   ├── test_dashboard_generator.py
│   ├── test_datasource_adapters.py
│   ├── test_grafana_types.py
│   ├── test_grafana_uploader.py
│   ├── test_manifest_generator.py
│   └── test_openslo_types.py
├── .gitignore
├── CLAUDE.md                          # Agent conventions, architecture notes, CI gotchas
├── Dockerfile                         # Internal RHEL8 image — pip-installs published package
├── README.md                          # User-facing docs + CLI reference
├── deploy.conf                        # RPM packaging metadata for Screwdriver
├── pyproject.toml                     # Build backend (setuptools) + sdv4 stubs
├── screwdriver.yaml                   # CI pipeline: validation → package → RPM
├── setup.cfg                          # Package metadata, deps, entry points, lint config
└── tox.ini                            # Local test + lint environments (py311, py312)
```

---

## Namespace Package Convention

`src/yahoo/` and `src/yahoo/sre/` do **not** have `__init__.py` files. They are pure namespace packages so the `yahoo` namespace can merge with other Yahoo packages in a shared Python environment. Never add `__init__.py` to these directories.

---

## Source Files — What Each One Does

### `src/yahoo/sre/openslo/openslo_types.py`

The domain model. Defines Python dataclasses that map 1:1 to the [OpenSLO v1](https://openslo.com/) YAML schema. Every class has a `from_dict(data)` class method that hydrates it from a parsed YAML dict.

| Class | Purpose |
|---|---|
| `Metadata` | `name`, `displayName`, `labels` dict, `annotations` dict |
| `MetricSourceSpec` | Wraps a metric query string |
| `MetricSourceDef` | Either `type` (inline, e.g. `"Prometheus"`) or `data_source_ref` (name of a DataSource object) + a `MetricSourceSpec` |
| `RatioMetricPart` | One side of a ratio (good or total), containing a `MetricSourceDef` |
| `RatioMetric` | `counter` flag + `good` + `total` parts |
| `ThresholdMetric` | Single metric with a threshold comparison |
| `SLISpec` | Either `ratio_metric` or `threshold_metric` (or both `None`) |
| `SLI` | Top-level SLI object. Helpers: `get_metric_source_type()` (inline type string), `get_datasource_ref()` (DataSource reference name) |
| `SLOObjective` | `target`, `time_slice_target`, `time_slice_window`, `op` |
| `SLOTimeWindow` | `duration` + `is_rolling` |
| `SLOIndicator` | Pointer from an SLO to its SLI via `metadata.name` |
| `SLOSpec` | `service`, `indicator`, `budgeting_method`, `objectives`, `time_windows` |
| `SLO` | Top-level SLO object |
| `DataSourceConnectionDetail` | Connection URL |
| `DataSourceSpec` | `type` (e.g. `"Prometheus"`) + connection details |
| `DataSource` | Top-level DataSource object. `get_grafana_uid()` reads `metadata.annotations["grafana-uid"]` |

The helper function `_get_time_windows(data)` handles backward compatibility — it prefers the `timeWindow` key but falls back to the deprecated `timeWindows` key with a logged warning.

### `src/yahoo/sre/openslo/manifest_generator.py`

CLI: **`openslo-config-generator`**

Takes a compact YAML manifest (see `example/openslo.yaml`) and expands it into individual `sli-{name}.yaml` / `slo-{name}.yaml` files. This is the "authoring" step — you write one manifest, and this tool produces valid OpenSLO objects.

| Function | Role |
|---|---|
| `_deep_merge(base, override)` | Recursive dict merge; override wins at leaf level |
| `_resolve(defaults, indicator)` | Extracts top-level override keys from an indicator entry and merges them into defaults |
| `_build_sli(cfg, indicator)` | Builds an SLI dict from resolved config. Handles `ratio` (requires `good`/`total`) and `threshold` (requires `query`) metric types |
| `_build_slo(cfg, indicator)` | Builds an SLO dict. Links to the SLI via `indicator.metadata.name: "{name}-sli"`. Defaults: `budgetingMethod: Occurrences`, `timeWindows: [{duration: 30d, isRolling: true}]` |
| `_write_indicator(output_dir, name, sli, slo)` | Writes YAML files with a generated header comment |
| `generate(manifest_path, output_dir)` | Main logic. Validates all indicators *before* writing any files (fail-closed: no partial output) |
| `main()` | argparse entry point (`--manifest`, `--output-dir`). Catches `ValueError`/`FileNotFoundError` → exit 1 |

**Required `defaults` fields:** `apiVersion`, `product_id`, `team`, `owner`, `service`.

**Validation rules:**
- Indicator names must match `^[a-zA-Z0-9][a-zA-Z0-9_-]*$`
- No duplicate names
- Each indicator must have both `sli` and `slo` sections
- Ratio SLIs need `good` + `total`; threshold SLIs need `query`

### `src/yahoo/sre/grafana/datasource_adapters.py`

Strategy pattern for metric backends. Defines how queries are validated, transformed, and what Grafana plugin type to use.

| Class | Type | Plugin | Query Language | Histograms |
|---|---|---|---|---|
| `MetricSource` (ABC) | — | — | — | — |
| `PromQLSource` (base) | — | `prometheus` | `PromQL` | Yes |
| `PrometheusSource` | `Prometheus` | `prometheus` | `PromQL` | Yes |
| `ChronosphereSource` | `Chronosphere` | `prometheus` | `PromQL` | Yes |
| `OpenTSDBSource` | `OpenTSDB` | `opentsdb` | `OpenTSDB` | No |

`PromQLSource.validate_query()` checks for empty queries and unbalanced parentheses/braces. `OpenTSDBSource` only checks for empty queries.

`get_metric_source(source_type, datasource_name)` is the factory function — pass `"Prometheus"`, `"Chronosphere"`, or `"OpenTSDB"`.

### `src/yahoo/sre/grafana/grafana_types.py`

Typed dataclasses for Grafana dashboard JSON. Each has a `to_dict()` method that produces the camelCase JSON Grafana expects.

| Class | Fields | Notes |
|---|---|---|
| `GrafanaTarget` | `expr`, `legend_format`, `ref_id` | Produces `{"expr": ..., "refId": ..., "legendFormat": ...}` |
| `GrafanaPanel` | `field_config`, `grid_pos`, `id`, `options`, `targets`, `title`, `type`, `datasource`, etc. | `to_dict()` propagates `datasource` down to each target's dict if set |
| `GrafanaDashboard` | All top-level Grafana JSON fields | `panels` can be `dict`s or `GrafanaPanel` instances — `to_dict()` handles both |

### `src/yahoo/sre/grafana/dashboard_generator.py`

CLI: **`dashboard-generator`**

The core tool. Reads a directory of OpenSLO YAML files and produces a Grafana dashboard JSON file.

**Key functions:**

| Function | Role |
|---|---|
| `_replace_rate_window(query, new_window)` | Regex-replaces `[5m]`-style windows in PromQL to `[1h]`, `[6h]`, `[24h]` for multi-window burn rate |
| `_row_panel(...)` | Builds a Grafana row panel dict |
| `_stat_panel(...)` | Builds a stat panel with thresholds, mappings, optional datasource |
| `_timeseries_panel(...)` | Builds a timeseries panel with overrides, thresholds |
| `_text_panel(...)` | Builds a markdown text panel |
| `load_openslo_files(dir)` | Scans `*.yaml`/`*.yml`, dispatches by `kind` to `SLI.from_dict` / `SLO.from_dict` / `DataSource.from_dict` |
| `parse_args()` | argparse for the `dashboard-generator` CLI |
| `main()` | Orchestrates everything: load → resolve datasource → create MetricSource → generate → write JSON |

**`DashboardGenerator` class:**

| Method | What it does |
|---|---|
| `__init__(metric_source, slis, slos, datasource_uid)` | Stores inputs |
| `_build_datasource_dict()` | Returns `{"type": "<plugin>", "uid": "<uid>"}` for Grafana |
| `get_product_id()` | Scans SLO labels first, then SLI labels for `product_id` |
| `generate_dashboard()` | Builds the full multi-SLO dashboard. Only SLOs whose linked SLI has `ratio_metric` get full sections. Target of `1.0` (zero error budget) is skipped. First SLO section is expanded; subsequent sections are collapsed rows |
| `_build_slo_section(slo, sli, ...)` | Builds all panels for one SLO: Scoreboard (6 stats), SLI Over Time (2 timeseries), Error Budget Economics (2 timeseries), SLO Metadata (1 text). Returns panels + next y-offset |
| `_generate_minimal_dashboard()` | Fallback when no ratio-metric SLO exists — produces a single "No SLO Data" text panel |
| `_build_slo_info_markdown(slo, sli, ...)` | Generates the markdown content for the SLO Info text panel (overview, labels, annotations) |

**Dashboard layout per SLO section (31 grid rows):**

```
y+0    Collapsible row header ("{SLO Name} (target: XX.X%)")
y+1    Scoreboard row
y+2    [Objective] [Current SLI] [Error Budget] [Burn Short] [Burn Window] [SLO Status*]
y+6    SLI Over Time row
y+7    [Success Ratio timeseries] [Error Ratio timeseries]
y+15   Error Budget Economics row
y+16   [Budget Remaining timeseries] [Burn Rate by Window timeseries]
y+24   SLO Metadata row
y+25   [SLO Info text panel]
y+31   (next section starts)
```

\* SLO Status panel only appears for PromQL sources (uses the `bool` modifier).

**Datasource resolution order:**
1. Check SLI `dataSourceRef` → look up matching `DataSource` YAML
2. Fall back to CLI positional args (`datasource_type`, `datasource_name`)
3. UID: `--datasource-uid` flag > `GRAFANA_DATASOURCE_UID` env var > `DataSource.metadata.annotations["grafana-uid"]`

**Output:** `{product_id}.json` in the output directory. Exits 1 if no `product_id` label is found.

### `src/yahoo/sre/grafana/grafana_uploader.py`

Library class (no CLI entry point) for POSTing dashboard JSON to Grafana's HTTP API.

| Method | What it does |
|---|---|
| `__init__(grafana_url, api_key, folder_name)` | Stores URL, sets Bearer auth header |
| `get_or_create_folder()` | POST to create folder; on 409 (exists), GET all folders and find by title. Returns folder UID |
| `upload_dashboard(path, folder_uid)` | Reads JSON from disk, POSTs to `/api/dashboards/db` with `overwrite: true` |
| `run(path)` | Convenience: creates folder + uploads. Raises `FileNotFoundError` if path doesn't exist |

All HTTP calls use `timeout=30`.

---

## Test Files

All tests live in `tests/` and run with pytest (some use `unittest.TestCase`).

| File | What it covers |
|---|---|
| `test_openslo_types.py` | Parsing all dataclasses from dicts, `SLI` helper methods, deprecated `timeWindows` warning, `DataSource.get_grafana_uid` |
| `test_manifest_generator.py` | `_deep_merge`, `_resolve`, `generate()` validation and output structure, `main()` exit codes. Uses `unittest.TestCase` + `tempfile` |
| `test_datasource_adapters.py` | Factory function, each `MetricSource` implementation's behavior (validation, transform, type strings) |
| `test_grafana_types.py` | `to_dict()` for `GrafanaTarget`, `GrafanaPanel`, `GrafanaDashboard` |
| `test_grafana_uploader.py` | Mocked HTTP: folder create/exists, upload success/failure, `run()` FileNotFoundError |
| `test_dashboard_generator.py` | `load_openslo_files`, datasource resolution, `DashboardGenerator.get_product_id`, dashboard structure, multi-SLO collapsed rows, datasource UID propagation, JSON serializability, skip-full-budget SLO, malformed YAML handling |

---

## Configuration Files

### `setup.cfg`

- **Package name:** `yahoo.sre.slo_dashboard`
- **Python requires:** `>= 3.12`
- **Runtime deps:** `pyyaml`, `requests`
- **Entry points:**
  - `openslo-config-generator` → `yahoo.sre.openslo.manifest_generator:main`
  - `dashboard-generator` → `yahoo.sre.grafana.dashboard_generator:main`
- **Lint config:** max line length 130, pylint disables `W0401` (wildcard import) and `W0703` (broad exception)

### `tox.ini`

- **Default envs:** `py311`, `py312` — runs `pytest tests/ -v`
- **`lint_mypy`:** mypy with `--namespace-packages --ignore-missing-imports`, outputs to `artifacts/mypy`
- **`lint_pylint`:** pylint with parseable output
- **`lint_pycodestyle`:** pycodestyle check
- Lint envs use `/opt/y/1.0/bin/python3.12` (Yahoo internal Python path)

### `pyproject.toml`

Declares `setuptools.build_meta` as the build backend. Contains empty `[tool.sdv4_installdeps]` sections for the Yahoo build system.

### `screwdriver.yaml`

Yahoo's CI/CD pipeline (Screwdriver v4):

1. **`validation`** — runs on every PR and commit. Uses `python-2504/validate_multiple@latest` which executes bandit, pylint, packaging checks, mypy, and unit tests.
2. **`python_package`** — publishes to internal PyPI after validation passes. Uses `python-2504/package_python@latest`.
3. **`package_rpm_alma9`** — builds an RPM on Alma Linux 9 after the Python package is published.

Shared config: `SCREWDRIVER_CONFIG_PRESET: strict`, `TOX_ENVLIST: py311,py312`, k8s executor.

### `deploy.conf`

RPM packaging metadata: package name `yahoo_sre_slo_dashboard`, depends on the pip package at `{{version}}`, targets `y1.0-python312`.

### `Dockerfile`

Based on `docker.ouroath.com:4443/pythonapp/rhel8:latest`. Installs `yahoo-sre-slo-dashboard==0.0.1` from internal Artifactory. CMD runs `python -m yahoo.sre.slo_dashboard` (note: this module path doesn't match the actual package layout — likely needs updating).

### `.github/dependabot.yml`

Weekly automated updates for Docker base images and pip dependencies.

### `.github/workflows/security-workflow.yml`

GitHub Actions on PR/push: dependency review (fail on high severity, license deny list) and IAC SAST via `yahoo-paranoids/actions/iac-sast`. Runs on `yahoo-enterprise-alma9-x86` runners.

---

## Data Flow

### Flow 1: Manifest → OpenSLO YAML

```
openslo.yaml (compact manifest)
    │
    ▼
openslo-config-generator CLI
    │  reads defaults + indicators
    │  validates all indicators (fail-closed)
    │  deep-merges per-indicator overrides
    │  builds SLI + SLO dicts
    ▼
openslo/
  sli-{name}.yaml    ← one per indicator
  slo-{name}.yaml    ← one per indicator
```

### Flow 2: OpenSLO YAML → Grafana JSON

```
openslo/ directory
  sli-*.yaml, slo-*.yaml, datasource.yaml (optional)
    │
    ▼
dashboard-generator CLI
    │  load_openslo_files() → SLI, SLO, DataSource objects
    │  resolve datasource (YAML ref > CLI args)
    │  get_metric_source() → MetricSource strategy
    │
    ▼
DashboardGenerator
    │  match SLOs to SLIs via indicator.metadata.name
    │  skip non-ratio SLIs, skip target=1.0
    │  build panel dicts (stat, timeseries, text, row)
    │  wrap in GrafanaDashboard
    ▼
{product_id}.json    ← Grafana-importable dashboard
```

### Flow 3: Grafana JSON → Grafana API (library only)

```
{product_id}.json
    │
    ▼
GrafanaUploader
    │  get_or_create_folder() → folder UID
    │  upload_dashboard() → POST /api/dashboards/db
    ▼
Grafana instance
```

---

## Key Design Decisions

1. **Hybrid panel model** — `DashboardGenerator` builds panels as raw `dict`s (not `GrafanaPanel` instances) for flexibility. `GrafanaDashboard.to_dict()` accepts both, calling `.to_dict()` on objects and passing dicts through.

2. **Ratio-only dashboard sections** — only SLOs whose linked SLI has a `ratioMetric` get full scoreboard/burn-rate panels. Threshold SLIs produce valid OpenSLO but don't drive the dashboard (they'd need a fundamentally different panel layout).

3. **Fail-closed manifest generation** — all indicators are validated before any files are written, preventing a partially-populated output directory from a bad manifest.

4. **Multi-window burn rates** — the generator uses regex to replace `[5m]` rate windows with `[1h]`, `[6h]`, and `[24h]` for multi-window burn rate calculations, following Google SRE alerting practices.

5. **Strategy pattern for datasources** — `MetricSource` ABC lets each backend define its own validation, query transformation, Grafana plugin type, and feature flags (histogram support, query language) without conditionals in the generator.

6. **Collapsed row sections** — in multi-SLO dashboards, the first SLO section is expanded and all subsequent sections are collapsed Grafana rows to keep the dashboard navigable.

---

## CLI Quick Reference

```bash
# Generate OpenSLO YAML from a manifest
openslo-config-generator --manifest example/openslo.yaml --output-dir example/openslo

# Generate Grafana dashboard JSON from OpenSLO YAML
dashboard-generator example/openslo/ example/ Prometheus

# Run tests
tox                              # full CI matrix (py311 + py312)
python -m pytest tests/ -v       # quick local run

# Lint
tox -e lint_pylint
tox -e lint_mypy
tox -e lint_pycodestyle
```

---

## Dependencies

| Dependency | Purpose |
|---|---|
| `pyyaml` | YAML parsing (manifest + OpenSLO files) and YAML generation |
| `requests` | HTTP calls to Grafana API (used by `GrafanaUploader`) |
| `pytest` | Test runner |
| `mypy` | Static type checking |
| `pylint` | Linting |
| `pycodestyle` | PEP 8 style checking |
