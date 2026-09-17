# Supply Chain Security Enforcement — Platform Architecture

**Version:** 1.0 | **Date:** September 2026 | **Status:** Part of Golden Path

---

## The Problem: Teams Doing Their Own Thing

In an enterprise with 80+ teams, without platform enforcement you get:
- Teams building containers without vulnerability scanning
- Teams pushing unsigned images to production
- Teams skipping SBOM generation
- Teams using deprecated/vulnerable base images
- Teams with hardcoded credentials in Dockerfiles
- Each team maintaining its own release process

**Result:** No supply chain visibility. One compromised dependency → lateral blast.

---

## The Solution: Enforce at Platform Layer, Not Team Layer

### Architecture: Three Enforcement Layers

```
┌─────────────────────────────────────────────────────────────┐
│  LAYER 1: GOLDEN PATH (Pull — make it easy)                │
│  ─────────────────────────────────────────────              │
│  platform-workflows/container-build.yml                     │
│  ├── Trivy scan (HIGH/CRITICAL block)                       │
│  ├── SLSA provenance (automatic)                            │
│  ├── SBOM generation (automatic)                            │
│  ├── Cosign signing (keyless, automatic)                    │
│  └── SARIF upload to GitHub Security tab                    │
│                                                             │
│  Teams get ALL of this with zero config:                    │
│  container-build:                                           │
│    uses: platform-workflows/.../container-build.yml@main    │
│    with:                                                    │
│      image-name: my-service                                 │
│      gcp-project-id: ${{ vars.GCP_PROJECT_ID }}             │
│      push: true                                             │
│                                                             │
│  Result: 4 lines → full supply chain security               │
└─────────────────────────────────────────────────────────────┘
                        │
                        ▼
┌─────────────────────────────────────────────────────────────┐
│  LAYER 2: ORG RULESETS (Push — make it required)           │
│  ─────────────────────────────────────────────              │
│  GitHub Org-level Repository Rulesets:                       │
│  ├── Required status checks: container-build must pass      │
│  ├── Required workflows: container-build.yml from           │
│  │   platform-workflows (GHEC feature)                      │
│  ├── Branch protection: no direct push to main              │
│  └── Code scanning: Trivy SARIF must upload                 │
│                                                             │
│  Effect: Even if a team writes their own Dockerfile,        │
│  they CANNOT merge without the platform container-build     │
│  workflow passing (which includes scan + sign + SBOM)       │
└─────────────────────────────────────────────────────────────┘
                        │
                        ▼
┌─────────────────────────────────────────────────────────────┐
│  LAYER 3: BINARY AUTHORIZATION (Gate — block at runtime)   │
│  ─────────────────────────────────────────────              │
│  GCP Binary Authorization Policy:                           │
│  ├── Attestor: cosign signature from platform CI            │
│  ├── Policy: Only deploy images signed by platform SA       │
│  ├── Cloud Run: --binary-authorization=default              │
│  ├── GKE: BinAuthz admission webhook                       │
│  └── Break-glass: emergency override with audit log         │
│                                                             │
│  Effect: Even if someone builds locally and pushes to       │
│  Artifact Registry manually, Cloud Run / GKE REJECTS       │
│  the image because it lacks the platform's cosign           │
│  attestation. This is the hard gate.                        │
└─────────────────────────────────────────────────────────────┘
```

### How Each Scenario Is Handled

| Scenario | Layer 1 (Golden Path) | Layer 2 (Rulesets) | Layer 3 (BinAuth) |
|---|---|---|---|
| Team uses platform workflow | All security automatic | Passes required checks | Signed, deploys fine |
| Team writes own CI, no scan | N/A | **Blocked** — missing required status | N/A |
| Team pushes image manually | N/A | N/A (no PR) | **Blocked** — unsigned image |
| Team skips cosign in own CI | N/A | **Blocked** — signature check fails | **Blocked** — no attestation |
| Compromised dependency | Trivy blocks the build | Required check fails | Image never signed |
| Emergency hotfix needed | Break-glass via env var | Admin bypass | Break-glass audit log |

### Implementation Checklist

**Phase 1: Golden Path (Day 5 — NOW)**
- [x] Container build reusable workflow with Trivy + cosign + SBOM + provenance
- [x] OIDC/WIF setup (keyless — no stored credentials)
- [x] GitHub Environments (dev/staging/prod)
- [x] Container scan results in GitHub Security tab (SARIF)

**Phase 2: Org Rulesets (Day 18 — Governance)**
- [ ] Create org-level repository ruleset requiring `container-build` status
- [ ] Required workflow: `container-build.yml` from platform-workflows
- [ ] Code scanning alert threshold: block merge on HIGH/CRITICAL
- [ ] Branch protection: require PR, 1+ reviewer, signed commits

**Phase 3: Binary Authorization (Day 7-8 — IaC)**
- [ ] Create Binary Authorization policy via Terraform
- [ ] Create attestor linked to cosign public key
- [ ] Enable BinAuth on Cloud Run services via Terraform module
- [ ] Create break-glass procedure + audit alerting

---

## Agent Integration

The platform agent enforces these automatically:

```
Developer: "Deploy my-service to production"

Agent:      Checking supply chain security...
            
            ✓ Container image signed (cosign, Sigstore/Fulcio)
            ✓ SLSA provenance attached (level 3)
            ✓ SBOM generated (CycloneDX, 142 packages)
            ✓ Vulnerability scan: 0 HIGH, 0 CRITICAL
            ✓ Binary Authorization: attestation valid
            
            Deploying to production...
```

```
Developer: "Deploy this image I built locally"

Agent:      I can't deploy that image. It wasn't built through the 
            platform pipeline, so it's missing:
            
            ✗ No cosign signature (required by Binary Authorization)
            ✗ No SLSA provenance attestation
            ✗ No vulnerability scan results
            
            The platform enforces supply chain security for all 
            production deployments. Use the platform CI workflow 
            to build and sign the image automatically.
            
            Want me to set up the CI workflow for your repo?
```

---

## Industry Grounding (2026)

- **SLSA v1.1** is now the standard for supply chain integrity (Linux Foundation, Feb 2026)
- **Cosign/Sigstore** keyless signing is production-ready and used by 78% of Fortune 500 companies using containers (Chainguard report, Q2 2026)
- **GCP Binary Authorization** supports cosign attestations natively since Feb 2026
- **GitHub Required Workflows** (GHEC) enforce org-wide CI policies without team opt-in
- **EU Cyber Resilience Act (2026)** mandates SBOM for all software products — enterprise IDPs must generate SBOMs by default
- **NIST SSDF v1.2** (Aug 2026) requires automated supply chain verification for federal suppliers — applicable to Ford's government contracts
