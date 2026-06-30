# US-006 — Production ready

## Status

Planned

## Summary

As a platform engineer,
I want the Hermes platform packaged as production-grade Docker images, validated by an automated CI pipeline, deployed to AWS EKS via Terraform-provisioned infrastructure, and observable from day one,
so that the platform can be shipped, operated, and scaled reliably in a production environment without manual steps.

## Business / Engineering value

All prior stories deliver working software on a developer's machine. This story makes that software production-operable: containerized, automatically tested on every change, deployed to managed Kubernetes on AWS, and backed by managed infrastructure provisioned as code.

The key deliverables are:

- **Docker images**: immutable, minimal, versioned container images for `hermes-gateway` and `hermes-admin`, pushed to Amazon ECR on every merge to `main`.
- **GitHub Actions CI/CD**: build → test → vulnerability scan → Docker push → EKS deploy, all gated on the feature branch workflow established in the engineering rules.
- **Terraform baseline**: VPC, EKS cluster, RDS PostgreSQL, ElastiCache Redis, ECR, ALB, WAF, ACM, and Route 53 provisioned as code. State in S3 with DynamoDB locking.
- **Kubernetes manifests**: `hermes-gateway` and `hermes-admin` Deployments with IRSA service accounts, readiness/liveness probes, resource limits, HPA, and Kustomize overlays per environment.

This milestone closes the gap between "works locally" and "runs in production."

## Scope

This story includes:

- Multi-stage `Dockerfile` for `hermes-gateway` and `hermes-admin` (JDK build stage → JRE runtime stage).
- `.dockerignore` to exclude build artifacts and secrets from image context.
- `.github/workflows/ci.yml`: build, test, OWASP dependency-check, Docker build and push to ECR (on `main`).
- `.github/workflows/deploy.yml`: triggered after CI on `main`; applies Kustomize overlays to EKS.
- GitHub OIDC federation with AWS IAM (no long-lived `AWS_ACCESS_KEY_ID` secrets ever committed).
- `infra/terraform/`: VPC, EKS, RDS, ElastiCache, ECR, ALB, WAF modules with a `prod` workspace.
- `infra/k8s/`: base manifests + `overlays/prod/` and `overlays/dev/` via Kustomize.
- `hermes-gateway` and `hermes-admin` Deployments with IRSA service accounts, resource requests/limits, HPA (CPU-based), PodDisruptionBudget.
- Readiness probe: `GET /actuator/health/readiness`; Liveness probe: `GET /actuator/health/liveness`.
- AWS Secrets Manager + External Secrets Operator for DB credentials injection.

## Out of scope

This story does not include:

- Grafana/Prometheus operator setup (Prometheus endpoint is delivered; scraping config is ops responsibility).
- Multi-region deployment.
- Blue/green or canary deployment strategy (rolling update is the default Kubernetes strategy and sufficient for v1.0.0).
- Disaster recovery runbooks.
- Performance load testing.
- Cost optimization (instance type selection, spot instances) — deferred post-launch.

## Acceptance criteria

- [ ] `docker build -f hermes-gateway/Dockerfile -t hermes-gateway:local .` succeeds and produces an image under 300 MB.
- [ ] The image starts and `GET /actuator/health` returns `{"status":"UP"}`.
- [ ] CI pipeline runs on every push to a feature/fix/docs/chore/ci branch and on every PR to `main`.
- [ ] A push to `main` triggers Docker image build, ECR push, and EKS rolling deployment automatically.
- [ ] `terraform plan` produces no errors against the `prod` workspace.
- [ ] `kubectl get pods -n hermes` shows all pods `Running` after deployment.
- [ ] Liveness and readiness probes pass on both `hermes-gateway` and `hermes-admin` pods.
- [ ] No long-lived AWS credentials appear anywhere in the repository (all AWS access via OIDC or IRSA).

## Technical notes

- **Java runtime image**: `eclipse-temurin:21-jre-alpine`. The JRE-only image is ~80 MB smaller than the JDK image. Alpine base minimizes CVE exposure.
- **Spring Boot layer index**: enable `<layers><enabled>true</enabled></layers>` in `spring-boot-maven-plugin` for cache-friendly layer ordering (dependencies → snapshot dependencies → resources → application).
- **OIDC federation**: GitHub Actions authenticates to AWS using `aws-actions/configure-aws-credentials@v4` with `role-to-assume`. The IAM role's trust policy allows `token.actions.githubusercontent.com` as the identity provider, scoped to the specific repository and `main` branch. No `AWS_ACCESS_KEY_ID` or `AWS_SECRET_ACCESS_KEY` secrets.
- **IRSA (IAM Roles for Service Accounts)**: each Kubernetes `ServiceAccount` is annotated with `eks.amazonaws.com/role-arn`. Pods assume their IAM role without static credentials. Roles grant least-privilege access: `hermes-gateway` gets ElastiCache read; `hermes-admin` gets RDS access and Secrets Manager read.
- **Kustomize**: base manifests in `infra/k8s/base/`; environment overrides in `infra/k8s/overlays/{dev,prod}/`. Production overlay sets replica counts, resource limits, and the prod image tag. Dev overlay sets `replicas: 1` and lower resource limits.
- **Dynamic route reload integration** (carried forward from US-004 out-of-scope): in v1.0.0, `hermes-gateway` reads routes from a `RouteDefinitionRepository` backed by PostgreSQL via a read-only datasource, completing the dynamic route reload loop deferred from US-004.

## Related tasks

- `TASK-001-docker-image.md`
- `TASK-002-github-actions-ci.md`
- `TASK-003-terraform-baseline.md`
- `TASK-004-eks-deploy.md`

## Dependencies

- All prior stories (`US-001` through `US-005`) must be `Done`.

## Validation

Local Docker build:

```bash
docker build -f hermes-gateway/Dockerfile -t hermes-gateway:local .
docker run --rm -p 8080:8080 hermes-gateway:local
curl http://localhost:8080/actuator/health
```

CI: open a PR to `main` and verify the `ci` workflow runs green.

Deployment: merge to `main` and verify `kubectl rollout status deployment/hermes-gateway -n hermes` completes successfully.

## Notes

- The `main` branch protection rules (required CI pass, no direct commits) established in the engineering rules from CLAUDE.md are enforced by GitHub branch protection settings — configure these as part of this milestone.
- Image tags follow the pattern `{ECR_REGISTRY}/hermes-gateway:{git-sha}` for immutability. The `latest` tag is never used in production manifests — every deployment references a specific SHA.
- The Terraform state bucket and DynamoDB table must be bootstrapped manually before `terraform init` can run. Document this in the `infra/terraform/README.md` as a one-time setup step.
