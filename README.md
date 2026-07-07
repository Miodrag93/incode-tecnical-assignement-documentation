# Incode Technical Assignment — Documentation

This repository documents the architecture, infrastructure, and delivery pipelines built for the Incode technical assignment: an EKS-based platform running two applications (a Django Ninja backend and a Vue3 frontend, both forks of the [RealWorld](https://github.com/gothinkster/realworld) reference app), provisioned with Terraform and deployed via ArgoCD GitOps.

It does not contain code — it's a map of the six repositories that make up the system, and an explanation of how they fit together.

## Repository map

| Repository | Purpose |
|---|---|
| [`terraform-aws-incode-assignment`](https://github.com/Miodrag93/terraform-aws-incode-assignment) | Root Terraform project — VPC, EKS, RDS, ALB, ECR, IAM, KMS, DNS/TLS |
| [`terraform-aws-modules-bootstrap`](https://github.com/Miodrag93/terraform-aws-modules-bootstrap) | Terraform module: remote state S3 bucket + KMS key + Secrets Manager secret |
| [`terraform-aws-modules-cloudflare-acm`](https://github.com/Miodrag93/terraform-aws-modules-cloudflare-acm) | Terraform module: ACM certificate, DNS-validated via Cloudflare |
| [`argocd-apps-incode`](https://github.com/Miodrag93/argocd-apps-incode) | GitOps repo — ArgoCD App-of-Apps, all Helm charts for cluster add-ons and applications |
| [`realworld-django-ninja-incode`](https://github.com/Miodrag93/realworld-django-ninja-incode) | Backend application (Django Ninja REST API + PostgreSQL) |
| [`vue3-realworld-example-app-incode`](https://github.com/Miodrag93/vue3-realworld-example-app-incode) | Frontend application (Vue 3 SPA, served via nginx) |

### Important DNS hostnames

All served off `crashbackloop.org` (Cloudflare-managed), proxied through Cloudflare and terminated at the ALB:

| Hostname | Points to |
|---|---|
| `dev-argocd.crashbackloop.org` | ArgoCD UI |
| `dev-grafana.crashbackloop.org` | Grafana (VictoriaMetrics stack) |
| `dev-devopsblogs.crashbackloop.org` | Entire application (frontend, which internally proxies `/api/` to the backend) |

## Architecture at a glance

```
                        ┌────────────┐
                        │ Cloudflare │  DNS + proxy for *.crashbackloop.org
                        └─────┬──────┘
                              │ HTTPS
                              ▼
                    ┌───────────────────┐
                    │   AWS ALB (443)   │  ACM cert (wildcard, DNS-validated via Cloudflare)
                    │  ELBSecurityPolicy│  default action: 403 (deny by default)
                    │  -TLS13-1-2-2021  │
                    └─────────┬─────────┘
                              │ HTTP, Host: *.crashbackloop.org
                              ▼
                  ┌────────────────────────┐
                  │ Target Group (IP mode)  │◄── TargetGroupBinding CRD
                  │  health check: /ping    │    (AWS Load Balancer Controller)
                  └───────────┬─────────────┘
                              ▼
                  ┌────────────────────────┐
                  │  Traefik (DaemonSet)    │  IngressRoute CRDs, host-based routing
                  │  ClusterIP :8000        │
                  └───────────┬─────────────┘
              ┌───────────────┼───────────────────┐
              ▼               ▼                   ▼
     incode-frontend    Grafana (dev-grafana)   ArgoCD UI (dev-argocd)
     (nginx SPA)
              │
              │ nginx reverse-proxies /api/ internally to
              ▼
     incode-backend.incode-backend.svc.cluster.local:8000
     (Django Ninja + Gunicorn, never exposed publicly)
              │
              ▼
        RDS PostgreSQL (private subnet, not publicly accessible)
```

Node capacity is provisioned by **Karpenter**, with a tainted `system` NodePool for platform components (ArgoCD, Traefik, LB controller, metrics-server, Fluent Bit, VictoriaMetrics stack) and an untainted `general` NodePool for application workloads.

---

## Infrastructure (Terraform)

Root project: `terraform-aws-incode-assignment`. Single environment (`dev`) under `environments/dev/`; additional environments (`staging`, `prod`) would follow the same directory pattern — one root module per environment, its own backend/provider/variables.

Region: `us-east-1`. Terraform `1.15.7`, AWS provider `6.53.0`, Cloudflare provider `5.21.1`.

### Networking

- VPC `incode-dev-vpc`, CIDR `10.10.0.0/16`, 3 AZs (`us-east-1a/b/c`), via `terraform-aws-modules/vpc/aws` v5.21.0
- Private subnets (`10.10.0.0/23`, `.2.0/23`, `.4.0/23`) — EKS worker nodes
- Public subnets (`10.10.6.0/23`, `.8.0/23`, `.10.0/23`) — ALB, EKS control-plane ENIs
- Database subnets (`10.10.247.0/24`–`.249.0/24`) — RDS, grouped as `incode-dev-db`
- **Single, non-HA NAT gateway** — deliberate lower-environment cost trade-off (documented inline in the Terraform)

### Compute — EKS

- Cluster `incode-cluster-dev`, Kubernetes `1.35`, via `terraform-aws-modules/eks/aws` v21.15.1
- **EKS Auto Mode** (`compute_config.enabled = true`, node pool `system`) — no explicit managed node groups; capacity for actual workloads comes from **Karpenter** (see [GitOps layer](#gitops--kubernetes-layer))
- Both public and private API endpoint access enabled
- Secrets envelope-encrypted with a dedicated customer-managed KMS key (`alias/eks-cluster-encryption`, rotation enabled)
- `authentication_mode = "API"`, cluster creator granted admin permissions
- Addons: `vpc-cni`, `kube-proxy`

### Database — RDS

- PostgreSQL 16.10, `db.t3.micro`, via `terraform-aws-modules/rds/aws` v6.12.0
- `multi_az = false` — deliberate lower-environment trade-off
- Storage: 20 GB allocated, autoscaling to 100 GB
- Master password sourced from Secrets Manager (`dev/infrastructure`), not AWS-managed
- `publicly_accessible = false`; security group only allows 5432 from the VPC CIDR
- `deletion_protection = true`, 7-day backup retention
- Parameter group managed outside the module deliberately, to avoid blocking destroy

### Container registry — ECR

- Two repos: `incode-backend`, `incode-frontend`
- Lifecycle policy: keep last 15 images
- Repository policy grants pull access to the EKS node IAM role

### Load balancing — ALB

- `incode-dev-alb`, internet-facing, public subnets
- HTTPS listener (443) only, TLS 1.3 policy, cert from the Cloudflare-ACM module
- Default action: fixed 403 response (deny by default); host-header rule forwards `*.crashbackloop.org` to the `traefik` target group
- Target group is **IP-mode**, populated dynamically by the in-cluster `TargetGroupBinding` CRD rather than by the AWS Load Balancer Controller provisioning the ALB itself
- Access logs shipped to S3 (`incode-dev-alb-logs`, SSE-AES256)

### DNS & TLS

- DNS hosted on **Cloudflare** for `crashbackloop.org`
- `terraform-aws-modules-cloudflare-acm` module requests a wildcard ACM certificate (`crashbackloop.org` + `*.crashbackloop.org`) and DNS-validates it by writing the validation CNAME into Cloudflare, then blocks on `aws_acm_certificate_validation`
- Root module additionally creates proxied Cloudflare CNAMEs for `dev-argocd`, `dev-devopsblogs` (frontend), and `dev-grafana`, all pointing at the ALB

### State bootstrap

- `terraform-aws-modules-bootstrap` module provisions the Terraform remote-state backend itself: a KMS-encrypted, versioned S3 bucket (`incode-assignment-terraform-state-dev`) and a Secrets Manager secret (`dev/infrastructure`)
- Locking uses **S3 native conditional-write locking** (`use_lockfile = true`, Terraform ≥ 1.10) — no DynamoDB lock table
- Because the same root module creates and then depends on this bucket, bootstrap is a two-step dance: apply with `-target=module.bootstrap` against local state first, then migrate to the S3 backend

### IAM

- IRSA roles (OIDC-trust-scoped to specific namespace/service-account) for: AWS Load Balancer Controller, Fluent Bit, Alertmanager (SNS publish)
- A static-credential IAM user (`github-action-service-account`) for CI, scoped to ECR push/pull — holds a broader-than-minimal `iam:PassRole` grant, worth tightening in a production iteration

---

## GitOps & Kubernetes layer

Repo: `argocd-apps-incode`. **App-of-Apps** pattern: a root Helm chart (`argocd-apps/`) renders one ArgoCD `Application` per component, each pointing at its own chart directory in the same repo with `automated: {prune, selfHeal}` sync policies (server-side apply where a controller mutates its own objects, e.g. webhook CA bundles).

### Bootstrap order (sync waves)

| Wave | Component | Why here |
|---|---|---|
| — | Karpenter | Installed manually via Helm before ArgoCD exists — needs to provision nodes for everything else |
| 1 | ArgoCD (self-managed) | Installed via Helm once, then manages itself via GitOps thereafter |
| 2 | AWS Load Balancer Controller, metrics-server | LB controller installs the `TargetGroupBinding` CRD Traefik needs |
| 3 | Traefik, VictoriaMetrics stack | Traefik needs the LB controller CRD; Grafana's route needs Traefik's CRDs |
| 4 | Fluent Bit | Independent of the above |
| 5 | incode-backend, incode-frontend | Need Traefik for routing and Fluent Bit for log shipping |

Secrets (DB URL, Grafana admin credentials) are **not** stored in git — created manually as plain Kubernetes Secrets per the bootstrap README, or provisioned by Terraform and wired in via IRSA annotations. No Vault/External-Secrets/Sealed-Secrets tooling is in use yet.

### Node provisioning — Karpenter

Two NodePools, both spot+on-demand, amd64/linux, 14-day expiry:
- **`general`** — application workloads (incode-backend, incode-frontend schedule here by default)
- **`system`** — tainted (`dedicated=system:NoSchedule`); all platform components (ArgoCD, Traefik, LB controller, metrics-server, Fluent Bit, VictoriaMetrics/Grafana/Alertmanager) carry the matching toleration

### Ingress path

- **AWS Load Balancer Controller** doesn't provision the ALB (Terraform does) — its only job here is the `TargetGroupBinding` CRD, registering Traefik pod IPs into the pre-existing Target Group.
- **Traefik** runs as a DaemonSet (one pod per node), `ClusterIP` service (no public LoadBalancer Service — exposure is entirely via the TargetGroupBinding). Routing is via `IngressRoute` CRDs, host-matched on `*.crashbackloop.org`.
- The backend has **no IngressRoute of its own** — it's only reachable in-cluster. The frontend's nginx reverse-proxies `/api/` requests to `incode-backend.incode-backend.svc.cluster.local:8000` directly, so the backend is never exposed publicly, by design.

### CI/CD integration

ArgoCD RBAC grants a scoped `github-actions` account get/sync permissions on the `incode-backend` and `incode-frontend` Applications only (everything else defaults to read-only). GitHub Actions authenticates with an ArgoCD API token and triggers syncs directly — see [CI/CD](#cicd) below.

---

## Applications

Both apps are forks of the RealWorld ("Conduit") reference implementation, repurposed as the deployable workload for this assignment. Application logic itself is generic RealWorld (blog/article CRUD, auth, comments); the Incode-specific work is entirely in the Docker/CI/deployment wiring.

### Backend — `realworld-django-ninja-incode`

- Django + Django Ninja REST API, JWT auth, PostgreSQL via `DATABASE_URL`
- Two-stage Docker build (`python:3.12-slim-bookworm`), runs as non-root user, served by Gunicorn (3 workers) on `:8000`
- Config entirely via environment variables (`DEBUG`, `ALLOWED_HOSTS`, `DATABASE_URL`) — the app hard-fails at startup if `DATABASE_URL` is unset outside `DEBUG` mode
- DB schema migrations run as an ArgoCD **PreSync hook Job** (`incode-backend-migrate`) ahead of each deployment, rather than at container start

### Frontend — `vue3-realworld-example-app-incode`

- Vue 3 + TypeScript SPA (Vite, Pinia, Vue Router with hash history)
- Two-stage Docker build: `node:20-alpine` builds static assets, served by `nginx-unprivileged:1.27-alpine` on `:8080`
- nginx handles SPA fallback routing (`try_files ... /index.html`) and reverse-proxies `/api/` to the backend's internal ClusterIP service — the browser only ever talks to the frontend's own origin
- `VITE_API_HOST` is a build-time ARG, defaulting to empty (same-origin relative `/api` calls), avoiding a rebuild per environment for the common case

### Known application-level gaps (carried over from the RealWorld base, not hardened for production)

- Neither app exposes a dedicated `/health` or `/ready` endpoint
- Backend has a hardcoded `SECRET_KEY` and `CORS_ORIGIN_ALLOW_ALL = True`; `ALLOWED_HOSTS` is set to `*` in the Helm values

---

## CI/CD

Both application repos share an identical two-workflow GitHub Actions pattern.

### `ci.yml`

Triggered on push to the main branch and on all pull requests. Parallel jobs:
- **Backend**: lint/format (`ruff`), type-check (`ty`), tests (`manage.py test` against in-memory SQLite with a fast password hasher)
- **Frontend**: lint (ESLint) + type-check (`vue-tsc`), unit tests (Vitest, coverage uploaded to Codecov), E2E tests (Playwright against a built preview server)

### `deploy.yml`

Triggered by `workflow_run` after `CI` succeeds on the main branch, or manually via `workflow_dispatch` (environment selector, currently only `dev`). Guarded so it never deploys a failed CI run.

1. Check out the exact commit CI validated
2. Compute the short git SHA as the image tag
3. Authenticate to AWS, log in to ECR
4. Build and push the image to ECR (`<account>.dkr.ecr.us-east-1.amazonaws.com/incode-{backend,frontend}:<sha>`)
5. Clone `argocd-apps-incode` over SSH (deploy key)
6. Patch `image.tag` in `<app>/<env>.values.yaml` with `yq`
7. Commit and push that change to the GitOps repo
8. Install the `argocd` CLI, run `argocd app sync <app> --prune --timeout 300` and wait for the app to report healthy

This is a **push-then-sync GitOps pattern**: CI writes the desired state to git (the source of truth), then explicitly triggers and waits on the ArgoCD reconciliation rather than relying solely on ArgoCD's own polling interval.

---

## Monitoring, logging & observability

### Metrics — VictoriaMetrics

Deployed via the `victoria-metrics-k8s-stack` umbrella chart (VictoriaMetrics' kube-prometheus-stack equivalent) into the `monitoring` namespace:
- `vmagent` scrapes in-cluster targets (kubelet/cAdvisor/kube-apiserver/kube-state-metrics + any ServiceMonitors)
- `vmsingle` stores metrics on a dedicated `gp2-csi` EBS-backed StorageClass, 7-day retention, 10 Gi
- `vmalert` evaluates alerting rules and forwards to `vmalertmanager`
- **Grafana** ships as part of the stack, exposed at `dev-grafana.crashbackloop.org` via a Traefik `IngressRoute`; admin credentials come from a manually pre-created Secret; no dashboard persistence (PVC disabled)

### Alerting

Four custom `VMRule` alerts (`team: incode`, `severity: critical`): `PodOOMKilled`, `PodCrashLooping`, `JobFailed`, `PodUnschedulable`.

Alertmanager routes matching alerts to an **AWS SNS** receiver (SigV4-authenticated via a dedicated IRSA service account, no static credentials) → SNS topic `incode-cluster-dev-alertmanager` → email subscription. Non-matching alerts are routed to a `blackhole` receiver and dropped. No Slack integration.

### Logging — Fluent Bit

`aws-for-fluent-bit` DaemonSet (runs on every node, including tainted system nodes) ships container logs to **CloudWatch Logs** only — one log group per namespace (`/eks/incode-cluster-dev/<namespace>`), 30-day retention, auto-created groups. A custom filter pipeline strips high-cardinality Kubernetes metadata (pod IPs, docker IDs, all labels) before shipping, to control log volume/cost. Logs and metrics are on entirely separate paths — there is no log-based metrics bridge (no Loki/VictoriaLogs).

### IRSA-based AWS access

Every controller that talks to AWS (Load Balancer Controller, Fluent Bit, Alertmanager) uses a dedicated IAM Role for Service Account, trust-scoped to its exact namespace/service-account pair — no static AWS credentials in the cluster, aside from the CI-only `github-action-service-account` IAM user used for ECR pushes.

---

## Known lower-environment trade-offs

Documented here for transparency — these are deliberate cost/complexity trade-offs for a single `dev` environment, called out inline in the Terraform, not oversights:

- Single NAT gateway (no per-AZ HA)
- RDS single-AZ (`multi_az = false`)
- No secret-management tooling (Vault/External-Secrets) — Kubernetes Secrets created manually
- `github-action-service-account` IAM user carries a broader `iam:PassRole` grant than strictly required
- No HPA/autoscaling configured on the two applications (fixed `replicaCount: 1`)
