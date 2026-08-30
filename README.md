# platform-workflows

Reusable GitHub Actions workflows and composite actions for the platform.

## Structure

```
.github/
  workflows/
    python-ci.yml    # Python: lint + test + optional security scan
    node-ci.yml      # Node.js: lint + test + build + optional Playwright E2E
    java-ci.yml      # Java: build + test + lint + optional OWASP dep check
  actions/           # Composite actions (coming soon)
```

## Usage

### Python CI

```yaml
jobs:
  ci:
    uses: terzo241-platform/platform-workflows/.github/workflows/python-ci.yml@main
    with:
      python-version: "3.12"
      lint-tool: ruff           # ruff, flake8, or py_compile
      enable-security-scan: true
```

### Node.js CI

```yaml
jobs:
  ci:
    uses: terzo241-platform/platform-workflows/.github/workflows/node-ci.yml@main
    with:
      node-version: "20"
      package-manager: npm      # npm, pnpm, or yarn
      enable-playwright: false
```

### Java CI

```yaml
jobs:
  ci:
    uses: terzo241-platform/platform-workflows/.github/workflows/java-ci.yml@main
    with:
      java-version: "17"
      java-distribution: temurin  # temurin, corretto, or zulu
      build-tool: maven            # maven or gradle
      enable-security-scan: false
```
