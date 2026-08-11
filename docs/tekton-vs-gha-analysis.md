# Tekton vs GitHub Actions: Enterprise CI/CD Analysis (2026)

> Grounded research for Ford platform modernization pitch

---

## 1. Market Position (2025-2026)

| Rank | Tool | Market Share | Trend |
|------|------|-------------|-------|
| 1 | **GitHub Actions** | ~50-56% | Growing |
| 2 | Jenkins | ~26-33% | Declining |
| 3 | GitLab CI/CD | ~22-32% | Stable |
| 4 | Azure DevOps | ~10-15% | Stable |
| 5 | **Tekton** | ~3-5% | Niche (K8s ecosystem) |

Sources: JetBrains Developer Ecosystem 2024, Stack Overflow Survey 2024

### GitHub Actions Scale
- **180M+ developers** on GitHub, **4M+ organizations**
- **90% of Fortune 100** are GitHub customers
- **10.54B GitHub Actions minutes** consumed in 2024 (~30% YoY growth)
- **20,000+ marketplace actions** available
- Source: https://github.blog/news-insights/octoverse/octoverse-2024/

---

## 2. Tekton Project Health — NOT Dying

**Verdict: Actively maintained, gaining institutional backing**

| Metric | Data |
|--------|------|
| CNCF Status | **Incubating** (accepted March 2026) |
| Latest Release | v1.15.0 LTS (July 31, 2026) |
| Release Cadence | Multiple per month across 5 supported branches |
| GitHub Stars | 9,000+ (tektoncd/pipeline) |
| Contributors | 600+, 5,950 commits, 5,000+ PRs |
| Last Commit | August 10, 2026 |
| Deprecation Signals | **None** — CNCF incubation is the opposite signal |

**Key backers:** Google (Cloud Build uses Tekton), Red Hat/IBM (OpenShift Pipelines = Tekton), **Ford is named as adopter in CNCF incubation announcement**

Source: https://www.cncf.io/blog/2026/03/24/tekton-becomes-a-cncf-incubating-project/

---

## 3. Automotive Industry CI/CD Landscape

| Company | CI/CD Stack | Scale | Impact |
|---------|-------------|-------|--------|
| **Ford** | Tekton + GHEC EMU + Atlantis | 80+ teams | Current state — this POC |
| **General Motors** | GitHub Enterprise + GitHub Actions | 19,000 devs, 150K repos | **Build times: 4-6 hours → 27 minutes** |
| **Mercedes-Benz** | GitHub Enterprise + GitHub Actions | 55,000 devs, 115K repos | 90% source code on GitHub |

GM case study: https://github.com/customer-stories/general-motors
Mercedes case study: https://github.com/customer-stories/mercedes-benz

---

## 4. Big Tech CI/CD Choices

| Company | CI/CD | Notes |
|---------|-------|-------|
| Google | Custom (TAP, Forge, Blaze/Bazel) | 50K+ changes/day, 4B+ test cases |
| Microsoft | Migrating internal to GitHub Actions | They own GitHub — strategic direction |
| Amazon | Custom (Brazil, Apollo, Pipelines) | Fully proprietary |
| Meta | Custom (Buck2, Sapling, Phabricator) | Proprietary |
| Netflix | Spinnaker (they created it) | CNCF project |
| Spotify | GitHub Enterprise + custom CI | Created Backstage (CNCF) |

**Takeaway:** Hyperscalers all build custom. Among enterprises **buying** CI/CD, the trend is consolidation onto GitHub Actions or GitLab CI.

---

## 5. Migration: Tekton → GHA

### No Public Migration Path Exists
No official guide from GitHub, Google, CNCF, or community. This is a **gap Ford can fill internally** — and a POC differentiator.

### What's Easy to Migrate

| Aspect | Why |
|--------|-----|
| Simple build/test/deploy steps | Direct mapping to GHA `run:` steps |
| Environment variables | Both support env vars at workflow/job/step level |
| Secrets injection | K8s Secrets → GHA Secrets |
| Conditional logic | Tekton `when` → GHA `if:` |
| Matrix builds | Both support parameterized execution |

### What's Hard to Migrate

| Aspect | Why |
|--------|-----|
| Pipeline/Task CRD structure | K8s CRDs have no 1:1 GHA equivalent |
| Container-per-step model | Tekton = explicit container per step; GHA = shared runner filesystem |
| Tekton Catalog tasks | K8s-specific ops have no Marketplace equivalent |
| Workspace/PVC artifact passing | Tekton PVCs → GHA artifact upload/download actions |
| Tekton Triggers (webhooks) | Different event models |

### Mapping Cheat Sheet

| Tekton Concept | GHA Equivalent |
|----------------|----------------|
| `Task` | Composite Action or Reusable Workflow |
| `Pipeline` | Workflow (`.github/workflows/*.yml`) |
| `PipelineRun` | Workflow Run |
| `TaskRun` | Job |
| `Step` (container) | Step (`run:` or `uses:`) |
| `Workspace` (PVC) | `actions/upload-artifact` + `actions/download-artifact` |
| `Param` | `inputs:` (workflow_call) or `env:` |
| `Result` | Job outputs (`outputs:`) |
| `When Expression` | `if:` condition |
| `Tekton Catalog` | GitHub Marketplace |
| `Tekton Triggers` | Native webhook events (`on:`) |
| `Tekton Chains` | GitHub Attestations (SLSA) |
| `Tekton Dashboard` | Actions tab + third-party dashboards |

---

## 6. Pros & Cons Comparison

### Tekton

| Category | ✅ Pro | ❌ Con |
|----------|-------|-------|
| Architecture | K8s-native CRDs, runs anywhere K8s runs | Requires K8s cluster, steep learning curve |
| Lock-in | CNCF incubating, fully vendor-neutral | Smaller ecosystem, fewer pre-built tasks |
| Flexibility | Complete control over execution, networking, storage | Complexity tax: YAML-heavy, high operational overhead |
| Security | Tekton Chains = SLSA L3, any KMS backend | Manual setup and operational management |
| Cost | Zero licensing, pure infrastructure | K8s operational cost (people + infra) |
| Scalability | Scales with K8s, no per-minute billing | You manage scaling yourself |

### GitHub Actions

| Category | ✅ Pro | ❌ Con |
|----------|-------|-------|
| Ease of Use | Lowest barrier, simple YAML, 20K+ marketplace actions | Vendor lock-in: workflow syntax is GitHub-proprietary |
| Ecosystem | 180M+ devs, largest marketplace, native GitHub integration | Marketplace security: only 4% of orgs pin action hashes |
| Enterprise | 90% Fortune 100, GHEC = 50K free min/month | Cost at scale if using hosted runners |
| Security | SLSA L3 with reusable workflows, Sigstore, 3-line setup | OIDC tied to GitHub identity |
| Scalability | ARC provides K8s-native autoscaling self-hosted runners | ARC adds operational complexity |
| Migration | Jenkins→GHA is well-documented | Tekton→GHA has zero tooling |

### Security Supply Chain Comparison

| Feature | Tekton | GitHub Actions |
|---------|--------|---------------|
| SLSA Level | L3 (with Tekton Chains) | L3 (with reusable workflows) |
| Signing | Sigstore/Cosign, x509, any KMS | Sigstore (Fulcio CA + Rekor) |
| Setup Complexity | **High** | **Low** (3-line workflow) |
| Marketplace Risk | N/A (no marketplace) | **High** — 80% of orgs use unpinned actions |

Source: https://www.datadoghq.com/state-of-devsecops/

### Cost at Scale

| Scenario | Tekton | GitHub Actions |
|----------|--------|---------------|
| Licensing | Free | Free tier: 2K min/mo; Enterprise: 50K min/mo |
| Hosted runners | N/A (self-hosted only) | $0.008/min Linux = $0.48/hr |
| Self-hosted (ARC on GKE) | e2-standard-4 ~$0.134/hr | Same infra cost, zero per-minute |
| **At scale (>50K min/month)** | **Pure infra cost** | **ARC eliminates per-minute — comparable** |

---

## 7. Key Findings for Ford Pitch

### The Story Is NOT "Tekton is bad, GHA is good"

1. **Tekton is healthy** — CNCF incubating, active releases, Ford is a named adopter
2. **The pitch is about developer experience + consolidation**, not replacing a failing tool
3. **GM is the killer case study**: same industry, 19K devs, 4-6hr → 27min builds
4. **Mercedes-Benz**: 55K devs, 90% code on GitHub — shows automotive scale

### Recommended Positioning

> "We're not replacing Tekton because it's broken. We're consolidating onto GitHub Actions because:
> 1. We already pay for GHEC — CI/CD is included
> 2. 80+ teams get a consistent platform vs managing K8s CRDs
> 3. Developer onboarding drops from weeks to hours
> 4. GM and Mercedes already proved this works at automotive scale
> 5. Marketplace gives us 20K+ pre-built integrations vs custom Tekton Tasks
> 6. ARC gives us the same K8s self-hosted runner model if we need it"

### Risk to Acknowledge

- **Ford is named as a Tekton adopter by CNCF** — some internal stakeholders may resist
- **Marketplace security** — must enforce action pinning (SHA, not tags)
- **No migration tooling exists** — Ford would be a pioneer here
- **Vendor lock-in** — GHA workflow YAML is GitHub-proprietary (mitigated by ARC portability)

---

## Sources

| Source | URL |
|--------|-----|
| GitHub Octoverse 2024 | https://github.blog/news-insights/octoverse/octoverse-2024/ |
| Tekton CNCF Incubation | https://www.cncf.io/blog/2026/03/24/tekton-becomes-a-cncf-incubating-project/ |
| Tekton Releases | https://github.com/tektoncd/pipeline/releases |
| GM Case Study | https://github.com/customer-stories/general-motors |
| Mercedes-Benz Case Study | https://github.com/customer-stories/mercedes-benz |
| Actions Runner Controller | https://github.com/actions/actions-runner-controller |
| Datadog DevSecOps Report | https://www.datadoghq.com/state-of-devsecops/ |
| GitHub Attestations | https://docs.github.com/en/actions/security-for-github-actions/using-artifact-attestations |
| Tekton Chains | https://tekton.dev/docs/chains/ |
| SLSA Levels | https://slsa.dev/spec/v1.0/levels |
| DORA Research | https://dora.dev/research/ |
