# TASK-002 — GitHub Actions CI/CD pipelines

## Status

Planned

## Parent story

`US-006-production-ready.md`

## Summary

Create two GitHub Actions workflows: a CI pipeline that runs on every branch push and PR, and a deploy pipeline that builds and pushes Docker images to ECR and triggers an EKS rolling update on every merge to `main`. All AWS authentication uses GitHub OIDC federation — no long-lived AWS credentials are stored as GitHub secrets.

## Technical scope

This task includes:

- `.github/workflows/ci.yml` — build, test, OWASP dependency-check, coverage gate.
- `.github/workflows/deploy.yml` — Docker build, ECR push, `kubectl rollout` to EKS (triggered on `main` after CI passes).
- IAM role for GitHub Actions OIDC (documented in this spec; provisioned via Terraform in TASK-003).
- GitHub repository secrets required: `AWS_REGION`, `ECR_REGISTRY`, `EKS_CLUSTER_NAME` (no credentials — values only).
- `CODEOWNERS` file establishing review requirements for `main` merges.
- Maven `.mvn/maven.config` with default CI flags (`--batch-mode --no-transfer-progress`).

## Out of scope

This task does not include:

- Pull request environments (ephemeral preview deployments — deferred).
- Slack or email notifications on failure.
- Matrix builds across multiple Java versions (Java 21 is the only supported version).
- Container image scanning beyond OWASP dependency check (Trivy scan is a recommended follow-up).
- Automated rollback on deployment failure (requires health check polling — deferred).

## Implementation notes

### CI workflow triggers

```yaml
on:
  push:
    branches-ignore:
      - main          # deploy.yml handles main
  pull_request:
    branches:
      - main
```

### CI workflow jobs

1. **build-and-test**:
   - `actions/checkout@v4`
   - `actions/setup-java@v4` with `java-version: '21'`, `distribution: 'temurin'`, `cache: 'maven'`
   - `.\mvnw.cmd clean verify --batch-mode`
   - Upload test results as artifact
   - Upload JaCoCo coverage report as artifact

2. **dependency-check**:
   - Depends on `build-and-test`
   - Run OWASP Dependency-Check Maven plugin: `.\mvnw.cmd dependency-check:check`
   - Fail on CVSS score >= 7 (`<failBuildOnCVSS>7</failBuildOnCVSS>`)
   - Upload HTML report as artifact

3. **coverage-gate** (optional — if JaCoCo is configured):
   - Parse JaCoCo XML; fail if line coverage < 70%

### Deploy workflow triggers

```yaml
on:
  push:
    branches:
      - main
```

### Deploy workflow jobs

1. **build-and-push** — runs on `ubuntu-latest`:
   - `actions/checkout@v4`
   - `aws-actions/configure-aws-credentials@v4` with `role-to-assume` (OIDC)
   - `aws-actions/amazon-ecr-login@v2`
   - Build image: `docker build -f hermes-gateway/Dockerfile -t $ECR_REGISTRY/hermes-gateway:$GITHUB_SHA .`
   - Push: `docker push $ECR_REGISTRY/hermes-gateway:$GITHUB_SHA`
   - Repeat for `hermes-admin`

2. **deploy-to-eks** — depends on `build-and-push`:
   - `aws-actions/configure-aws-credentials@v4`
   - Update kubeconfig: `aws eks update-kubeconfig --region $AWS_REGION --name $EKS_CLUSTER_NAME`
   - `kustomize edit set image` to update the prod overlay image tag to `$GITHUB_SHA`
   - `kubectl apply -k infra/k8s/overlays/prod/`
   - `kubectl rollout status deployment/hermes-gateway -n hermes --timeout=5m`
   - `kubectl rollout status deployment/hermes-admin -n hermes --timeout=5m`

### OIDC IAM role trust policy

```json
{
  "Version": "2012-10-17",
  "Statement": [{
    "Effect": "Allow",
    "Principal": {
      "Federated": "arn:aws:iam::{ACCOUNT_ID}:oidc-provider/token.actions.githubusercontent.com"
    },
    "Action": "sts:AssumeRoleWithWebIdentity",
    "Condition": {
      "StringEquals": {
        "token.actions.githubusercontent.com:aud": "sts.amazonaws.com"
      },
      "StringLike": {
        "token.actions.githubusercontent.com:sub": "repo:eithel-gonzalez/hermes-api-platform:ref:refs/heads/main"
      }
    }
  }]
}
```

### Maven CI flags (.mvn/maven.config)

```
--batch-mode
--no-transfer-progress
```

These suppress interactive prompts and download progress bars, producing cleaner CI logs.

### ci.yml reference

```yaml
name: CI

on:
  push:
    branches-ignore:
      - main
  pull_request:
    branches:
      - main

jobs:
  build-and-test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - uses: actions/setup-java@v4
        with:
          java-version: '21'
          distribution: 'temurin'
          cache: 'maven'

      - name: Build and test
        run: ./mvnw clean verify

      - name: Upload test results
        if: always()
        uses: actions/upload-artifact@v4
        with:
          name: test-results
          path: '**/target/surefire-reports/*.xml'

  dependency-check:
    runs-on: ubuntu-latest
    needs: build-and-test
    steps:
      - uses: actions/checkout@v4

      - uses: actions/setup-java@v4
        with:
          java-version: '21'
          distribution: 'temurin'
          cache: 'maven'

      - name: OWASP Dependency Check
        run: ./mvnw dependency-check:check -DfailBuildOnCVSS=7

      - name: Upload dependency check report
        if: always()
        uses: actions/upload-artifact@v4
        with:
          name: dependency-check-report
          path: '**/target/dependency-check-report.html'
```

### deploy.yml reference (abridged)

```yaml
name: Deploy

on:
  push:
    branches:
      - main

permissions:
  id-token: write    # required for OIDC
  contents: read

jobs:
  build-and-push:
    runs-on: ubuntu-latest
    env:
      AWS_REGION: ${{ secrets.AWS_REGION }}
      ECR_REGISTRY: ${{ secrets.ECR_REGISTRY }}
    steps:
      - uses: actions/checkout@v4

      - name: Configure AWS credentials (OIDC)
        uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: ${{ secrets.AWS_DEPLOY_ROLE_ARN }}
          aws-region: ${{ env.AWS_REGION }}

      - name: Login to ECR
        uses: aws-actions/amazon-ecr-login@v2

      - name: Build and push hermes-gateway
        run: |
          docker build -f hermes-gateway/Dockerfile \
            -t $ECR_REGISTRY/hermes-gateway:${{ github.sha }} .
          docker push $ECR_REGISTRY/hermes-gateway:${{ github.sha }}

      - name: Build and push hermes-admin
        run: |
          docker build -f hermes-admin/Dockerfile \
            -t $ECR_REGISTRY/hermes-admin:${{ github.sha }} .
          docker push $ECR_REGISTRY/hermes-admin:${{ github.sha }}

  deploy-to-eks:
    runs-on: ubuntu-latest
    needs: build-and-push
    steps:
      - uses: actions/checkout@v4

      - name: Configure AWS credentials (OIDC)
        uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: ${{ secrets.AWS_DEPLOY_ROLE_ARN }}
          aws-region: ${{ secrets.AWS_REGION }}

      - name: Update kubeconfig
        run: |
          aws eks update-kubeconfig \
            --region ${{ secrets.AWS_REGION }} \
            --name ${{ secrets.EKS_CLUSTER_NAME }}

      - name: Update image tags and deploy
        run: |
          cd infra/k8s/overlays/prod
          kustomize edit set image \
            hermes-gateway=${{ secrets.ECR_REGISTRY }}/hermes-gateway:${{ github.sha }}
          kustomize edit set image \
            hermes-admin=${{ secrets.ECR_REGISTRY }}/hermes-admin:${{ github.sha }}
          kubectl apply -k .

      - name: Wait for rollout
        run: |
          kubectl rollout status deployment/hermes-gateway -n hermes --timeout=5m
          kubectl rollout status deployment/hermes-admin -n hermes --timeout=5m
```

## Expected files

```text
hermes-api-platform/
├── .github/
│   └── workflows/
│       ├── ci.yml
│       └── deploy.yml
├── .mvn/
│   └── maven.config                  ← add CI batch flags
└── CODEOWNERS                        ← require review for main merges
```

## Acceptance criteria

- [ ] `ci.yml` triggers on every push to a non-`main` branch and on PRs to `main`.
- [ ] `ci.yml` fails the PR if any test fails or if OWASP finds a CVSS >= 7 vulnerability.
- [ ] `deploy.yml` triggers only on pushes to `main`.
- [ ] `deploy.yml` pushes images tagged with the commit SHA — never `latest`.
- [ ] `deploy.yml` uses OIDC (`id-token: write`) — no `AWS_ACCESS_KEY_ID` secret exists in the repo.
- [ ] `kubectl rollout status` succeeds within the 5-minute timeout after a merge to `main`.
- [ ] `CODEOWNERS` requires at least one reviewer for changes under `infra/`, `.github/`, and `hermes-gateway/src/`.

## Validation

1. Open a feature branch, push a commit — verify `ci.yml` runs and passes in GitHub Actions.
2. Open a PR to `main` — verify `ci.yml` is a required status check.
3. Merge to `main` — verify `deploy.yml` runs, images are pushed to ECR with the correct SHA tag, and pods roll out successfully.

## Commit suggestion

```text
ci: add GitHub Actions CI pipeline and OIDC-authenticated EKS deploy workflow
```

## Notes

- **`permissions: id-token: write`** must be set at the workflow or job level for OIDC to work. Without it, GitHub does not generate the OIDC token and the `configure-aws-credentials` action fails silently with an auth error.
- **`github.sha` as image tag**: using the full commit SHA as the image tag makes deployments fully reproducible and auditable. Given a pod's image tag, the exact code it runs can be identified with `git show {sha}`.
- **OWASP Dependency-Check first run**: the plugin downloads its vulnerability database (~200 MB) on the first run. The database should be cached in the GitHub Actions cache using `actions/cache@v4` keyed on the OWASP DB date (`nvd.dataMirrorReleaseDate`), or by setting `<autoUpdate>false</autoUpdate>` and providing a pre-seeded database. Without caching, every CI run downloads the full NVD database — slow and rate-limited by NVD.
- **Branch protection settings** (configure in GitHub repository settings, not in code): require `ci` workflow to pass before merging; require at least 1 approver; dismiss stale reviews on new commits; do not allow bypassing required status checks for administrators.
- **`kustomize edit set image`** mutates `kustomization.yaml` in-place during the workflow run. The change is not committed back to the repository — it exists only in the runner's working directory and is applied immediately via `kubectl apply -k`. This is intentional: the Git repository tracks the `latest` overlay configuration, not the per-deployment SHA. The SHA is in the ECR image label and the pod spec for auditability.
