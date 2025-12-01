# Part 2 — CI/CD Pipelines

## Why This Matters

A CI/CD pipeline is your team's quality gatekeeper. It should be impossible to merge code that doesn't compile, breaks tests, or introduces security vulnerabilities.

> "If your pipeline takes 30 minutes, developers will push 3 times a day. If it takes 3 minutes, they'll push 30 times. Speed changes behavior." — Unknown DevOps engineer

The goal: **Every commit is a candidate for production.**

---

## What Goes Wrong Without This

### The "It Works on My Machine" Deploy

```
Developer: "I tested locally, it works"
Pipeline: Doesn't exist
Production: "Object reference not set to an instance of an object"

Postmortem:
- Developer had .NET 8.0.4, production had 8.0.0
- Developer's machine had a config file that wasn't committed
- Developer's database had test data that masked the bug

Cost: 4 hours of downtime, customer escalation
```

### The Friday Night Surprise

```
Friday 5 PM: Code merged without running tests
Friday 6 PM: Security team finds hardcoded API key in commit
Friday 7 PM: Key rotated, dependent services break
Saturday 2 AM: All systems restored

Root cause: No automated scanning in pipeline
```

### The Merge Queue Nightmare

```
Monday 9 AM:
- Developer A merges "Add order filtering"
- Developer B merges "Update payment service"
- Developer C merges "Refactor customer API"
- All PRs passed their own tests

Monday 10 AM:
- Build fails: incompatible changes
- 3 developers now blocked
- 2 hours to untangle

Root cause: No integration testing between merges
```

---

## GitHub Actions Deep Dive

### Understanding the Workflow File

```yaml
# .github/workflows/ci.yml

# Name shown in GitHub UI
name: CI Pipeline

# When to run
on:
  push:
    branches: [main, develop]
  pull_request:
    branches: [main]
  workflow_dispatch:  # Manual trigger

# Environment variables for all jobs
env:
  DOTNET_VERSION: '8.0.x'
  CONFIGURATION: Release

# Permissions (principle of least privilege)
permissions:
  contents: read
  checks: write
  pull-requests: write

jobs:
  # First job: Build and test
  build:
    runs-on: ubuntu-latest

    steps:
      - name: Checkout code
        uses: actions/checkout@v4
        with:
          fetch-depth: 0  # Full history for versioning

      - name: Setup .NET
        uses: actions/setup-dotnet@v4
        with:
          dotnet-version: ${{ env.DOTNET_VERSION }}

      - name: Cache NuGet packages
        uses: actions/cache@v4
        with:
          path: ~/.nuget/packages
          key: ${{ runner.os }}-nuget-${{ hashFiles('**/*.csproj') }}
          restore-keys: |
            ${{ runner.os }}-nuget-

      - name: Restore dependencies
        run: dotnet restore

      - name: Build
        run: dotnet build --no-restore --configuration ${{ env.CONFIGURATION }}

      - name: Test with coverage
        run: |
          dotnet test --no-build --configuration ${{ env.CONFIGURATION }} \
            --collect:"XPlat Code Coverage" \
            --results-directory ./coverage \
            --logger "trx;LogFileName=test-results.trx"

      - name: Upload test results
        uses: actions/upload-artifact@v4
        if: always()  # Upload even if tests fail
        with:
          name: test-results
          path: coverage/
```

### Workflow Syntax Essentials

```yaml
# Job dependencies
jobs:
  build:
    runs-on: ubuntu-latest
    # ...

  test:
    needs: build  # Waits for build to complete
    runs-on: ubuntu-latest
    # ...

  deploy-staging:
    needs: [build, test]  # Waits for both
    runs-on: ubuntu-latest
    # ...

# Matrix builds - test multiple configurations
jobs:
  test:
    strategy:
      matrix:
        os: [ubuntu-latest, windows-latest]
        dotnet: ['7.0.x', '8.0.x']
      fail-fast: false  # Continue other jobs if one fails

    runs-on: ${{ matrix.os }}
    steps:
      - uses: actions/setup-dotnet@v4
        with:
          dotnet-version: ${{ matrix.dotnet }}

# Conditional execution
steps:
  - name: Deploy to production
    if: github.ref == 'refs/heads/main' && github.event_name == 'push'
    run: ./deploy.sh

  - name: Skip on forks
    if: github.repository == 'myorg/myrepo'
    run: echo "Running on main repo"
```

---

## Complete OrderFlow Pipeline

### CI Pipeline

```yaml
# .github/workflows/ci.yml
name: OrderFlow CI

on:
  push:
    branches: [main, develop]
    paths-ignore:
      - '**.md'
      - 'docs/**'
  pull_request:
    branches: [main]

env:
  DOTNET_VERSION: '8.0.x'
  REGISTRY: ghcr.io
  IMAGE_NAME: ${{ github.repository }}

permissions:
  contents: read
  packages: write
  security-events: write
  pull-requests: write
  checks: write

jobs:
  # ============================================
  # BUILD & TEST
  # ============================================
  build:
    name: Build & Test
    runs-on: ubuntu-latest

    services:
      # Spin up dependencies for integration tests
      postgres:
        image: postgres:16
        env:
          POSTGRES_USER: orderflow
          POSTGRES_PASSWORD: orderflow
          POSTGRES_DB: orderflow_test
        ports:
          - 5432:5432
        options: >-
          --health-cmd pg_isready
          --health-interval 10s
          --health-timeout 5s
          --health-retries 5

      redis:
        image: redis:7
        ports:
          - 6379:6379
        options: >-
          --health-cmd "redis-cli ping"
          --health-interval 10s
          --health-timeout 5s
          --health-retries 5

    outputs:
      version: ${{ steps.version.outputs.version }}

    steps:
      - name: Checkout
        uses: actions/checkout@v4
        with:
          fetch-depth: 0

      - name: Setup .NET
        uses: actions/setup-dotnet@v4
        with:
          dotnet-version: ${{ env.DOTNET_VERSION }}

      - name: Calculate version
        id: version
        run: |
          # Use GitVersion or simple approach
          VERSION="1.0.${{ github.run_number }}"
          if [[ "${{ github.ref }}" != "refs/heads/main" ]]; then
            VERSION="${VERSION}-preview"
          fi
          echo "version=${VERSION}" >> $GITHUB_OUTPUT
          echo "Building version: ${VERSION}"

      - name: Cache NuGet
        uses: actions/cache@v4
        with:
          path: ~/.nuget/packages
          key: nuget-${{ runner.os }}-${{ hashFiles('**/*.csproj', '**/Directory.Packages.props') }}
          restore-keys: nuget-${{ runner.os }}-

      - name: Restore
        run: dotnet restore

      - name: Build
        run: |
          dotnet build --no-restore -c Release \
            -p:Version=${{ steps.version.outputs.version }}

      - name: Unit Tests
        run: |
          dotnet test --no-build -c Release \
            --filter "Category!=Integration" \
            --collect:"XPlat Code Coverage" \
            --results-directory ./TestResults \
            --logger "trx;LogFileName=unit-tests.trx" \
            -- DataCollectionRunSettings.DataCollectors.DataCollector.Configuration.Format=opencover

      - name: Integration Tests
        run: |
          dotnet test --no-build -c Release \
            --filter "Category=Integration" \
            --results-directory ./TestResults \
            --logger "trx;LogFileName=integration-tests.trx"
        env:
          ConnectionStrings__Database: "Host=localhost;Database=orderflow_test;Username=orderflow;Password=orderflow"
          ConnectionStrings__Redis: "localhost:6379"

      - name: Upload Test Results
        uses: actions/upload-artifact@v4
        if: always()
        with:
          name: test-results
          path: TestResults/
          retention-days: 7

      - name: Code Coverage Report
        uses: danielpalme/ReportGenerator-GitHub-Action@5
        with:
          reports: TestResults/**/coverage.opencover.xml
          targetdir: CoverageReport
          reporttypes: HtmlInline;Cobertura;MarkdownSummaryGithub

      - name: Upload Coverage Report
        uses: actions/upload-artifact@v4
        with:
          name: coverage-report
          path: CoverageReport/

      - name: Coverage Summary
        uses: irongut/CodeCoverageSummary@v1.3.0
        with:
          filename: CoverageReport/Cobertura.xml
          badge: true
          format: markdown
          output: both
          thresholds: '60 80'

      - name: Add Coverage PR Comment
        uses: marocchino/sticky-pull-request-comment@v2
        if: github.event_name == 'pull_request'
        with:
          path: code-coverage-results.md

      - name: Fail if coverage below threshold
        run: |
          COVERAGE=$(grep -oP 'Line coverage: \K[0-9.]+' code-coverage-results.md || echo "0")
          echo "Line coverage: ${COVERAGE}%"
          if (( $(echo "$COVERAGE < 60" | bc -l) )); then
            echo "::error::Coverage ${COVERAGE}% is below 60% threshold"
            exit 1
          fi

      - name: Publish
        run: |
          dotnet publish src/OrderFlow.Api/OrderFlow.Api.csproj \
            --no-build -c Release \
            -o ./publish \
            -p:Version=${{ steps.version.outputs.version }}

      - name: Upload Build Artifact
        uses: actions/upload-artifact@v4
        with:
          name: orderflow-api
          path: publish/
          retention-days: 7

  # ============================================
  # CODE QUALITY
  # ============================================
  code-quality:
    name: Code Quality
    runs-on: ubuntu-latest

    steps:
      - name: Checkout
        uses: actions/checkout@v4

      - name: Setup .NET
        uses: actions/setup-dotnet@v4
        with:
          dotnet-version: ${{ env.DOTNET_VERSION }}

      - name: Verify Formatting
        run: dotnet format --verify-no-changes --verbosity diagnostic

      - name: Run Analyzers
        run: |
          dotnet build -c Release \
            -warnaserror \
            -p:TreatWarningsAsErrors=true

  # ============================================
  # SECURITY SCANNING
  # ============================================
  security:
    name: Security Scan
    runs-on: ubuntu-latest

    steps:
      - name: Checkout
        uses: actions/checkout@v4

      - name: Setup .NET
        uses: actions/setup-dotnet@v4
        with:
          dotnet-version: ${{ env.DOTNET_VERSION }}

      - name: Restore
        run: dotnet restore

      # Dependency vulnerability scanning
      - name: NuGet Audit
        run: dotnet list package --vulnerable --include-transitive 2>&1 | tee audit.txt

      - name: Check for vulnerabilities
        run: |
          if grep -q "has the following vulnerable packages" audit.txt; then
            echo "::error::Vulnerable packages detected!"
            cat audit.txt
            exit 1
          fi

      # SAST with CodeQL
      - name: Initialize CodeQL
        uses: github/codeql-action/init@v3
        with:
          languages: csharp
          queries: security-and-quality

      - name: Build for CodeQL
        run: dotnet build -c Release

      - name: Perform CodeQL Analysis
        uses: github/codeql-action/analyze@v3
        with:
          category: "/language:csharp"

      # Secret scanning
      - name: Scan for secrets
        uses: trufflesecurity/trufflehog@main
        with:
          path: ./
          base: ${{ github.event.repository.default_branch }}
          head: HEAD

  # ============================================
  # CONTAINER BUILD
  # ============================================
  container:
    name: Build Container
    runs-on: ubuntu-latest
    needs: [build, code-quality, security]
    if: github.event_name == 'push'

    steps:
      - name: Checkout
        uses: actions/checkout@v4

      - name: Set up Docker Buildx
        uses: docker/setup-buildx-action@v3

      - name: Login to Container Registry
        uses: docker/login-action@v3
        with:
          registry: ${{ env.REGISTRY }}
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}

      - name: Extract metadata
        id: meta
        uses: docker/metadata-action@v5
        with:
          images: ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}
          tags: |
            type=ref,event=branch
            type=ref,event=pr
            type=sha,prefix=
            type=raw,value=${{ needs.build.outputs.version }}
            type=raw,value=latest,enable=${{ github.ref == 'refs/heads/main' }}

      - name: Build and push
        uses: docker/build-push-action@v5
        with:
          context: .
          file: ./src/OrderFlow.Api/Dockerfile
          push: true
          tags: ${{ steps.meta.outputs.tags }}
          labels: ${{ steps.meta.outputs.labels }}
          cache-from: type=gha
          cache-to: type=gha,mode=max
          build-args: |
            VERSION=${{ needs.build.outputs.version }}

      # Container security scanning
      - name: Scan container
        uses: aquasecurity/trivy-action@master
        with:
          image-ref: ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}:${{ needs.build.outputs.version }}
          format: 'sarif'
          output: 'trivy-results.sarif'
          severity: 'CRITICAL,HIGH'

      - name: Upload Trivy results
        uses: github/codeql-action/upload-sarif@v3
        with:
          sarif_file: 'trivy-results.sarif'
```

### CD Pipeline (Deployment)

```yaml
# .github/workflows/cd.yml
name: OrderFlow CD

on:
  workflow_run:
    workflows: ["OrderFlow CI"]
    types: [completed]
    branches: [main]
  workflow_dispatch:
    inputs:
      environment:
        description: 'Environment to deploy to'
        required: true
        default: 'staging'
        type: choice
        options:
          - staging
          - production
      version:
        description: 'Version to deploy (leave empty for latest)'
        required: false
        type: string

env:
  REGISTRY: ghcr.io
  IMAGE_NAME: ${{ github.repository }}

jobs:
  # ============================================
  # DEPLOY TO STAGING
  # ============================================
  deploy-staging:
    name: Deploy to Staging
    runs-on: ubuntu-latest
    if: >
      (github.event_name == 'workflow_run' && github.event.workflow_run.conclusion == 'success') ||
      (github.event_name == 'workflow_dispatch' && inputs.environment == 'staging')
    environment:
      name: staging
      url: https://staging.orderflow.example.com

    steps:
      - name: Checkout
        uses: actions/checkout@v4

      - name: Login to Azure
        uses: azure/login@v1
        with:
          creds: ${{ secrets.AZURE_CREDENTIALS }}

      - name: Get version
        id: version
        run: |
          if [ -n "${{ inputs.version }}" ]; then
            echo "version=${{ inputs.version }}" >> $GITHUB_OUTPUT
          else
            # Get latest version from registry
            VERSION=$(az acr repository show-tags \
              --name orderflowacr \
              --repository orderflow-api \
              --orderby time_desc \
              --top 1 \
              --output tsv)
            echo "version=${VERSION}" >> $GITHUB_OUTPUT
          fi

      - name: Deploy to Container Apps
        uses: azure/container-apps-deploy-action@v1
        with:
          resourceGroup: orderflow-staging-rg
          containerAppName: orderflow-api
          imageToDeploy: ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}:${{ steps.version.outputs.version }}

      - name: Wait for deployment
        run: |
          echo "Waiting for deployment to stabilize..."
          sleep 30

      - name: Run smoke tests
        run: |
          STAGING_URL="https://staging.orderflow.example.com"

          # Health check
          STATUS=$(curl -s -o /dev/null -w "%{http_code}" "${STAGING_URL}/health")
          if [ "$STATUS" != "200" ]; then
            echo "::error::Health check failed with status ${STATUS}"
            exit 1
          fi

          # Readiness check
          STATUS=$(curl -s -o /dev/null -w "%{http_code}" "${STAGING_URL}/health/ready")
          if [ "$STATUS" != "200" ]; then
            echo "::error::Readiness check failed with status ${STATUS}"
            exit 1
          fi

          echo "Smoke tests passed!"

      - name: Run E2E tests
        run: |
          # Run Playwright or similar E2E tests against staging
          npm ci
          npx playwright test --project=api
        working-directory: ./tests/e2e
        env:
          BASE_URL: https://staging.orderflow.example.com

      - name: Notify on failure
        if: failure()
        uses: slackapi/slack-github-action@v1
        with:
          payload: |
            {
              "text": "Staging deployment failed for OrderFlow",
              "blocks": [
                {
                  "type": "section",
                  "text": {
                    "type": "mrkdwn",
                    "text": "*Staging Deployment Failed* :x:\n*Version:* ${{ steps.version.outputs.version }}\n*Workflow:* <${{ github.server_url }}/${{ github.repository }}/actions/runs/${{ github.run_id }}|View Run>"
                  }
                }
              ]
            }
        env:
          SLACK_WEBHOOK_URL: ${{ secrets.SLACK_WEBHOOK_URL }}

  # ============================================
  # DEPLOY TO PRODUCTION
  # ============================================
  deploy-production:
    name: Deploy to Production
    runs-on: ubuntu-latest
    needs: deploy-staging
    if: >
      (github.event_name == 'workflow_run' && github.ref == 'refs/heads/main') ||
      (github.event_name == 'workflow_dispatch' && inputs.environment == 'production')
    environment:
      name: production
      url: https://api.orderflow.example.com

    steps:
      - name: Checkout
        uses: actions/checkout@v4

      - name: Login to Azure
        uses: azure/login@v1
        with:
          creds: ${{ secrets.AZURE_CREDENTIALS }}

      - name: Get version from staging
        id: version
        run: |
          # Get the version currently running in staging
          VERSION=$(az containerapp show \
            --name orderflow-api \
            --resource-group orderflow-staging-rg \
            --query "properties.template.containers[0].image" \
            --output tsv | cut -d':' -f2)
          echo "version=${VERSION}" >> $GITHUB_OUTPUT
          echo "Deploying version: ${VERSION}"

      - name: Create deployment record
        run: |
          # Record deployment for rollback purposes
          az storage blob upload \
            --account-name orderflowdeployments \
            --container-name deployments \
            --name "production/$(date +%Y%m%d_%H%M%S)_${{ steps.version.outputs.version }}.json" \
            --content '{"version":"${{ steps.version.outputs.version }}","deployer":"${{ github.actor }}","sha":"${{ github.sha }}"}'

      - name: Deploy with traffic splitting
        run: |
          # Deploy new revision with 0% traffic
          az containerapp update \
            --name orderflow-api \
            --resource-group orderflow-production-rg \
            --image ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}:${{ steps.version.outputs.version }} \
            --revision-suffix "v${{ github.run_number }}"

          NEW_REVISION="orderflow-api--v${{ github.run_number }}"

          # Canary: 10% traffic to new revision
          echo "Starting canary deployment (10% traffic)..."
          az containerapp ingress traffic set \
            --name orderflow-api \
            --resource-group orderflow-production-rg \
            --revision-weight "${NEW_REVISION}=10"

          sleep 60

          # Check error rate (simplified - use proper monitoring in production)
          # If errors are acceptable, continue rollout

          # 50% traffic
          echo "Increasing to 50% traffic..."
          az containerapp ingress traffic set \
            --name orderflow-api \
            --resource-group orderflow-production-rg \
            --revision-weight "${NEW_REVISION}=50"

          sleep 60

          # 100% traffic
          echo "Rolling out to 100% traffic..."
          az containerapp ingress traffic set \
            --name orderflow-api \
            --resource-group orderflow-production-rg \
            --revision-weight "${NEW_REVISION}=100"

      - name: Verify deployment
        run: |
          PROD_URL="https://api.orderflow.example.com"

          for i in {1..5}; do
            STATUS=$(curl -s -o /dev/null -w "%{http_code}" "${PROD_URL}/health")
            if [ "$STATUS" == "200" ]; then
              echo "Health check passed (attempt ${i})"
              exit 0
            fi
            echo "Health check failed (attempt ${i}), retrying..."
            sleep 10
          done

          echo "::error::Production health check failed after 5 attempts"
          exit 1

      - name: Create GitHub release
        uses: softprops/action-gh-release@v1
        with:
          tag_name: v${{ steps.version.outputs.version }}
          name: Release ${{ steps.version.outputs.version }}
          generate_release_notes: true

      - name: Notify success
        uses: slackapi/slack-github-action@v1
        with:
          payload: |
            {
              "text": "Production deployment successful!",
              "blocks": [
                {
                  "type": "section",
                  "text": {
                    "type": "mrkdwn",
                    "text": "*Production Deployment Successful* :rocket:\n*Version:* ${{ steps.version.outputs.version }}\n*Deployed by:* ${{ github.actor }}"
                  }
                }
              ]
            }
        env:
          SLACK_WEBHOOK_URL: ${{ secrets.SLACK_WEBHOOK_URL }}

  # ============================================
  # ROLLBACK (Manual trigger only)
  # ============================================
  rollback:
    name: Rollback Production
    runs-on: ubuntu-latest
    if: github.event_name == 'workflow_dispatch' && failure()
    environment:
      name: production

    steps:
      - name: Login to Azure
        uses: azure/login@v1
        with:
          creds: ${{ secrets.AZURE_CREDENTIALS }}

      - name: Get previous revision
        id: previous
        run: |
          # Get the second most recent revision (the one before current)
          PREVIOUS=$(az containerapp revision list \
            --name orderflow-api \
            --resource-group orderflow-production-rg \
            --query "sort_by([?properties.trafficWeight==\`0\`], &properties.createdTime)[-1].name" \
            --output tsv)
          echo "revision=${PREVIOUS}" >> $GITHUB_OUTPUT

      - name: Rollback
        run: |
          az containerapp ingress traffic set \
            --name orderflow-api \
            --resource-group orderflow-production-rg \
            --revision-weight "${{ steps.previous.outputs.revision }}=100"

      - name: Notify rollback
        uses: slackapi/slack-github-action@v1
        with:
          payload: |
            {
              "text": "Production rolled back!",
              "blocks": [
                {
                  "type": "section",
                  "text": {
                    "type": "mrkdwn",
                    "text": "*Production Rollback Executed* :warning:\n*Rolled back to:* ${{ steps.previous.outputs.revision }}\n*Triggered by:* ${{ github.actor }}"
                  }
                }
              ]
            }
        env:
          SLACK_WEBHOOK_URL: ${{ secrets.SLACK_WEBHOOK_URL }}
```

---

## Quality Gates

### Enforcing Standards

```yaml
# Required status checks in branch protection
# Configure in: Settings > Branches > Branch protection rules

# Minimum checks before merge:
# - build (Required)
# - code-quality (Required)
# - security (Required)

# Example: Fail if formatting is wrong
- name: Check Formatting
  run: |
    dotnet format --verify-no-changes
    if [ $? -ne 0 ]; then
      echo "::error::Code formatting check failed. Run 'dotnet format' locally."
      exit 1
    fi

# Example: Fail if coverage drops
- name: Coverage Gate
  run: |
    CURRENT=$(cat coverage.txt | grep "Line" | awk '{print $2}' | tr -d '%')
    THRESHOLD=80
    if (( $(echo "$CURRENT < $THRESHOLD" | bc -l) )); then
      echo "::error::Coverage ${CURRENT}% is below ${THRESHOLD}% threshold"
      exit 1
    fi

# Example: Fail on critical vulnerabilities
- name: Vulnerability Gate
  run: |
    dotnet list package --vulnerable --include-transitive > vuln.txt
    if grep -q "Critical" vuln.txt; then
      echo "::error::Critical vulnerabilities found!"
      cat vuln.txt
      exit 1
    fi
```

### PR Labeling and Size Checks

```yaml
# .github/workflows/pr-checks.yml
name: PR Checks

on:
  pull_request:
    types: [opened, synchronize, reopened]

jobs:
  size-check:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 0

      - name: Check PR size
        run: |
          ADDITIONS=$(git diff --numstat origin/${{ github.base_ref }}...HEAD | awk '{s+=$1} END {print s}')
          DELETIONS=$(git diff --numstat origin/${{ github.base_ref }}...HEAD | awk '{s+=$2} END {print s}')
          TOTAL=$((ADDITIONS + DELETIONS))

          echo "Changes: +${ADDITIONS} -${DELETIONS} (total: ${TOTAL})"

          if [ $TOTAL -gt 500 ]; then
            echo "::warning::Large PR detected (${TOTAL} lines). Consider breaking it up."
          fi

          if [ $TOTAL -gt 1000 ]; then
            echo "::error::PR too large (${TOTAL} lines). Please split into smaller PRs."
            exit 1
          fi

      - name: Label PR
        uses: actions/github-script@v7
        with:
          script: |
            const { data: files } = await github.rest.pulls.listFiles({
              owner: context.repo.owner,
              repo: context.repo.repo,
              pull_number: context.issue.number
            });

            const labels = [];
            const changes = files.reduce((acc, f) => acc + f.changes, 0);

            // Size labels
            if (changes < 50) labels.push('size/XS');
            else if (changes < 200) labels.push('size/S');
            else if (changes < 500) labels.push('size/M');
            else if (changes < 1000) labels.push('size/L');
            else labels.push('size/XL');

            // Type labels based on files
            if (files.some(f => f.filename.includes('test'))) labels.push('tests');
            if (files.some(f => f.filename.endsWith('.md'))) labels.push('documentation');
            if (files.some(f => f.filename.includes('.github/workflows'))) labels.push('ci/cd');

            await github.rest.issues.addLabels({
              owner: context.repo.owner,
              repo: context.repo.repo,
              issue_number: context.issue.number,
              labels: labels
            });
```

---

## Security Scanning

### Dependency Scanning with Dependabot

```yaml
# .github/dependabot.yml
version: 2
updates:
  # NuGet packages
  - package-ecosystem: "nuget"
    directory: "/"
    schedule:
      interval: "weekly"
      day: "monday"
      time: "09:00"
      timezone: "UTC"
    open-pull-requests-limit: 10
    reviewers:
      - "security-team"
    labels:
      - "dependencies"
      - "security"
    groups:
      # Group minor/patch updates together
      minor-and-patch:
        patterns:
          - "*"
        update-types:
          - "minor"
          - "patch"
    ignore:
      # Ignore major version updates for framework packages
      - dependency-name: "Microsoft.AspNetCore.*"
        update-types: ["version-update:semver-major"]

  # GitHub Actions
  - package-ecosystem: "github-actions"
    directory: "/"
    schedule:
      interval: "weekly"
    labels:
      - "ci/cd"
      - "dependencies"

  # Docker base images
  - package-ecosystem: "docker"
    directory: "/src/OrderFlow.Api"
    schedule:
      interval: "weekly"
    labels:
      - "docker"
      - "dependencies"
```

### CodeQL Configuration

```yaml
# .github/codeql/codeql-config.yml
name: "OrderFlow CodeQL Config"

queries:
  - uses: security-and-quality
  - uses: security-extended

paths-ignore:
  - '**/Tests/**'
  - '**/test/**'
  - '**/*.test.cs'

# Custom queries for .NET
query-filters:
  - exclude:
      id: cs/weak-crypto

# Threat model specific queries
packs:
  - codeql/csharp-queries
```

### Container Scanning

```yaml
- name: Scan container with Trivy
  uses: aquasecurity/trivy-action@master
  with:
    image-ref: '${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}:${{ github.sha }}'
    format: 'table'
    exit-code: '1'
    ignore-unfixed: true
    vuln-type: 'os,library'
    severity: 'CRITICAL,HIGH'

- name: Scan container with Grype
  uses: anchore/scan-action@v3
  with:
    image: '${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}:${{ github.sha }}'
    fail-build: true
    severity-cutoff: high
```

---

## Deployment Strategies

### Blue-Green Deployment

```
┌─────────────────────────────────────────────────────────────┐
│                      LOAD BALANCER                          │
│                     (Traffic Router)                        │
└─────────────────────────────┬───────────────────────────────┘
                              │
              ┌───────────────┴───────────────┐
              │                               │
              ▼                               ▼
     ┌─────────────────┐           ┌─────────────────┐
     │   BLUE (v1.0)   │           │  GREEN (v1.1)   │
     │   [ACTIVE]      │           │   [STANDBY]     │
     │                 │           │                 │
     │  ┌───────────┐  │           │  ┌───────────┐  │
     │  │   Pod 1   │  │           │  │   Pod 1   │  │
     │  └───────────┘  │           │  └───────────┘  │
     │  ┌───────────┐  │           │  ┌───────────┐  │
     │  │   Pod 2   │  │           │  │   Pod 2   │  │
     │  └───────────┘  │           │  └───────────┘  │
     │  ┌───────────┐  │           │  ┌───────────┐  │
     │  │   Pod 3   │  │           │  │   Pod 3   │  │
     │  └───────────┘  │           │  └───────────┘  │
     └─────────────────┘           └─────────────────┘

After switch:
- Green becomes active (100% traffic)
- Blue becomes standby (instant rollback available)
```

### Canary Deployment

```
Time 0: Deploy canary
┌────────────────────────────────────────────────────────────┐
│                        TRAFFIC SPLIT                        │
│                    95% → v1.0 (Stable)                      │
│                     5% → v1.1 (Canary)                      │
└────────────────────────────────────────────────────────────┘

Time 1: Increase if healthy
┌────────────────────────────────────────────────────────────┐
│                        TRAFFIC SPLIT                        │
│                    75% → v1.0 (Stable)                      │
│                    25% → v1.1 (Canary)                      │
└────────────────────────────────────────────────────────────┘

Time 2: Increase if still healthy
┌────────────────────────────────────────────────────────────┐
│                        TRAFFIC SPLIT                        │
│                    50% → v1.0 (Stable)                      │
│                    50% → v1.1 (Canary)                      │
└────────────────────────────────────────────────────────────┘

Time 3: Full rollout
┌────────────────────────────────────────────────────────────┐
│                        TRAFFIC SPLIT                        │
│                     0% → v1.0 (Old)                         │
│                   100% → v1.1 (New)                         │
└────────────────────────────────────────────────────────────┘
```

### Rolling Deployment

```yaml
# Kubernetes rolling update
spec:
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxUnavailable: 1    # At most 1 pod down at a time
      maxSurge: 1          # At most 1 extra pod during update

  replicas: 4

# Timeline:
# Start: [v1] [v1] [v1] [v1]
# Step 1: [v1] [v1] [v1] [v2] ← New pod starting
# Step 2: [v1] [v1] [v2] [v2] ← Old pod terminated
# Step 3: [v1] [v2] [v2] [v2]
# Step 4: [v2] [v2] [v2] [v2] ← Complete
```

---

## Pipeline Optimization

### Caching Strategies

```yaml
# Multi-layer caching for faster builds
- name: Cache NuGet packages
  uses: actions/cache@v4
  with:
    path: |
      ~/.nuget/packages
      !~/.nuget/packages/unwanted
    key: nuget-${{ runner.os }}-${{ hashFiles('**/*.csproj', '**/Directory.Packages.props') }}
    restore-keys: |
      nuget-${{ runner.os }}-

- name: Cache build outputs
  uses: actions/cache@v4
  with:
    path: |
      **/bin
      **/obj
    key: build-${{ runner.os }}-${{ hashFiles('**/*.cs', '**/*.csproj') }}
    restore-keys: |
      build-${{ runner.os }}-

# Docker layer caching
- name: Build with cache
  uses: docker/build-push-action@v5
  with:
    cache-from: type=gha
    cache-to: type=gha,mode=max
```

### Parallelization

```yaml
jobs:
  # Run these in parallel
  unit-tests:
    runs-on: ubuntu-latest
    steps:
      - run: dotnet test --filter "Category!=Integration"

  integration-tests:
    runs-on: ubuntu-latest
    services:
      postgres: ...
    steps:
      - run: dotnet test --filter "Category=Integration"

  code-quality:
    runs-on: ubuntu-latest
    steps:
      - run: dotnet format --verify-no-changes

  security-scan:
    runs-on: ubuntu-latest
    steps:
      - run: dotnet list package --vulnerable

  # This waits for all above to complete
  deploy:
    needs: [unit-tests, integration-tests, code-quality, security-scan]
    runs-on: ubuntu-latest
    steps:
      - run: echo "All checks passed, deploying..."
```

### Fail Fast

```yaml
strategy:
  fail-fast: true  # Cancel other jobs if one fails
  matrix:
    project: [Api, Worker, Gateway]

# Or use continue-on-error for non-critical checks
- name: Optional lint check
  continue-on-error: true
  run: dotnet format --verify-no-changes
```

---

## Secrets Management

### GitHub Secrets

```yaml
# Use secrets in workflows
env:
  API_KEY: ${{ secrets.API_KEY }}

# Environment-specific secrets
jobs:
  deploy:
    environment: production  # Uses production secrets
    steps:
      - run: echo "Using ${{ secrets.PROD_CONNECTION_STRING }}"
```

### Azure Key Vault Integration

```yaml
- name: Get secrets from Key Vault
  uses: Azure/get-keyvault-secrets@v1
  with:
    keyvault: "orderflow-keyvault"
    secrets: 'DatabasePassword, ApiKey, JwtSecret'
  id: keyvault

- name: Use secrets
  run: |
    echo "Password length: ${#DATABASE_PASSWORD}"
  env:
    DATABASE_PASSWORD: ${{ steps.keyvault.outputs.DatabasePassword }}
```

### OIDC Authentication (No Stored Credentials)

```yaml
# Modern approach: Workload Identity Federation
permissions:
  id-token: write
  contents: read

- name: Login to Azure with OIDC
  uses: azure/login@v1
  with:
    client-id: ${{ secrets.AZURE_CLIENT_ID }}
    tenant-id: ${{ secrets.AZURE_TENANT_ID }}
    subscription-id: ${{ secrets.AZURE_SUBSCRIPTION_ID }}
```

---

## Hands-On Exercise: Build OrderFlow Pipeline

### Step 1: Create CI Workflow

Create `.github/workflows/ci.yml` with:
- Build and test stages
- Code coverage reporting
- Security scanning

### Step 2: Add Quality Gates

Configure branch protection:
1. Go to Settings > Branches
2. Add rule for `main`
3. Require status checks: build, test, security
4. Require PR reviews

### Step 3: Create CD Workflow

Create `.github/workflows/cd.yml` with:
- Staging deployment (auto on main)
- Production deployment (manual approval)
- Smoke tests after deploy

### Step 4: Add Dependabot

Create `.github/dependabot.yml` for:
- NuGet packages
- GitHub Actions
- Docker base images

---

## Deliverables

1. **CI Pipeline**: Complete `ci.yml` with build, test, scan stages
2. **CD Pipeline**: `cd.yml` with staging and production deployments
3. **Quality Gates**: Branch protection configuration
4. **Security Scanning**: Dependabot and CodeQL setup
5. **Documentation**: Pipeline troubleshooting guide

---

## Reflection Questions

1. What's the difference between continuous delivery and continuous deployment?
2. Why should you fail fast in CI but use canary deploys in CD?
3. How do you handle database migrations in a zero-downtime deployment?
4. What metrics would you track to measure pipeline health?

---

## Resources

### GitHub Actions
- [GitHub Actions Documentation](https://docs.github.com/en/actions)
- [Actions Marketplace](https://github.com/marketplace?type=actions)
- [Awesome Actions](https://github.com/sdras/awesome-actions)
- [GitHub Actions for .NET](https://www.youtube.com/watch?v=R8_veQiYBjI) - Nick Chapsas

### CI/CD Practices
- [Continuous Delivery](https://www.oreilly.com/library/view/continuous-delivery/9780321670250/) - Jez Humble
- [The DevOps Handbook](https://www.oreilly.com/library/view/the-devops-handbook/9781457191381/)
- [Deployment Strategies](https://www.youtube.com/watch?v=AWVTKBUnoIg) - TechWorld with Nana

### Security
- [GitHub Security Features](https://docs.github.com/en/code-security)
- [CodeQL Documentation](https://codeql.github.com/docs/)
- [Trivy](https://trivy.dev/)
- [OWASP CI/CD Security](https://owasp.org/www-project-devsecops-guideline/latest/02a-Build-and-CI)

### Azure DevOps Alternative
- [Azure Pipelines YAML](https://learn.microsoft.com/en-us/azure/devops/pipelines/yaml-schema)
- [Azure DevOps Tutorial](https://www.youtube.com/watch?v=4BibQ69MD8c) - Adam Marczak

---

**Next:** [Part 3 — Containers & Docker](./part-3.md)
