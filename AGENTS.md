# Petclinic Platform — Codex Instructions

This repository contains all infrastructure code for deploying Spring Petclinic Microservices to AWS. The application repository (`spring-petclinic-microservices`) is read-only; never modify it from this project.

## Working Style

- Read `docs/technical-spec.md` before implementing a backlog story. It is the source of truth for CIDRs, ports, instance sizes, security groups, Kubernetes resources, probe timings, and alert thresholds.
- Check `docs/jira-backlog.md` for scope and dependency order.
- Before editing a specialized area, read its applicable guide: `.codex/guides/rules/terraform.md`, `kubernetes.md`, `helm.md`, `pipelines.md`, or `docs.md`.
- Inspect existing code and preserve user changes before editing.
- Keep changes narrowly scoped. Do not create or modify live AWS, Kubernetes, ArgoCD, or Jira resources unless the user explicitly asks.
- Prefer read-only validation first. Explain any operation that can incur cost, alter infrastructure, or affect production.
- Never modify generated Terraform state, saved plans, credentials, `.env` files, private keys, or real `*.tfvars` files.

## Repository Layout

```text
terraform/environments/{dev,prod}/   # Root modules
terraform/modules/                   # Reusable AWS modules
helm/petclinic-service/              # Generic chart shared by all services
helm-values/                         # Per-service and per-environment values
k8s/base/                            # Shared Kubernetes resources
k8s/argocd/                          # ArgoCD installation and applications
.github/workflows/                   # CI workflows; ArgoCD performs CD
scripts/                             # Operational scripts
docs/                                # Specifications, runbooks, and ADRs
```

Directories may be introduced incrementally as backlog stories are implemented. Do not treat a documented future path as an existing file.

## Terraform Conventions

- Use Terraform >= 1.6 and AWS provider `~> 5.0` in `eu-central-1`.
- Store state in S3 with DynamoDB locking using `petclinic/{env}/terraform.tfstate`.
- Put reusable code in `terraform/modules/`; environment roots call those modules.
- Name AWS resources `petclinic-{env}-{resource}`.
- Tag every supported resource with `Project=petclinic`, `Environment={dev|prod}`, and `ManagedBy=terraform`.
- Give variables a description and type; add defaults only when sensible. Mark secret outputs sensitive.
- Each module should have `main.tf`, `variables.tf`, `outputs.tf`, and `versions.tf`.
- Run `terraform fmt -recursive` and the applicable `terraform validate` after edits.
- Always create and review a saved plan before applying: `terraform plan -out plan.out`, then `terraform apply plan.out`.
- Never run `terraform destroy`, `terragrunt destroy`, or `terraform apply -destroy`.

## Kubernetes and Helm Conventions

- Use namespaces `petclinic-dev` and `petclinic-prod`.
- Add `app.kubernetes.io/name`, `app.kubernetes.io/part-of=petclinic`, and `app.kubernetes.io/managed-by=Helm` labels.
- Every Deployment needs readiness and liveness probes at `/actuator/health/readiness` and `/actuator/health/liveness`.
- Every container needs requests and limits; the baseline is 128Mi requested and 512Mi limited.
- Use commit-SHA image tags. Never use `latest` in production.
- Use ExternalSecret resources backed by AWS Secrets Manager; never place secrets in YAML.
- Preserve startup order: Config Server, Discovery Server, then application services. Use init containers where required.
- Maintain one generic chart in `helm/petclinic-service/`, with service and environment values in `helm-values/`.
- Validate charts with `helm lint` and `helm template` before committing.

## ArgoCD and CI/CD Conventions

- GitHub Actions performs CI only: build, scan, push, and update image tags. It must never run `kubectl apply` or `helm upgrade`.
- Authenticate to AWS from GitHub Actions through OIDC; never use long-lived access keys.
- Pin third-party actions to immutable commit SHAs.
- Run Trivy after image builds and fail on critical vulnerabilities.
- Dev ArgoCD applications auto-sync with prune and self-heal. Production sync is manual.
- Create one ArgoCD Application per service per environment.

## Security Rules

1. No secrets in code, examples, logs, Terraform variables, manifests, or Git history.
2. Block public access on every S3 bucket.
3. Allow `0.0.0.0/0` ingress only for an ALB on ports 80 and 443.
4. Encrypt RDS, S3, EBS, and Secrets Manager data at rest.
5. Use least-privilege IAM with specific actions and resource ARNs; avoid wildcard action/resource pairs.
6. This learning project intentionally uses public subnets without NAT to reduce cost; security groups are the perimeter. Follow ADR-0001 when present.
7. Never destroy Terraform infrastructure or critical production Kubernetes resources through Codex.
8. Never stage or commit `.env`, `*.tfvars`, `*.pem`, `*.key`, kubeconfig, or credential files. Avoid bulk `git add .` and `git add -A`; stage explicit paths.

## Environment Baseline

| Setting | Dev | Prod |
|---|---|---|
| Region | eu-central-1 | eu-central-1 |
| Namespace | petclinic-dev | petclinic-prod |
| State key | petclinic/dev/terraform.tfstate | petclinic/prod/terraform.tfstate |
| RDS | db.t4g.micro, single-AZ | db.t4g.micro, single-AZ |
| EKS nodes | 2 x t4g.small ARM | 2 x t4g.small ARM |
| Deployment | ArgoCD auto-sync | ArgoCD manual sync |
| Replicas | 1 per service | 2+ per service with HPA |

## Application Services

| Service | Port | MySQL | Notes |
|---|---:|---|---|
| config-server | 8888 | No | Starts first |
| discovery-server | 8761 | No | Starts second |
| api-gateway | 8080 | No | Public-facing frontend and routing |
| customers-service | 8081 | Yes | Owners and pets |
| visits-service | 8082 | Yes | Visit records |
| vets-service | 8083 | Yes | Vet data |
| genai-service | 8084 | Optional | Uses `OPENAI_API_KEY` from a secret |
| admin-server | 9090 | No | Spring Boot Admin |

Build container images for `linux/arm64` using Eclipse Temurin 17, with a 512M application memory limit and commit-SHA tags. Use `SPRING_PROFILES_ACTIVE=docker`; add `mysql` for RDS-backed services.

## Validation Commands

Run only checks relevant to the changed files and report tools that are unavailable:

```bash
terraform fmt -recursive -check
terraform validate
terraform plan -out plan.out
helm lint helm/petclinic-service/ -f helm-values/{service}.yaml -f helm-values/{env}.yaml
helm template petclinic helm/petclinic-service/ -f helm-values/{service}.yaml -f helm-values/{env}.yaml
kubectl apply --dry-run=client -f {manifest}
checkov -d terraform/modules/{module}
```

Do not apply a plan, deploy, sync ArgoCD, mutate Jira, or use a live cluster merely to validate a code change.

## Codex Project Features

- Repository skills live in `.agents/skills/` and can be invoked as `$deploy-dev`, `$deploy-prod`, `$logs`, `$review-terraform`, `$rollback`, `$security-scan`, `$smoke-test`, `$terraform-apply`, and `$terraform-plan`.
- Read-only specialist agents live in `.codex/agents/`. Use them when the user explicitly asks for delegation or parallel agent work.
- Detailed project and reviewer guidance lives in `.codex/guides/`; keep it synchronized with the root instructions when conventions change.
- Project MCP servers and environment defaults live in `.codex/config.toml`. MCP availability still depends on local tools, network access, and authentication.
- Safety hooks live in `.codex/hooks.json`. Review and trust them with `/hooks`; they block destructive infrastructure commands and risky secret staging.
- This repository is Codex-only. Do not introduce instructions or configuration for other coding-agent runtimes unless the user explicitly requests them.

## Code Review Rules

- Lead with concrete findings ordered by severity and cite file and line.
- Prioritize secret exposure, destructive behavior, IAM/network over-permission, missing encryption, invalid configuration, deployment regressions, and missing validation.
- Distinguish confirmed defects from recommendations and cost tradeoffs.
- If no findings are found, say so and identify any checks that could not be run.
