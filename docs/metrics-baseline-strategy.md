# Tekton-to-GHA Migration: Metrics Strategy & Business Case

> Before vs After measurement framework for Ford platform modernization

---

## PART 1: PULLING EXISTING METRICS FROM TEKTON

### 1A. Tekton Native Prometheus Metrics

Tekton exposes metrics on port 9090 via the pipelines controller.

**PipelineRun metrics:**

| Metric | Type |
|--------|------|
| `tekton_pipelines_controller_pipelinerun_duration_seconds_[bucket,sum,count]` | Histogram |
| `tekton_pipelines_controller_pipelinerun_total` | Counter (labels: status) |
| `tekton_pipelines_controller_running_pipelineruns` | Gauge |
| `tekton_pipelines_controller_running_pipelineruns_waiting_on_pipeline_resolution` | Gauge |
| `tekton_pipelines_controller_running_pipelineruns_waiting_on_task_resolution` | Gauge |

**TaskRun metrics:**

| Metric | Type |
|--------|------|
| `tekton_pipelines_controller_taskrun_duration_seconds_[bucket,sum,count]` | Histogram |
| `tekton_pipelines_controller_taskrun_total` | Counter (labels: status) |
| `tekton_pipelines_controller_running_taskruns` | Gauge |
| `tekton_pipelines_controller_running_taskruns_throttled_by_quota` | Gauge |
| `tekton_pipelines_controller_running_taskruns_throttled_by_node` | Gauge |
| `tekton_pipelines_controller_taskruns_pod_latency_milliseconds` | Histogram |

Configuration: `config-observability` ConfigMap in `tekton-pipelines` namespace.

### 1B. Essential PromQL Queries to Run NOW

```promql
-- Average pipeline duration (last 7 days)
rate(tekton_pipelines_controller_pipelinerun_duration_seconds_sum[7d])
  / rate(tekton_pipelines_controller_pipelinerun_duration_seconds_count[7d])

-- Pipeline success rate (percentage)
sum(tekton_pipelines_controller_pipelinerun_total{status="success"})
  / sum(tekton_pipelines_controller_pipelinerun_total) * 100

-- Pipeline failure rate over time
rate(tekton_pipelines_controller_pipelinerun_total{status="failed"}[1h])

-- Duration percentiles (p50, p90, p99)
histogram_quantile(0.50, rate(tekton_pipelines_controller_pipelinerun_duration_seconds_bucket[7d]))
histogram_quantile(0.90, rate(tekton_pipelines_controller_pipelinerun_duration_seconds_bucket[7d]))
histogram_quantile(0.99, rate(tekton_pipelines_controller_pipelinerun_duration_seconds_bucket[7d]))

-- Pod scheduling latency / queue time (p95)
histogram_quantile(0.95, rate(tekton_pipelines_controller_taskruns_pod_latency_milliseconds_bucket[1h]))

-- Throughput (pipeline runs per hour)
increase(tekton_pipelines_controller_pipelinerun_total[1h])

-- Throttled runs (capacity bottleneck indicator)
tekton_pipelines_controller_running_taskruns_throttled_by_quota
tekton_pipelines_controller_running_taskruns_throttled_by_node

-- Per-pipeline duration breakdown
sum by (pipeline)(rate(tekton_pipelines_controller_pipelinerun_duration_seconds_sum[7d]))

-- Controller backpressure
kn_workqueue_depth
```

### 1C. tkn CLI Commands for Historical Data

```bash
# List all PipelineRuns (JSON for extraction)
tkn pipelinerun list -A -o json --limit 0

# Extract build times
tkn pipelinerun list -o json | jq '.items[] | {
  name: .metadata.name,
  pipeline: .metadata.labels["tekton.dev/pipeline"],
  status: .status.conditions[0].reason,
  startTime: .status.startTime,
  completionTime: .status.completionTime
}'

# Check if Tekton Results is installed (long-term history)
kubectl get pods -n tekton-pipelines | grep results
```

### 1D. DORA Metrics from Tekton

Tekton has **no built-in DORA metrics**. Must be calculated:

| DORA Metric | How to Calculate from Tekton | Limitation |
|-------------|------------------------------|------------|
| Deployment Frequency | `increase(pipelinerun_total{status="success"}[24h])` | Only counts pipeline runs, not actual deployments |
| Lead Time for Changes | Correlate git commit timestamp with PipelineRun completion | Requires joining with Git API |
| Change Failure Rate | Failed/total pipeline runs | DORA = deployments causing incidents, not pipeline failures |
| MTTR | Time between failed and next successful deploy run | Requires external incident tracker for true MTTR |

---

## PART 2: GITHUB ACTIONS METRICS

### 2A. GHA Native APIs

| API | Endpoint | Key Data |
|-----|----------|----------|
| Workflow Runs | `GET /repos/{owner}/{repo}/actions/runs` | status, conclusion, timing, commit SHA |
| Run Timing | `GET /repos/{owner}/{repo}/actions/runs/{id}/timing` | billable ms by OS, total duration |
| Job Detail | `GET /repos/{owner}/{repo}/actions/runs/{id}/jobs` | per-job created_at, started_at, completed_at |
| Billing | `GET /organizations/{org}/settings/billing/usage` | total org usage |

**GHEC Native Dashboard:** Organization > Insights > Actions
- Usage + Performance metrics, filterable, up to 100-day range, CSV export

### 2B. DORA from GHA

| DORA Metric | GHA API Fields |
|-------------|----------------|
| Deployment Frequency | Count runs where `event=deployment`, grouped by day |
| Lead Time | `head_commit.timestamp` to deployment workflow `updated_at` |
| Change Failure Rate | `conclusion=failure` on deploy workflows / total |
| MTTR | Failed deploy `updated_at` to next successful deploy `updated_at` |
| Queue Time | Job `started_at` minus `created_at` |

### 2C. Third-Party Tools

| Tool | Best For | DORA | Cost |
|------|----------|------|------|
| Datadog CI Visibility | Full observability | Via GHA action | Enterprise pricing |
| LinearB | Engineering metrics | All 4 | $29/mo/contributor |
| Sleuth | Deploy tracking | All 4 | Free tier available |
| Faros AI | Multi-tool aggregation | All 4 + rework | 100+ connectors |
| DX (getdx.com) | Developer experience | DORA + SPACE | Enterprise |

---

## PART 3: BEFORE vs AFTER COMPARISON

### 3A. Metrics Comparison Table

| Category | Metric | Current (Tekton) | Target (GHA) | How to Measure |
|----------|--------|-------------------|---------------|----------------|
| **Pipeline Execution** | Build time (p50) | ___min | ___min | PromQL vs GHA timing API |
| | Build time (p95) | ___min | ___min | Same |
| | Test time (p50) | ___min | ___min | TaskRun duration vs GHA job |
| | Deploy time (p50) | ___min | ___min | Deploy pipeline vs GHA workflow |
| | Queue/wait time (p95) | ___min | ___min | pod_latency vs job started-created |
| **Developer Experience** | Onboarding time | ___weeks | ___days | Survey: time to first commit |
| | Pipeline authoring | ___hours | ___hours | Survey: time to create CI/CD |
| | Pipeline YAML LOC | ~300 lines | ~80 lines | Direct count |
| | Files per pipeline | 8-12 files | 1 file | Direct count |
| | K8s expertise needed | Yes | No | Binary |
| | Local testing | No | Yes (`act`) | Binary |
| **Maintenance** | Platform admin FTEs | ___ | ___ | Hours tracking |
| | CRDs to manage | 22-25 | 0 | Direct count |
| | ConfigMaps | 10+ | 0 | Direct count |
| | Upgrade frequency | ___/year | 0 (SaaS) | Count |
| **Infrastructure Cost** | Controller compute | $___/mo | $0 (SaaS) | K8s resource costs |
| | Pipeline compute | $___/mo | $___/mo | GKE vs GHA/ARC |
| | Total CI/CD cost | $___/mo | $___/mo | Sum |
| **DORA Metrics** | Deploy Frequency | ___/day | ___/day | PipelineRun vs GHA deploys |
| | Lead Time | ___hours | ___hours | Commit-to-deploy |
| | Change Failure Rate | ___% | ___% | Failed/total |
| | MTTR | ___hours | ___hours | Incident + deploy data |
| **Security** | Scan integration effort | ___hours | ___min | Time to add SAST/SCA |
| | OIDC/WIF | Manual | Native | Binary |
| **Debugging** | Time to find failure | ___min | ___min | Survey |
| | Log access | kubectl | Browser UI | Qualitative |
| **Scale** | Catalog size | ~200 tasks | 20,000+ actions | 100x ratio |

### 3B. DORA Performance Benchmarks

| Metric | Elite | High | Medium | Low |
|--------|-------|------|--------|-----|
| Deploy Frequency | Many/day | Daily-weekly | Weekly-monthly | Monthly+ |
| Lead Time | < 1 day | 1 day - 1 week | 1 week - 1 month | > 1 month |
| Change Failure Rate | 5% | 10% | 15% | 64% |
| Recovery Time | < 1 hour | < 1 day | 1 day - 1 week | 1 month+ |

High performers deploy **200x more frequently** with **2,555x faster lead times**.

### 3C. ROI Calculation Template

```
=== ANNUAL SAVINGS ===

1. Developer Time Savings
   Current: ___ hrs/pipeline x ___ pipelines/yr x $___/hr = $___/yr
   GHA:     ___ hrs/pipeline x ___ pipelines/yr x $___/hr = $___/yr
   Savings: $___/yr

2. Platform Admin Reduction
   Current Tekton admin: ___ FTEs x $___/yr = $___/yr
   GHA admin:            ___ FTEs x $___/yr = $___/yr
   Savings: $___/yr

3. Infrastructure Cost Delta
   Current (GKE for Tekton): $___/mo x 12 = $___/yr
   GHA (included + ARC):     $___/mo x 12 = $___/yr
   Savings: $___/yr

4. Developer Productivity (DORA-based)
   Elite teams spend 22% less on unplanned work
   Hours recovered: ___ devs x ___ hrs/wk x 52 x $___/hr = $___/yr

=== MIGRATION COSTS (ONE-TIME) ===

5. Migration labor: ___ pipelines x ___ hrs x $___/hr = $___
6. Training: ___ devs x ___ hrs x $___/hr = $___
7. Parallel running: ___ months x dual cost = $___

=== ROI ===
Payback: Migration cost / Monthly savings = ___ months
3-year ROI: ((Annual savings x 3) - Migration cost) / Migration cost x 100 = ___%
```

**Reference: Forrester TEI 2025 = 376% ROI, < 6-month payback, $67.9M NPV**

---

## PART 4: PRESENTING TO LEADERSHIP

### VP Engineering (Developer Productivity)
- Lead with DORA: "High performers deploy 200x more frequently"
- Show: 3-4x YAML reduction, 1 file vs 12, 20K marketplace actions
- **GM lead**: "Critical builds from 4-6 hours to 27 minutes"

### CTO (Technical Strategy)
- Platform maturity: "73% of platform-mature orgs drive AI success" (Puppet 2026)
- Technical debt: eliminate 22-25 CRDs, 10+ ConfigMaps
- AI readiness: GHA + Copilot = 56% faster task completion
- **Mercedes lead**: "55K devs, 90% source code on GitHub"

### CFO (Cost & ROI)
- Forrester: "376% ROI, < 6-month payback"
- TCO waterfall: Tekton infra + admin vs GHA included
- "90% Fortune 100 on GitHub — not experimental"
- **GM lead**: "150K repos migrated, zero production impact"

### Recommended Visualizations
1. **DORA Quadrant**: Ford's current position vs target (Elite/High/Medium/Low)
2. **TCO Waterfall**: Stacked bar — license, compute, labor, maintenance
3. **Time-to-Value**: Line chart with break-even point
4. **Complexity Reduction**: 22 CRDs → 0, 12 files → 1, 300 LOC → 80

---

## KEY STATISTICS FOR THE PITCH

| Statistic | Source |
|-----------|--------|
| 376% ROI over 3 years | Forrester TEI 2025 |
| < 6-month payback | Forrester TEI 2025 |
| 90% Fortune 100 use GitHub | GitHub 2025 |
| GM: 4-6 hours → 27 minutes | GM case study |
| Mercedes: 55K devs on GitHub | Mercedes case study |
| 200x more frequent deploys (elite) | DORA 2016 |
| 73% platform-mature → AI success | Puppet 2026 |
| 20,000+ actions vs ~200 Tekton tasks | Direct count |
| 22-25 CRDs eliminated | Direct count |
| 56% faster with Copilot | McKinsey/GitHub |
| Ford: 22K devs, 9K active GitHub users | Internal |

---

## IMMEDIATE ACTION PLAN

**Week 1 — Baseline Tekton Metrics:**
1. Verify Prometheus scraping: `kubectl port-forward -n tekton-pipelines svc/tekton-pipelines-controller 9090:9090`
2. Run all PromQL queries from Section 1B
3. Check for Tekton Results: `kubectl get pods -n tekton-pipelines | grep results`
4. Export 90 days: `tkn pipelinerun list -A -o json --limit 0 > tekton_baseline.json`
5. Count CRDs: `kubectl get crds | grep tekton | wc -l`

**Week 2 — Build Comparison:**
1. Recreate 2-3 Tekton pipelines as GHA workflows (measure LOC and time)
2. Set up GHA API queries for timing
3. Fill comparison table with real numbers

**Week 3 — Build the Deck:**
1. DORA quadrant with Ford's current position
2. TCO waterfall Tekton vs GHA
3. 3 pitch variants: VP (productivity), CTO (technical), CFO (cost)
