# TASK-003 — Terraform baseline (AWS infrastructure)

## Status

Planned

## Parent story

`US-006-production-ready.md`

## Summary

Provision the AWS infrastructure for the Hermes platform using Terraform: VPC with public/private subnets, EKS cluster, RDS PostgreSQL, ElastiCache Redis, ECR repositories, ALB with WAF, ACM certificate, and Route 53 DNS. State is stored in an S3 bucket with DynamoDB locking. GitHub OIDC provider and IRSA roles are also provisioned here.

## Technical scope

This task includes:

- `infra/terraform/` module structure (see below).
- Remote state backend: S3 bucket + DynamoDB table (bootstrapped manually once).
- Terraform workspaces: `prod` (primary). `dev` workspace is optional — uses smaller instance types.
- All AWS resources required for the platform to operate:
  - **VPC**: 3 AZs, public subnets (ALB), private subnets (EKS nodes, RDS, ElastiCache).
  - **EKS**: managed node group, OIDC provider for IRSA.
  - **RDS**: PostgreSQL 16, Multi-AZ, encrypted, in private subnets.
  - **ElastiCache**: Redis 7, cluster mode disabled (single shard, 1 replica), in private subnets.
  - **ECR**: two repositories (`hermes-gateway`, `hermes-admin`), image scanning enabled.
  - **ALB**: HTTPS listener (ACM cert), HTTP→HTTPS redirect, WAF association.
  - **WAF**: AWS Managed Rules (AWSManagedRulesCommonRuleSet, AWSManagedRulesSQLiRuleSet).
  - **ACM**: certificate for the platform domain, DNS-validated via Route 53.
  - **Route 53**: A record alias pointing the domain to the ALB.
  - **GitHub OIDC IAM provider** + deploy role (used by GitHub Actions in TASK-002).
  - **IRSA roles**: `hermes-gateway-role` (ElastiCache read, Secrets Manager read) and `hermes-admin-role` (RDS access, Secrets Manager read).
  - **AWS Secrets Manager**: stores DB credentials and JWT signing key. Read by pods via IRSA.

## Out of scope

This task does not include:

- Multi-region setup.
- AWS Backup configuration for RDS.
- CloudWatch alarms and dashboards (ops responsibility post-launch).
- VPN or AWS Direct Connect.
- Cost budget alerts.

## Implementation notes

### Module layout

```text
infra/terraform/
├── main.tf               ← root module, workspace-aware locals
├── variables.tf
├── outputs.tf
├── backend.tf            ← S3 + DynamoDB remote state config
├── versions.tf           ← required_providers, terraform version constraint
└── modules/
    ├── vpc/              ← VPC, subnets, NAT Gateway, route tables
    ├── eks/              ← EKS cluster, managed node group, OIDC provider
    ├── rds/              ← RDS PostgreSQL, subnet group, security group
    ├── elasticache/      ← ElastiCache Redis, subnet group, security group
    ├── ecr/              ← ECR repositories, lifecycle policies
    ├── alb/              ← ALB, listeners, target groups
    ├── waf/              ← WAFv2 web ACL, managed rule groups
    ├── acm/              ← ACM certificate, Route 53 validation records
    ├── dns/              ← Route 53 hosted zone data source, A record alias
    ├── iam-github-oidc/  ← GitHub Actions OIDC provider + deploy role
    └── iam-irsa/         ← IRSA roles for hermes-gateway and hermes-admin
```

### Key Terraform patterns

```hcl
# backend.tf
terraform {
  backend "s3" {
    bucket         = "hermes-terraform-state-{account_id}"
    key            = "hermes/{env}/terraform.tfstate"
    region         = "us-east-1"
    dynamodb_table = "hermes-terraform-locks"
    encrypt        = true
  }
}

# versions.tf
terraform {
  required_version = ">= 1.9"
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"
    }
  }
}
```

### Workspace-aware sizing

```hcl
# main.tf
locals {
  env = terraform.workspace   # "prod" or "dev"

  eks_node_instance_type = {
    prod = "t3.medium"
    dev  = "t3.small"
  }[local.env]

  rds_instance_class = {
    prod = "db.t3.medium"
    dev  = "db.t3.micro"
  }[local.env]

  multi_az = local.env == "prod"
}
```

### Key resource specifications

| Resource | Prod spec |
|---|---|
| EKS node group | `t3.medium` × min 2 / max 6, On-Demand |
| RDS PostgreSQL | `db.t3.medium`, Multi-AZ, 20 GB gp3, encrypted |
| ElastiCache Redis | `cache.t3.micro`, 1 primary + 1 replica, encrypted at-rest |
| ALB | Internet-facing, HTTPS 443 + HTTP 80 redirect |
| WAF | AWSManagedRulesCommonRuleSet + SQLiRuleSet |
| ECR lifecycle | Keep last 30 images; delete untagged after 1 day |

### Bootstrap (one-time manual steps before `terraform init`)

The S3 backend and DynamoDB table cannot be created by Terraform (circular dependency). Bootstrap manually:

```bash
# Create state bucket (versioning + encryption)
aws s3api create-bucket --bucket hermes-terraform-state-{account_id} \
  --region us-east-1
aws s3api put-bucket-versioning --bucket hermes-terraform-state-{account_id} \
  --versioning-configuration Status=Enabled
aws s3api put-bucket-encryption --bucket hermes-terraform-state-{account_id} \
  --server-side-encryption-configuration \
  '{"Rules":[{"ApplyServerSideEncryptionByDefault":{"SSEAlgorithm":"AES256"}}]}'

# Create DynamoDB lock table
aws dynamodb create-table \
  --table-name hermes-terraform-locks \
  --attribute-definitions AttributeName=LockID,AttributeType=S \
  --key-schema AttributeName=LockID,KeyType=HASH \
  --billing-mode PAY_PER_REQUEST \
  --region us-east-1
```

Document this bootstrap in `infra/terraform/README.md`.

### IRSA role binding example (hermes-gateway)

```hcl
module "hermes_gateway_irsa" {
  source = "./modules/iam-irsa"

  role_name        = "hermes-gateway-role"
  eks_oidc_issuer  = module.eks.oidc_issuer_url
  service_account  = "hermes-gateway"
  namespace        = "hermes"

  policy_arns = [
    aws_iam_policy.elasticache_read.arn,
    aws_iam_policy.secrets_manager_read.arn
  ]
}
```

## Expected files

```text
infra/terraform/
├── main.tf
├── variables.tf
├── outputs.tf
├── backend.tf
├── versions.tf
├── README.md               ← bootstrap instructions
└── modules/
    ├── vpc/
    │   ├── main.tf
    │   ├── variables.tf
    │   └── outputs.tf
    ├── eks/  (same structure)
    ├── rds/
    ├── elasticache/
    ├── ecr/
    ├── alb/
    ├── waf/
    ├── acm/
    ├── dns/
    ├── iam-github-oidc/
    └── iam-irsa/
```

## Acceptance criteria

- [ ] `terraform init -backend-config=...` completes successfully against the S3 backend.
- [ ] `terraform workspace select prod && terraform plan` exits with 0 (no errors) and shows expected resource additions.
- [ ] `terraform apply` provisions all resources without manual intervention.
- [ ] EKS cluster is reachable: `aws eks update-kubeconfig ... && kubectl get nodes` shows nodes in `Ready` state.
- [ ] RDS endpoint is resolvable from within the VPC (private subnet DNS).
- [ ] ElastiCache primary endpoint is resolvable from within the VPC.
- [ ] ECR repositories exist: `aws ecr describe-repositories --repository-names hermes-gateway hermes-admin`.
- [ ] ALB responds on HTTPS with a valid ACM certificate.
- [ ] WAF web ACL is associated with the ALB.
- [ ] GitHub OIDC provider exists in IAM and the deploy role can be assumed from GitHub Actions.

## Validation

```bash
cd infra/terraform
terraform init
terraform workspace select prod
terraform plan -out=tfplan
terraform apply tfplan
```

EKS connectivity:

```bash
aws eks update-kubeconfig --region us-east-1 --name hermes-prod
kubectl get nodes
```

Expected: 2+ nodes in `Ready` state.

## Commit suggestion

```text
feat: add Terraform baseline for EKS, RDS, ElastiCache, ALB, WAF, and IRSA
```

## Notes

- **Do not commit the Terraform state file**: it is in S3. The `.gitignore` should include `*.tfstate`, `*.tfstate.*`, `.terraform/`, and `.terraform.lock.hcl` is the exception — the lock file **should** be committed (it pins provider versions, equivalent to `package-lock.json`).
- **Sensitive outputs**: RDS password and ElastiCache auth token must not appear in `terraform output` as plaintext. Mark them `sensitive = true` in `outputs.tf`. Secrets Manager holds the actual values; Terraform only provisions the Secrets Manager secret resource (the value is rotated outside Terraform).
- **Terraform version constraint**: pin to `>= 1.9` — Terraform 1.9 introduced `for_each` improvements used by the IRSA module. The `.terraform-version` file (for `tfenv`) should match: `1.9.x`.
- **EKS cluster autoscaler or Karpenter**: for v1.0.0, the managed node group has `min_size=2, max_size=6` with static node count. Cluster autoscaler or Karpenter setup is deferred — horizontal pod autoscaler (HPA) handles pod-level scaling within the fixed node pool.
- **WAF rate limiting vs gateway rate limiting**: both are in place. The WAF provides a coarse IP-level rate limit (e.g., 2000 requests per 5 minutes) to block DDoS at the edge, before traffic reaches the gateway. The gateway's Redis-backed rate limiter provides fine-grained per-client limits per route. They are complementary, not redundant.
