# PR #6 Review — OpenSLO Templates & Manifest Generator Changes

**PR:** [yahoo-sre/slo-dashboard#6](https://github.com/yahoo-sre/slo-dashboard/pull/6)
**Author:** Ryan Dolce (`dolce_yahoo`)
**Branch:** `SRE-27204` → `main`
**Jira:** SRE-26121, SRE-26122, SRE-27209
**Filed:** Mar 25, 2026 — Status: Open (awaiting review)

---

## What Was Added

### 1. `templates/openslo/openslo.yaml` — The Manifest Template

This is the file a customer fills out. It replaces the old `example/openslo.yaml` with a
heavily-annotated template that documents every field, its requiredness, and where the
value ends up after generation.

**New required `defaults` fields:**

| Field | Purpose | Lands in |
|---|---|---|
| `apiVersion` | Must be `openslo/v1` | SLI + SLO top-level |
| `product_id` | Yahoo numeric product ID (quoted string) | SLI + SLO label |
| `service` | Service name | SLO `spec.service` |
| `corp_application` | Corp application name | SLI + SLO label |
| `corp_environment` | Corp environment name | SLI + SLO label |
| `criticality` | Criticality level | SLI + SLO label |
| `data_classification` | Public, private, etc. | SLI + SLO label |
| `corp_owner` | Slack channel or email | SLI + SLO label |

These replace the old `team` and `owner` fields.

**New optional fields:**

| Field | Purpose | Lands in |
|---|---|---|
| `product_version` | Product version string | SLI + SLO annotation (only when present) |
| `budgetingMethod` | `Occurrences` / `Timeslices` / `RatioTimeslices` | SLO `spec.budgetingMethod` |
| `timeWindow` | Default evaluation window (e.g. `{duration: 30d, isRolling: true}`) | SLO `spec.timeWindow` |

**Template structure:**

The template provides three commented examples:
- **Option A (ratio):** An active, uncommented `availability` indicator with `good`/`total` queries
- **Option B (threshold):** A fully commented `latency-p99` indicator with a single `query`
- **Per-indicator overrides:** A commented `critical-checkout` example showing how to override
  `service`, `corp_owner`, `budgetingMethod`, and `timeWindow` for a single indicator

### 2. `templates/openslo/sli.yaml` — SLI Reference Template

A standalone OpenSLO SLI template showing what the generator *produces* (or what someone
would write by hand). It documents four metric variants:

- **Option A:** Ratio metric with `good` / `total` (active, uncommented)
- **Option B:** Ratio metric with `bad` / `total` (commented)
- **Option C:** Precomputed ratio with `rawType` + `raw` (commented)
- **Option D:** Threshold metric (commented)

The labels section mirrors the new `corp_*` fields. Annotations show `product_version`
as optional.

### 3. `templates/openslo/slo.yaml` — SLO Reference Template

A standalone OpenSLO SLO template showing what the generator *produces*. It documents:

- **Indicator linking:** inline `indicator.metadata.name` vs `indicatorRef`
- **Time windows:** rolling vs calendar-aligned
- **Budgeting methods:** Occurrences, Timeslices, RatioTimeslices
- **Objectives:** `target` vs `targetPercent`, `op`/`value` for thresholds,
  `timeSliceTarget`/`timeSliceWindow` for time-sliced budgeting
- **Alert policies:** Fully commented example (marked `[NOT IMPLEMENTED]`)

---

## What Was Changed

### `manifest_generator.py`

| Change | Before | After |
|---|---|---|
| Required defaults | `apiVersion`, `product_id`, `team`, `owner`, `service` | `apiVersion`, `product_id`, `service`, `corp_application`, `corp_environment`, `criticality`, `data_classification`, `corp_owner` |
| SLI labels | `product_id`, `team` | `product_id`, `corp_application`, `corp_environment`, `criticality`, `data_classification`, `corp_owner` |
| SLI annotations | `owner` (always present) | `product_version` (only when set) |
| SLO labels | `product_id` | `product_id`, `corp_application`, `corp_environment`, `criticality`, `data_classification`, `corp_owner` |
| SLO annotations | `alert-channel`, `escalation-policy`, `business-impact` | `product_version` (optional), `business-impact` |
| SLO time window key | `timeWindows` (deprecated) | `timeWindow` (preferred) |

### `CLAUDE.md`

Rewritten with more structure: project overview, dev commands, code quality, CI/CD section,
updated architecture tree, updated required defaults list. Adds `black` formatter references.
Removes the CI failure modes section and the `namespace_packages` note.

### `example/` directory

Deleted entirely: `example/openslo.yaml` and all four generated files under `example/openslo/`.
Replaced by `templates/openslo/`.

### `tests/test_manifest_generator.py`

All test fixtures and assertions updated to use `corp_*` fields instead of `team`/`owner`.
The test changes are consistent and complete for the code that was modified.

---

## Observations

### Issues That Should Be Fixed Before Merging

#### 1. `sli.yaml` documents unsupported metric variants

The SLI template shows Options B (`bad/total`) and C (`raw/precomputed`) as valid
commented-out alternatives. The actual codebase does **not** support either of these.

`RatioMetric.from_dict()` in `openslo_types.py` only parses `good` and `total`:

```python
good=RatioMetricPart.from_dict(data.get("good", {})),
total=RatioMetricPart.from_dict(data.get("total", {})),
```

If someone follows Option B and puts `bad` instead of `good`, the parser silently creates
an empty `RatioMetricPart` for `good` with no query. The dashboard generator would then
produce panels with blank PromQL expressions — broken dashboards with no error message.

**Recommendation:** Mark these with `[NOT IMPLEMENTED]` like the alert policies in `slo.yaml`,
or remove them until the parser supports them.

#### 2. `black` formatter references in `CLAUDE.md` are phantom

The updated CLAUDE.md adds:

```
# Check code formatting with black directly
black --check --diff src/yahoo/sre tests

# Auto-format code with black directly
black src/yahoo/sre tests
```

And states: "Black code formatter configuration is defined in `tox.ini`."

There is no `black` configuration anywhere in this repo — no `[tool.black]` in
`pyproject.toml`, no `black` tox env, no `black` in any dependency list. Running
these commands would fail with "command not found" in a fresh environment. This will
confuse anyone following the dev guide.

#### 3. `README.md` is now stale

The README was not updated in this PR. After merge it will:

- Reference `example/openslo.yaml` which no longer exists
- Document the old required defaults (`team`, `owner`) which the generator now rejects
- Show example commands pointing at deleted paths
- Not mention the new `templates/` directory at all

#### 4. `businessImpact` labeled `[REQUIRED]` but not validated

The `openslo.yaml` template marks `businessImpact` as `[REQUIRED]` in the SLO section.
The generator code does:

```python
slo_annotations["business-impact"] = spec.get("businessImpact", "")
```

If someone omits `businessImpact`, they get a silent empty string annotation — no
error, no warning. Either the code should validate it (raise `ValueError` when blank)
or the template should label it `[OPTIONAL]`.

GitHub Copilot flagged this same issue and suggested adding validation. It was marked
resolved in the review thread but the code was not changed.

### Issues Worth Noting (Lower Priority)

#### 5. `alertChannel` and `escalationPolicy` silently removed

The old generator always wrote `alert-channel` and `escalation-policy` as SLO annotations
(even if empty). These annotations are now completely gone from both the code and the
templates. If any downstream system was reading them, this is a silent breaking change.

The PR description doesn't mention this removal explicitly.

#### 6. SLIs can now have completely empty annotations

Previously every SLI got `annotations: {"owner": "..."}`. Now SLI annotations are only
populated if `product_version` is set. Most generated SLIs will have
`annotations: {}` — an empty dict in the YAML output. Not necessarily wrong, but a
behavioral change for anything parsing the generated files.

#### 7. `metricSourceRef` vs `dataSourceRef` mismatch in templates

The `sli.yaml` template uses `metricSourceRef` in comments:

```yaml
# metricSourceRef: <datasource-name>
```

The actual parser reads `dataSourceRef`:

```python
data_source_ref=data.get("dataSourceRef", ""),
```

Someone following the template's commented guidance would use the wrong key name.
Ryan acknowledged this in the Copilot review thread as "Will be resolved in a future PR."

#### 8. `slo.yaml` documents `targetPercent` which is not implemented

The SLO template shows `targetPercent: 99.9` as an alternative to `target: 0.999`.
`SLOObjective.from_dict()` only reads `target` — the `targetPercent` value would be
silently ignored. Less dangerous since it's shown as an either/or comment, but could
still mislead someone.

#### 9. CI failure modes knowledge was deleted

The original `CLAUDE.md` had a valuable "Common failure modes" section:

- bandit B101 — no `assert` in `src/`
- bandit B113 — all `requests` calls need `timeout=`
- pylint W1514 — all `open()` calls need `encoding="utf-8"`
- mypy dict narrowing — `dict[str, Any]` annotations required
- packaging — `namespace_packages` missing from `setup.cfg` fails sdist

This institutional knowledge is gone with no replacement. These are the kind of things
that save a developer 30 minutes of debugging when CI goes red.

#### 10. Python version contradiction remains

`CLAUDE.md` now says "Supported Versions: Python 3.12+" and `setup.cfg` says
`python_requires = >= 3.12`. But `tox.ini` and `screwdriver.yaml` both run `py311`.
If 3.11 isn't supported, the CI config is wrong. If it is, the docs and `setup.cfg`
are wrong. This predates the PR but the CLAUDE.md rewrite reinforced the contradiction
without resolving it.

#### 11. `templates/grafana/` in CLAUDE.md layout but doesn't exist

The updated architecture tree in CLAUDE.md lists:

```
templates/
  grafana/
  openslo/
```

The `templates/grafana/` directory is not created in this PR and doesn't exist on the
branch. Presumably planned for later, but documenting non-existent paths is misleading.

### What Looks Good

- The `timeWindows` → `timeWindow` fix in the generator output is correct and overdue —
  the parser already preferred `timeWindow` and warned on the deprecated key
- The `corp_*` field expansion is clean — required fields validated up front, labels
  populated consistently in both SLI and SLO output
- `product_version` as a conditional annotation (only written when set) is a good pattern
  that avoids cluttering output with empty strings
- The test updates are thorough — every fixture, assertion, and edge case was updated
  to use the new field names
- The `openslo.yaml` manifest template is excellent documentation: every field annotated,
  required/optional clearly marked, good inline examples with per-indicator overrides
- The `slo.yaml` template correctly marks AlertPolicies as `[NOT IMPLEMENTED]` — the
  SLI template should follow this same pattern for unsupported metric variants
