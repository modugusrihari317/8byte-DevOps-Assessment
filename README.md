# 8byte DevOps Assignment

A production-grade, end-to-end DevOps implementation covering infrastructure provisioning, CI/CD automation, monitoring, and logging on AWS.

---

## Table of Contents

1. [Architecture Overview](#architecture-overview)
2. [Repository Structure](#repository-structure)
3. [Part 1 — Infrastructure Provisioning](#part-1--infrastructure-provisioning)
4. [Part 2 — CI/CD Pipeline](#part-2--cicd-pipeline)
5. [Part 3 — Monitoring & Logging](#part-3--monitoring--logging)
6. [Part 4 — Security & Best Practices](#part-4--security--best-practices)
7. [Local Development Setup](#local-development-setup)
8. [GitHub Secrets Setup](#github-secrets-setup)
9. [Architecture Decisions](#architecture-decisions)
10. [Cost Optimization](#cost-optimization)
11. [Challenges & Resolutions](#challenges--resolutions)

---

## Architecture Overview

```
Internet
    │
    ▼
[Application Load Balancer]  ← Public Subnets (AZ-a, AZ-b)
    │  HTTPS (443) / HTTP→HTTPS redirect
    ▼
[Auto Scaling Group]         ← Private Subnets (AZ-a, AZ-b)
  EC2 t3.micro × 2
  Node.js App on port 3000
    │
    ▼
[RDS PostgreSQL 15]          ← Private Subnets (Multi-AZ in prod)
    │
    ▼
[AWS Secrets Manager]        ← DB credentials (no plaintext secrets)

Monitoring Stack (Docker Compose on separate EC2 or locally):
  Prometheus → scrapes app + node-exporter + postgres-exporter
  Grafana    → dashboards (Infrastructure + Application)
  Loki       → log aggregation
  Promtail   → log shipping from containers/system
```

---

## Repository Structure

```
.
├── terraform/                     # Part 1: Infrastructure
│   ├── main.tf                    # Root module — wires all submodules
│   ├── variables.tf               # All configurable parameters
│   ├── outputs.tf                 # Key resource outputs
│   ├── environments/
│   │   └── staging/terraform.tfvars
│   └── modules/
│       ├── vpc/                   # VPC, subnets, NAT, flow logs
│       ├── security-groups/       # ALB / App / RDS SGs
│       ├── alb/                   # ALB, target group, listeners, S3 access logs
│       ├── ec2/                   # Launch template, ASG, IAM role
│       └── rds/                   # PostgreSQL, Secrets Manager, param group
│
├── app/                           # Sample Node.js application
│   ├── index.js                   # Express app + Prometheus metrics
│   ├── Dockerfile                 # Multi-stage, non-root image
│   ├── package.json
│   └── __tests__/unit/            # Jest unit tests
│
├── .github/workflows/
│   ├── ci-cd.yml                  # Part 2: Full CI/CD pipeline
│   └── terraform.yml              # Terraform plan/apply automation
│
├── monitoring/                    # Part 3: Monitoring & Logging
│   ├── prometheus/
│   │   ├── prometheus.yml         # Scrape config (app, node, postgres, blackbox)
│   │   └── alerts/alerts.yml      # Alert rules (infra + app + DB)
│   ├── grafana/
│   │   ├── grafana-datasources.yml
│   │   └── dashboards/
│   │       ├── infrastructure.json  # CPU, memory, disk, network
│   │       └── application.json     # Request rate, error rate, latency, DB
│   └── promtail-config.yml        # Log shipping to Loki
│
├── scripts/
│   ├── bootstrap-state.sh         # One-time S3+DynamoDB state setup
│   └── init.sql                   # DB schema + seed data
│
├── docker-compose.yml             # Full local stack (app+db+monitoring)
└── README.md
```

---

## Part 1 — Infrastructure Provisioning

### Prerequisites

- AWS CLI configured (`aws configure`)
- Terraform >= 1.5.0
- An AWS account with sufficient IAM permissions

### Step 1: Bootstrap Remote State (run once)

```bash
chmod +x scripts/bootstrap-state.sh
AWS_REGION=ap-south-1 ./scripts/bootstrap-state.sh
```

This creates:
- S3 bucket `8byte-terraform-state` (versioned, encrypted, private)
- DynamoDB table `terraform-state-lock` (prevents concurrent apply)

### Step 2: Provision Staging Infrastructure

```bash
cd terraform

# Initialise providers and backend
terraform init

# Preview changes
terraform plan -var-file="environments/staging/terraform.tfvars" -var="environment=staging"

# Apply
terraform apply -var-file="environments/staging/terraform.tfvars" -var="environment=staging"
```

### Step 3: View Outputs

```bash
terraform output alb_dns_name
terraform output asg_name
```

### What Gets Created

| Resource | Details |
|---|---|
| VPC | `/16` CIDR, 2 public + 2 private subnets across 2 AZs |
| NAT Gateways | One per AZ for high availability |
| ALB | Internet-facing, HTTP→HTTPS redirect, access logs to S3 |
| Auto Scaling Group | t3.micro × 2, target tracking CPU 70% |
| RDS PostgreSQL 15 | Private subnet, encrypted, Performance Insights enabled |
| Secrets Manager | DB password auto-generated, rotatable |
| VPC Flow Logs | All traffic logged to CloudWatch Logs |
| IAM Role (EC2) | SSM access + Secrets Manager read (least privilege) |

---

## Part 2 — CI/CD Pipeline

### Pipeline Stages

```
PR opened
  └─► [test] Unit + Integration tests + npm audit
        └─► [security-scan] Trivy filesystem scan
              │
              │  (merge to main only)
              ▼
          [build-push] Build Docker image → push to ECR
                └─► Trivy image scan
                      └─► [deploy-staging] ASG instance refresh → smoke test
                            └─► [deploy-production] ← MANUAL APPROVAL REQUIRED
```

### Setting Up Manual Approval

In your GitHub repository:
1. Go to **Settings → Environments → New environment** → name it `production`
2. Enable **Required reviewers**, add yourself or your team
3. Every production deploy will pause and wait for approval

### Pipeline Features

- **Test isolation**: Postgres service container spun up in CI — no mocks needed for integration tests
- **Caching**: `npm ci` cache + Docker layer cache via GitHub Actions cache
- **Security**: Trivy scans both filesystem (dependencies) and the built container image; results uploaded to GitHub Security tab
- **Rolling deploys**: ASG instance refresh with `MinHealthyPercentage=50` (staging) / `75` (prod)
- **Slack notifications**: Success and failure alerts with direct link to the run

---

## Part 3 — Monitoring & Logging

### Dashboards

**Dashboard 1 — Infrastructure Overview** (`monitoring/grafana/dashboards/infrastructure.json`)
- CPU usage % per instance
- Memory usage % per instance  
- Disk usage % (gauge)
- Network I/O (RX/TX bytes/s)

**Dashboard 2 — Application Metrics** (`monitoring/grafana/dashboards/application.json`)
- Request rate (req/s)
- Error rate % (5xx) — stat with colour thresholds
- Response latency p50 / p95 / p99
- Requests by status code
- Active DB connections
- DB I/O wait time

### Alert Rules

| Alert | Threshold | Severity |
|---|---|---|
| HighCPUUsage | > 85% for 5m | warning |
| CriticalCPUUsage | > 95% for 2m | critical |
| HighMemoryUsage | > 85% for 5m | warning |
| DiskSpaceLow | < 15% free for 5m | warning |
| InstanceDown | target unreachable 2m | critical |
| HighErrorRate | 5xx > 5% for 3m | critical |
| SlowResponses | p99 > 2s for 5m | warning |
| AppDown | health check failing 1m | critical |
| PostgresTooManyConnections | > 80 connections | warning |

### Centralized Logging

| Layer | Tool | What's Captured |
|---|---|---|
| Application logs | Promtail → Loki | Container stdout/stderr (JSON) |
| System logs | Promtail → Loki | `/var/log/*.log` |
| Access logs | Promtail → Loki | Nginx/ALB access logs |
| VPC traffic | VPC Flow Logs → CloudWatch | All TCP/UDP flows |
| DB queries | RDS CloudWatch | Slow queries > 1s, connections |

---

## Part 4 — Security & Best Practices

### Secret Management

Secrets are managed via **AWS Secrets Manager**:

- RDS password is **auto-generated** by Terraform using `random_password` (24 chars, special chars)
- Stored as JSON in Secrets Manager with host, port, username, password, dbname
- EC2 instances pull the secret at boot via IAM role — **no plaintext credentials anywhere**
- GitHub Actions secrets store only `AWS_ACCESS_KEY_ID` and `AWS_SECRET_ACCESS_KEY`

```bash
# To retrieve the DB password manually (for debugging):
aws secretsmanager get-secret-value \
  --secret-id "8byte/staging/db-credentials" \
  --query SecretString --output text | jq .
```

### Backup Strategy

**RDS Automated Backups:**
- Staging: 1-day retention (cost saving)
- Production: 7-day retention
- Daily backup window: 03:00–04:00 UTC
- Point-in-time recovery enabled

**Manual snapshot before major changes:**
```bash
aws rds create-db-snapshot \
  --db-instance-identifier 8byte-staging-postgres \
  --db-snapshot-identifier manual-pre-migration-$(date +%Y%m%d)
```

**S3 ALB access logs:** Lifecycle rule expires logs after 30 days.

### Security Hardening

- **Network segmentation**: App servers in private subnets, only reachable via ALB on port 3000
- **RDS isolation**: Only accessible from App security group on port 5432
- **EBS encryption**: All EC2 volumes encrypted at rest
- **RDS encryption**: Storage encrypted with AWS-managed KMS key
- **Non-root Docker**: App container runs as `appuser` (UID non-zero)
- **IAM least privilege**: EC2 role only has SSM + Secrets Manager read for its own secret
- **VPC Flow Logs**: Full traffic audit trail
- **ALB deletion protection**: Enabled in production
- **TLS 1.3**: ALB configured with `ELBSecurityPolicy-TLS13-1-2-2021-06`

---

## Local Development Setup

```bash
# 1. Clone the repo
git clone https://github.com/<your-org>/8byte-devops.git
cd 8byte-devops

# 2. Start the full stack (app + postgres + prometheus + grafana + loki)
docker compose up -d

# 3. Access services
#   App:        http://localhost:3000
#   Prometheus: http://localhost:9090
#   Grafana:    http://localhost:3001  (admin / admin123)
#   Loki:       http://localhost:3100

# 4. Run tests locally
cd app
npm install
npm test
```

---

## GitHub Secrets Setup

Add these secrets in **Settings → Secrets and variables → Actions**:

| Secret | Description |
|---|---|
| `AWS_ACCESS_KEY_ID` | IAM user key for staging deploys |
| `AWS_SECRET_ACCESS_KEY` | IAM user secret for staging deploys |
| `AWS_ACCESS_KEY_ID_PROD` | IAM user key for production |
| `AWS_SECRET_ACCESS_KEY_PROD` | IAM user secret for production |
| `SLACK_WEBHOOK_URL` | Incoming webhook for deploy notifications |

---

## Architecture Decisions

### Why EC2 + ASG instead of ECS/EKS?

For a startup context, EC2 Auto Scaling Groups offer simpler operational overhead than EKS (no control plane cost, no CNI complexity). ECS Fargate would be the natural next step as the team grows — the Dockerfile is already Fargate-compatible.

### Why ALB access logs to S3 instead of CloudWatch?

S3 is ~15× cheaper than CloudWatch Logs for high-volume access logs. A lifecycle rule expires them after 30 days to prevent unbounded cost.

### Why Loki over ELK?

Loki has significantly lower resource consumption than Elasticsearch (no index overhead). For a startup, Loki + Promtail + Grafana integrates natively with the existing Prometheus/Grafana stack, reducing operational surface area.

### Why separate ASG credentials for prod?

Staging and production use separate IAM users/roles with separate GitHub environment secrets. This prevents a compromised staging key from affecting production.

---

## Cost Optimization

| Area | Measure | Estimated Saving |
|---|---|---|
| EC2 | t3.micro (burstable), right-sized for demo | Baseline |
| RDS | db.t3.micro, multi-AZ only in prod | ~50% vs always multi-AZ |
| Storage | gp3 (cheaper + faster than gp2) | ~20% on EBS |
| NAT Gateway | One per AZ (HA) — reduce to 1 for dev | ~50% NAT cost |
| ALB Logs | S3 + 30-day lifecycle | ~15× vs CloudWatch |
| RDS | Auto storage scaling (no over-provisioning) | Variable |
| Backups | 1-day retention in staging | Minimal snapshot cost |

---

## Challenges & Resolutions

### Challenge 1: Terraform state corruption risk on concurrent runs

**Problem**: Two developers running `terraform apply` simultaneously can corrupt state.

**Resolution**: Added DynamoDB state locking table (`terraform-state-lock`). Terraform acquires a lock before any plan/apply and releases it on completion. The bootstrap script creates this as the very first step.

### Challenge 2: DB password in Terraform state

**Problem**: `random_password` result is stored in the Terraform state file, which is sensitive.

**Resolution**: The state is stored in an encrypted S3 bucket (AES-256). Additionally, the `sensitive = true` flag on the output prevents it from being printed in logs. For extra hardening, RDS password rotation can be enabled via Secrets Manager.

### Challenge 3: EC2 instances need DB credentials without hardcoding

**Problem**: Passing DB passwords as environment variables or hardcoding them in AMIs is a security anti-pattern.

**Resolution**: The EC2 IAM role has a scoped `secretsmanager:GetSecretValue` permission for only the specific secret ARN. The userdata script fetches the secret at boot time using `aws secretsmanager get-secret-value`, writes it to `/etc/app.env` with `600` permissions, and the app reads it from there.

### Challenge 4: Zero-downtime deployments with ASG

**Problem**: Replacing instances in an ASG during a deployment can cause dropped connections.

**Resolution**: Used ASG Instance Refresh with `MinHealthyPercentage=50` (staging) and `75` (prod). The ALB health check waits for instances to pass `/health` before routing traffic. The `InstanceWarmup` period (60–120s) prevents new instances from being counted as healthy prematurely.

### Challenge 5: CI pipeline security without slowing down PRs

**Problem**: Full Trivy scans can take 2–3 minutes, making PR feedback slow.

**Resolution**: Split jobs — `test` runs fast unit/integration tests in parallel, `security-scan` runs after tests pass. The Docker image scan runs only on merge to main (after build), not on every PR. This keeps PR feedback under 2 minutes while full security scanning runs on every merge.
