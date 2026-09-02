# XanePay — AWS Deployment & Security Guide
## Production-Ready Infrastructure, Security Hardening & Operational Runbook

> **Document Status:** Authoritative reference for all deployment, infrastructure, and security decisions.
> **Last Updated:** 2026-08-30
> **Scope:** Full XanePay stack — Engine (Treasury Engine, port 3001), Backend (XaneApp, port 8080), Frontend (React Native / Web), and all supporting infrastructure.

---

## Table of Contents

1. [Application Inventory — What Gets Deployed](#1-application-inventory)
2. [AWS Architecture — Target State](#2-aws-architecture)
3. [Network Architecture — VPC & Security Groups](#3-network-architecture)
4. [Compute — ECS Fargate (Containers)](#4-compute)
5. [Database — RDS PostgreSQL](#5-database)
6. [Cache/Queue — ElastiCache Redis](#6-cachequeue)
7. [Secrets Management — AWS Secrets Manager + KMS](#7-secrets-management)
8. [Container Registry — ECR](#8-container-registry)
9. [Load Balancing & TLS — ALB + ACM](#9-load-balancing)
10. [DNS — Route 53](#10-dns)
11. [CI/CD Pipeline — GitHub Actions → ECR → ECS](#11-cicd-pipeline)
12. [Monitoring, Logging & Alerting](#12-monitoring)
13. [Security Hardening Checklist](#13-security-hardening)
14. [Wallet & Key Custody on AWS](#14-wallet-key-custody)
15. [Backup & Disaster Recovery](#15-backup-disaster-recovery)
16. [Compliance & Regulatory](#16-compliance)
17. [Cost Estimation](#17-cost-estimation)
18. [Deployment Runbook — Step by Step](#18-deployment-runbook)
19. [Incident Response Playbook](#19-incident-response)

---

## 1. Application Inventory

Every deployable component and its role. Nothing gets deployed unless it is in this table.

| Component | Language | Port | Repo Location | Docker Image | Purpose |
|---|---|---|---|---|---|
| **Treasury Engine** | TypeScript (strict) | 3001 | `XaneApp/engine/` | `xanepay-engine` | Fiat↔crypto orchestration, double-entry ledger, provider routing |
| **XaneApp Backend** | JavaScript (Node.js) | 8080 | `XaneApp/backend/` | `xanepay-backend` | User auth, wallet management, Firebase integration |
| **XaneApp Frontend** | React Native / Web | 443 (CDN) | `XaneApp/frontend/` | N/A (static) | Mobile/web client — deployed to CloudFront + S3 |
| **PostgreSQL 16** | — | 5432 | — | AWS RDS | Engine ledger (ACID), audit logs, conversions, executions |
| **Redis 7** | — | 6379 | — | AWS ElastiCache | Rate caching, BullMQ jobs, HMAC dedup, distributed locks |
| **Cloudflare Tunnel** | — | — | — | Cloudflare | Dev-only webhook forwarding (NOT for production) |

### What Else Is Required (Not Code — Infrastructure)

| Requirement | AWS Service | Why |
|---|---|---|
| TLS certificates | ACM (free) | All traffic HTTPS — no exceptions |
| DNS management | Route 53 | `api.xanepay.com`, `engine.xanepay.com` |
| Secrets vault | Secrets Manager + KMS | Provider API keys, HMAC secrets, DB credentials, wallet keys |
| Container registry | ECR | Private Docker images |
| Log aggregation | CloudWatch Logs | Structured JSON logs from all services |
| Alerting | CloudWatch Alarms + SNS | PagerDuty/Slack for critical events |
| Static hosting | S3 + CloudFront | Web frontend (if applicable) |
| WAF | AWS WAF | DDoS protection, rate limiting at edge |
| VPN | Client VPN or SSM | Admin access — no SSH, no open ports |

---

## 2. AWS Architecture — Target State

```
                                    INTERNET
                                       │
                              ┌────────▼────────┐
                              │   CloudFlare     │ ← (optional: DDoS + edge caching)
                              │   or AWS WAF     │
                              └────────┬────────┘
                                       │
                              ┌────────▼────────┐
                              │    Route 53      │ ← DNS: api.xanepay.com
                              └────────┬────────┘
                                       │
                              ┌────────▼────────┐
                              │  Application     │ ← TLS termination (ACM cert)
                              │  Load Balancer   │   Path routing:
                              │  (ALB)           │   /api/v1/engine/* → Engine TG
                              │                  │   /api/v1/*        → Backend TG
                              └─┬──────────────┬─┘
                                │              │
              ┌─────────────────▼──┐    ┌──────▼─────────────────┐
              │    ECS Fargate      │    │    ECS Fargate          │
              │   Engine Service    │    │   Backend Service       │
              │   (2–4 tasks)       │    │   (2–4 tasks)           │
              │   Port 3001         │    │   Port 8080             │
              └────────┬───────────┘    └────────┬───────────────┘
                       │                         │
          ┌────────────▼─────────────────────────▼──────────────┐
          │                  PRIVATE SUBNETS                     │
          │                                                      │
          │  ┌──────────────────┐    ┌──────────────────────┐   │
          │  │  RDS PostgreSQL  │    │  ElastiCache Redis   │   │
          │  │  16 (Multi-AZ)   │    │  7 (Cluster Mode)    │   │
          │  │  Port 5432       │    │  Port 6379           │   │
          │  │  Encrypted       │    │  Encrypted + Auth    │   │
          │  └──────────────────┘    └──────────────────────┘   │
          │                                                      │
          │  ┌──────────────────┐    ┌──────────────────────┐   │
          │  │  Secrets Manager │    │  KMS (CMK)           │   │
          │  │  (all secrets)   │    │  (envelope encryption│   │
          │  └──────────────────┘    │   for wallet keys)   │   │
          │                          └──────────────────────┘   │
          └──────────────────────────────────────────────────────┘
```

### Multi-AZ Design

Everything runs across **at least 2 Availability Zones**. A single AZ failure must not take the system down.

| Component | AZ Strategy |
|---|---|
| ALB | Spans all AZs in the VPC |
| ECS Tasks | Spread across 2+ AZs (placement strategy) |
| RDS | Multi-AZ standby (automatic failover) |
| ElastiCache | Multi-AZ with auto-failover (replication group) |

---

## 3. Network Architecture — VPC & Security Groups

### VPC Design

```
VPC: 10.0.0.0/16 (65,536 addresses)

├── Public Subnets (ALB only — NO compute here)
│   ├── 10.0.1.0/24   (AZ-a)
│   └── 10.0.2.0/24   (AZ-b)
│
├── Private Subnets (Compute — ECS tasks)
│   ├── 10.0.10.0/24  (AZ-a)
│   └── 10.0.20.0/24  (AZ-b)
│
├── Data Subnets (RDS, ElastiCache — no internet access)
│   ├── 10.0.100.0/24 (AZ-a)
│   └── 10.0.200.0/24 (AZ-b)
│
└── NAT Gateway (1 per AZ — for outbound API calls to providers)
    ├── 10.0.1.0/24 → NAT-a → IGW
    └── 10.0.2.0/24 → NAT-b → IGW
```

### Security Groups — Principle of Least Privilege

> [!CAUTION]
> **NEVER open port 22 (SSH) to 0.0.0.0/0.** Use AWS Systems Manager Session Manager for all server access. There are no SSH keys to steal if SSH doesn't exist.

| Security Group | Inbound | Outbound | Attached To |
|---|---|---|---|
| `sg-alb` | 443 from `0.0.0.0/0` (HTTPS only) | All to VPC | ALB |
| `sg-engine` | 3001 from `sg-alb` only | 5432 to `sg-rds`, 6379 to `sg-redis`, 443 to `0.0.0.0/0` (provider APIs) | Engine ECS tasks |
| `sg-backend` | 8080 from `sg-alb` only | 5432 to `sg-rds`, 6379 to `sg-redis`, 443 to `0.0.0.0/0` | Backend ECS tasks |
| `sg-rds` | 5432 from `sg-engine` + `sg-backend` | None | RDS instance |
| `sg-redis` | 6379 from `sg-engine` + `sg-backend` | None | ElastiCache cluster |

### NACLs (Network ACLs)

| Subnet Tier | Rule |
|---|---|
| Public | Allow 443 in from anywhere, deny all other inbound |
| Private | Allow traffic from public subnets on service ports only |
| Data | Allow traffic from private subnets on DB/cache ports only. **Deny ALL internet.** |

### VPC Endpoints (Private Connectivity — No Internet)

These keep AWS service traffic within the VPC — never crossing the public internet.

| Service | Endpoint Type | Why |
|---|---|---|
| ECR (dkr + api) | Interface | Pull images without NAT |
| Secrets Manager | Interface | Fetch secrets without NAT |
| CloudWatch Logs | Interface | Ship logs without NAT |
| S3 | Gateway | ECR layer storage |
| KMS | Interface | Decrypt wallet keys without NAT |
| STS | Interface | IAM role assumption |

---

## 4. Compute — ECS Fargate (Containers)

### Why Fargate (Not EC2, Not Lambda)

| Option | Verdict | Reason |
|---|---|---|
| **ECS Fargate** | ✅ **Chosen** | No server management, automatic scaling, security patching handled by AWS, pay-per-use |
| EC2 | ❌ | Server patching burden, SSH key management, over-provisioning waste |
| Lambda | ❌ | Cold starts unacceptable for financial APIs, 15-min timeout kills BullMQ workers, no persistent Redis connections |
| EKS | ❌ Overkill for now | Kubernetes overhead not justified at current scale |

### Task Definitions

#### Engine Task

```json
{
  "family": "xanepay-engine",
  "networkMode": "awsvpc",
  "requiresCompatibilities": ["FARGATE"],
  "cpu": "512",
  "memory": "1024",
  "executionRoleArn": "arn:aws:iam::role/ecsTaskExecutionRole",
  "taskRoleArn": "arn:aws:iam::role/xanepay-engine-task-role",
  "containerDefinitions": [
    {
      "name": "engine",
      "image": "<account>.dkr.ecr.<region>.amazonaws.com/xanepay-engine:latest",
      "portMappings": [{ "containerPort": 3001 }],
      "environment": [
        { "name": "NODE_ENV", "value": "production" },
        { "name": "PORT", "value": "3001" }
      ],
      "secrets": [
        { "name": "DATABASE_URL", "valueFrom": "arn:aws:secretsmanager:...:xanepay/engine/database-url" },
        { "name": "REDIS_URL", "valueFrom": "arn:aws:secretsmanager:...:xanepay/engine/redis-url" },
        { "name": "HMAC_SECRET", "valueFrom": "arn:aws:secretsmanager:...:xanepay/engine/hmac-secret" },
        { "name": "ADMIN_API_KEY", "valueFrom": "arn:aws:secretsmanager:...:xanepay/engine/admin-api-key" },
        { "name": "ONSWITCH_SERVICE_KEY", "valueFrom": "arn:aws:secretsmanager:...:xanepay/engine/onswitch-key" },
        { "name": "SENTRY_DSN", "valueFrom": "arn:aws:secretsmanager:...:xanepay/engine/sentry-dsn" }
      ],
      "logConfiguration": {
        "logDriver": "awslogs",
        "options": {
          "awslogs-group": "/ecs/xanepay-engine",
          "awslogs-region": "<region>",
          "awslogs-stream-prefix": "engine"
        }
      },
      "healthCheck": {
        "command": ["CMD-SHELL", "node -e \"require('http').get('http://127.0.0.1:3001/api/v1/health/live',r=>process.exit(r.statusCode===200?0:1)).on('error',()=>process.exit(1))\""],
        "interval": 30,
        "timeout": 5,
        "retries": 3,
        "startPeriod": 15
      }
    }
  ]
}
```

#### Scaling Policy

| Metric | Scale Out | Scale In | Min | Max |
|---|---|---|---|---|
| CPU > 70% for 3 min | +1 task | -1 task (CPU < 30%) | 2 | 6 |
| ALB request count > 500/min | +2 tasks | -1 task | 2 | 6 |

> [!IMPORTANT]
> **BullMQ concurrency is capped at 5.** The engine's own load tests proved throughput *degrades* above this. Scaling means more ECS tasks, NOT raising `BULLMQ_CONCURRENCY` per task.

---

## 5. Database — RDS PostgreSQL

### Instance Configuration

| Parameter | Value | Rationale |
|---|---|---|
| Engine | PostgreSQL 16 | ACID required for financial ledger; matches dev/test |
| Instance class | `db.r6g.large` (2 vCPU, 16 GB) | Ledger is write-heavy; ARM-based for cost efficiency |
| Storage | 100 GB gp3, auto-scaling to 500 GB | gp3 gives 3,000 IOPS baseline + 125 MB/s throughput |
| Multi-AZ | **Yes** (synchronous standby) | Automatic failover — no data loss |
| Encryption at rest | **Yes** — KMS CMK | Non-negotiable for financial data |
| Encryption in transit | **Yes** — `sslmode=verify-full` | Non-negotiable |
| Backup | Automated, 30-day retention | Point-in-time recovery to the second |
| Performance Insights | Enabled | Query-level visibility |
| Deletion protection | **Enabled** | Prevents accidental `terraform destroy` |
| Public accessibility | **NO** | Data subnet — no internet route |

### Database Users & Roles

> [!CAUTION]
> **The application MUST NOT connect as the `postgres` superuser.** Create dedicated roles with minimum required privileges.

```sql
-- Engine application role (read/write, but CANNOT modify schema)
CREATE ROLE xanepay_engine LOGIN PASSWORD '<from-secrets-manager>';
GRANT CONNECT ON DATABASE xanepay_engine TO xanepay_engine;
GRANT USAGE ON SCHEMA public TO xanepay_engine;
GRANT SELECT, INSERT ON ALL TABLES IN SCHEMA public TO xanepay_engine;
-- Explicitly allow UPDATE only on tables that need it (NOT ledger tables)
GRANT UPDATE ON conversions, executions, quotes, providers, fee_config TO xanepay_engine;
-- REVOKE dangerous operations from ledger tables (defense in depth alongside triggers)
REVOKE UPDATE, DELETE ON ledger_transactions, ledger_operations, audit_logs FROM xanepay_engine;

-- Migration role (schema changes only — used by CI/CD, never by runtime)
CREATE ROLE xanepay_migrator LOGIN PASSWORD '<from-secrets-manager>';
GRANT ALL PRIVILEGES ON DATABASE xanepay_engine TO xanepay_migrator;

-- Read-only role (for dashboards, reporting, analytics)
CREATE ROLE xanepay_readonly LOGIN PASSWORD '<from-secrets-manager>';
GRANT CONNECT ON DATABASE xanepay_engine TO xanepay_readonly;
GRANT SELECT ON ALL TABLES IN SCHEMA public TO xanepay_readonly;
```

### PostgreSQL Parameters (Parameter Group)

```
# Connection security
ssl = on
ssl_min_protocol_version = TLSv1.3
password_encryption = scram-sha-256

# Logging (for audit trail)
log_statement = 'mod'                    # Log all INSERT/UPDATE/DELETE
log_connections = on
log_disconnections = on
log_line_prefix = '%t [%p] %q%u@%d '

# Performance for write-heavy ledger
shared_buffers = 4GB                     # 25% of RAM
effective_cache_size = 12GB              # 75% of RAM
work_mem = 64MB
maintenance_work_mem = 1GB
max_connections = 100
```

---

## 6. Cache/Queue — ElastiCache Redis

### Configuration

| Parameter | Value | Rationale |
|---|---|---|
| Engine | Redis 7 | Matches dev/test; required by BullMQ |
| Node type | `cache.r6g.large` (2 vCPU, 12.93 GB) | BullMQ needs memory headroom |
| Cluster mode | **Disabled** (single shard + replica) | BullMQ does not support cluster mode |
| Multi-AZ | **Yes** (auto-failover) | Prevents queue loss on AZ failure |
| Encryption at rest | **Yes** — KMS | Non-negotiable |
| Encryption in transit | **Yes** — TLS | Non-negotiable |
| AUTH token | **Yes** — from Secrets Manager | Non-negotiable |
| Maxmemory policy | **`noeviction`** | BullMQ stores jobs — eviction silently drops accepted payouts |
| Backup | Daily snapshot, 7-day retention | Recover queued jobs if needed |

> [!CAUTION]
> **ElastiCache defaults to `volatile-lru`.** You MUST explicitly set `noeviction` in the parameter group. The engine's startup guard (`redis-eviction-guard.ts`) will refuse to boot if this is wrong — but set it correctly so the guard never fires.

### Redis Parameter Group

```
maxmemory-policy = noeviction
timeout = 0
tcp-keepalive = 300
notify-keyspace-events = ""
```

---

## 7. Secrets Management — AWS Secrets Manager + KMS

### Secret Inventory

> [!CAUTION]
> **Provider secrets (API keys, webhook secrets) live ONLY in environment variables sourced from Secrets Manager. They are NEVER stored in the database.** This is enforced architecturally — `ProviderRepository` drops any secret on write and returns NULL on read. See [CLAUDE.md Rule 4](file:///c:/Projects/XanePay/CLAUDE.md).

| Secret Path | Contents | Rotation |
|---|---|---|
| `xanepay/engine/database-url` | `postgresql://xanepay_engine:<password>@<host>:5432/xanepay_engine?sslmode=verify-full` | 90-day auto-rotation |
| `xanepay/engine/redis-url` | `rediss://:<auth-token>@<host>:6379` | 90-day auto-rotation |
| `xanepay/engine/hmac-secret` | HMAC-SHA256 signing key (≥32 chars) | 90-day manual rotation with dual-key overlap |
| `xanepay/engine/admin-api-key` | Admin API authentication key (≥32 chars) | 90-day manual rotation |
| `xanepay/engine/onswitch-key` | OnSwitch provider service key | Per provider contract |
| `xanepay/engine/sentry-dsn` | Sentry error tracking DSN | Never (static) |
| `xanepay/engine/wallet-seed` | HD wallet seed phrase (if self-managed) | **NEVER** (backed up to cold storage) |
| `xanepay/backend/firebase-service-account` | Firebase Admin SDK credentials | Per Firebase rotation policy |
| `xanepay/backend/paystack-secret-key` | Paystack API secret | Per Paystack contract |
| `xanepay/backend/jwt-secret` | JWT signing secret | 90-day rotation |

### KMS Key Structure

```
CMK: alias/xanepay-master
  ├── Used by: Secrets Manager (automatic envelope encryption)
  ├── Used by: RDS (storage encryption)
  ├── Used by: ElastiCache (at-rest encryption)
  ├── Used by: S3 (log archive encryption)
  └── Used by: ECS task execution role (decrypt secrets at launch)

CMK: alias/xanepay-wallet (SEPARATE KEY — restricted access)
  ├── Used by: Wallet key encryption ONLY
  ├── Key policy: Only the engine task role can decrypt
  └── Key rotation: AWS-managed annual rotation
```

### IAM Policies for Secret Access

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "EngineSecretsRead",
      "Effect": "Allow",
      "Action": [
        "secretsmanager:GetSecretValue"
      ],
      "Resource": [
        "arn:aws:secretsmanager:*:*:secret:xanepay/engine/*"
      ]
    },
    {
      "Sid": "EngineKMSDecrypt",
      "Effect": "Allow",
      "Action": [
        "kms:Decrypt",
        "kms:DescribeKey"
      ],
      "Resource": [
        "arn:aws:kms:*:*:key/<master-key-id>",
        "arn:aws:kms:*:*:key/<wallet-key-id>"
      ]
    }
  ]
}
```

> [!IMPORTANT]
> **The backend service CANNOT access engine secrets, and vice versa.** IAM policies enforce secret isolation per service. A compromised backend cannot read the engine's HMAC key or provider credentials.

---

## 8. Container Registry — ECR

### Repository Structure

| Repository | Lifecycle Policy | Scan on Push |
|---|---|---|
| `xanepay-engine` | Keep last 10 tagged images + last 3 untagged | **Yes** — block deployment on CRITICAL/HIGH CVEs |
| `xanepay-backend` | Keep last 10 tagged images + last 3 untagged | **Yes** |

### Image Tagging Strategy

```
<account>.dkr.ecr.<region>.amazonaws.com/xanepay-engine:<git-sha>
<account>.dkr.ecr.<region>.amazonaws.com/xanepay-engine:latest     (only on main branch)
<account>.dkr.ecr.<region>.amazonaws.com/xanepay-engine:staging    (on staging branch)
```

### ECR Scan Policy

```json
{
  "rules": [
    {
      "rulePriority": 1,
      "description": "Block deployment on CRITICAL vulnerabilities",
      "selection": {
        "tagStatus": "ANY",
        "countType": "sinceImagePushed",
        "countUnit": "days",
        "countNumber": 1
      },
      "action": {
        "type": "expire"
      }
    }
  ]
}
```

---

## 9. Load Balancing & TLS — ALB + ACM

### ALB Configuration

| Parameter | Value |
|---|---|
| Scheme | Internet-facing |
| Listeners | HTTPS:443 only (HTTP:80 → 301 redirect to HTTPS) |
| TLS policy | `ELBSecurityPolicy-TLS13-1-2-2021-06` (TLS 1.3 preferred, 1.2 minimum) |
| Idle timeout | 60s |
| Access logging | Enabled → S3 bucket |
| WAF attached | Yes |

### Listener Rules (Path-Based Routing)

| Priority | Condition | Forward To | Health Check |
|---|---|---|---|
| 1 | Path: `/api/v1/health/*`, `/api/v1/conversions/*`, `/api/v1/webhooks/*`, `/api/v1/admin/*` | Engine Target Group (port 3001) | `GET /api/v1/health/live` |
| 2 | Path: `/api/v1/*` | Backend Target Group (port 8080) | `GET /health` |
| Default | All other | Return 404 | — |

### ACM Certificate

```
Domain: *.xanepay.com
Validation: DNS (Route 53 auto-validation)
Auto-renewal: Yes (ACM handles this)
```

---

## 10. DNS — Route 53

| Record | Type | Target | TTL |
|---|---|---|---|
| `api.xanepay.com` | A (Alias) | ALB | — |
| `api.xanepay.com` | AAAA (Alias) | ALB | — |
| `www.xanepay.com` | A (Alias) | CloudFront distribution | — |
| `xanepay.com` | A (Alias) | CloudFront distribution | — |

### Health Checks

Route 53 health check on `api.xanepay.com/api/v1/health/live` — if down for 3 consecutive checks, alert via SNS.

---

## 11. CI/CD Pipeline — GitHub Actions → ECR → ECS

### Pipeline Architecture

```
┌──────────────┐     ┌──────────────┐     ┌──────────────┐     ┌──────────────┐
│   Git Push    │ ──► │  GitHub      │ ──► │  Build &     │ ──► │  Deploy to   │
│   (engine    │     │  Actions     │     │  Test        │     │  ECS         │
│   branch)    │     │              │     │              │     │              │
└──────────────┘     └──────────────┘     └──────────────┘     └──────────────┘
                                               │
                                    ┌──────────▼──────────┐
                                    │  Quality Gates      │
                                    │  ✅ typecheck       │
                                    │  ✅ lint            │
                                    │  ✅ unit tests      │
                                    │  ✅ integration tests│
                                    │  ✅ security scan   │
                                    │  ✅ ECR image scan  │
                                    │  ✅ ledger/verify   │
                                    └─────────────────────┘
```

### GitHub Actions Workflow

```yaml
# .github/workflows/deploy-engine.yml
name: Deploy Engine

on:
  push:
    branches: [engine]
    paths: ['XaneApp/engine/**']

env:
  AWS_REGION: eu-west-1            # or af-south-1 for Cape Town (closest to Lagos)
  ECR_REPOSITORY: xanepay-engine
  ECS_CLUSTER: xanepay-production
  ECS_SERVICE: xanepay-engine

jobs:
  test:
    runs-on: ubuntu-latest
    services:
      postgres:
        image: postgres:16-alpine
        env:
          POSTGRES_DB: xanepay_engine_test
          POSTGRES_USER: test
          POSTGRES_PASSWORD: test
        ports: ['5433:5432']
        options: --health-cmd pg_isready --health-interval 10s
      redis:
        image: redis:7-alpine
        ports: ['6379:6379']
        options: --health-cmd "redis-cli ping"
    
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with: { node-version: 20 }
      - run: npm ci
        working-directory: XaneApp/engine
      - run: npm run typecheck
        working-directory: XaneApp/engine
      - run: npm run lint
        working-directory: XaneApp/engine
      - run: npm run test
        working-directory: XaneApp/engine
      - run: npm run test:integration
        working-directory: XaneApp/engine
        env:
          USE_TESTCONTAINERS: 'false'
          DATABASE_URL: postgresql://test:test@localhost:5433/xanepay_engine_test
          REDIS_URL: redis://localhost:6379

  deploy:
    needs: test
    runs-on: ubuntu-latest
    permissions:
      id-token: write
      contents: read
    
    steps:
      - uses: actions/checkout@v4
      
      - uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: arn:aws:iam::<account>:role/github-actions-deploy
          aws-region: ${{ env.AWS_REGION }}
      
      - uses: aws-actions/amazon-ecr-login@v2
        id: ecr
      
      - name: Build & push image
        working-directory: XaneApp/engine
        run: |
          IMAGE_TAG=${{ github.sha }}
          docker build -t $ECR_REGISTRY/$ECR_REPOSITORY:$IMAGE_TAG .
          docker build -t $ECR_REGISTRY/$ECR_REPOSITORY:latest .
          docker push $ECR_REGISTRY/$ECR_REPOSITORY:$IMAGE_TAG
          docker push $ECR_REGISTRY/$ECR_REPOSITORY:latest
        env:
          ECR_REGISTRY: ${{ steps.ecr.outputs.registry }}
      
      - name: Run migrations
        run: |
          # Use the builder stage to run migrations
          docker build --target builder -t xanepay-migrate XaneApp/engine/
          docker run --rm \
            -e DATABASE_URL=${{ secrets.PROD_DATABASE_URL }} \
            xanepay-migrate npm run migrate
      
      - name: Deploy to ECS
        run: |
          aws ecs update-service \
            --cluster $ECS_CLUSTER \
            --service $ECS_SERVICE \
            --force-new-deployment
      
      - name: Wait for stable deployment
        run: |
          aws ecs wait services-stable \
            --cluster $ECS_CLUSTER \
            --services $ECS_SERVICE
```

### Deployment Strategy

| Strategy | Configuration |
|---|---|
| Type | **Rolling update** (ECS default) |
| Min healthy | 100% (always keep current capacity running) |
| Max percent | 200% (spin up new tasks before draining old) |
| Deregistration delay | 30s (ALB drains connections gracefully) |
| Circuit breaker | Enabled — auto-rollback on failed deployment |

---

## 12. Monitoring, Logging & Alerting

### Logging Architecture

```
Engine (Pino JSON) ──► CloudWatch Logs ──► CloudWatch Insights (query)
                                       ──► S3 (archive, 1-year retention)
                                       ──► (optional) Datadog/Grafana
```

### Log Retention

| Log Group | Retention | Archive |
|---|---|---|
| `/ecs/xanepay-engine` | 30 days (hot) | S3 Glacier, 7-year retention (financial compliance) |
| `/ecs/xanepay-backend` | 30 days (hot) | S3 Glacier, 7-year retention |
| ALB access logs | 90 days (S3) | S3 Glacier after 90 days |
| RDS audit logs | 30 days (CloudWatch) | S3 Glacier after 30 days |

### CloudWatch Alarms (Critical)

| Alarm | Condition | Action |
|---|---|---|
| **Engine unhealthy** | ALB healthy host count < 2 for 2 min | SNS → PagerDuty (P1) |
| **RDS CPU > 80%** | CPUUtilization > 80% for 5 min | SNS → Slack |
| **RDS storage < 20%** | FreeStorageSpace < 20 GB | SNS → PagerDuty (P2) |
| **Redis memory > 80%** | DatabaseMemoryUsagePercentage > 80% | SNS → PagerDuty (P2) |
| **Redis evictions > 0** | Evictions > 0 | SNS → PagerDuty (P1) — should NEVER happen with noeviction |
| **5xx error rate > 1%** | ALB HTTPCode_Target_5XX_Count > 1% of total | SNS → PagerDuty (P1) |
| **Conversion stuck** | Custom metric: conversions in non-terminal state > 30 min | SNS → PagerDuty (P2) |
| **Ledger drift detected** | Custom metric from `reconcileLedgerBalances()` | SNS → PagerDuty (P1) |
| **Payout velocity breach** | Custom metric: payouts/hour exceeds R17 limit | SNS → PagerDuty (P1) + halt payouts |

### Sentry Integration (Already in Engine)

The engine already has `@sentry/node` as a dependency. Configure in production:

```
SENTRY_DSN=https://<key>@sentry.io/<project>
SENTRY_ENVIRONMENT=production
SENTRY_RELEASE=<git-sha>
```

### Custom Metrics (CloudWatch Embedded Metric Format)

Push these from the engine via structured JSON logs:

```json
{
  "_aws": {
    "Timestamp": 1693407600000,
    "CloudWatchMetrics": [{
      "Namespace": "XanePay/Engine",
      "Dimensions": [["Environment"]],
      "Metrics": [
        { "Name": "ConversionsCompleted", "Unit": "Count" },
        { "Name": "ConversionLatencyMs", "Unit": "Milliseconds" },
        { "Name": "LedgerDriftDetected", "Unit": "Count" },
        { "Name": "PayoutVelocity", "Unit": "Count/Second" },
        { "Name": "ProviderHealthScore", "Unit": "Percent" }
      ]
    }]
  }
}
```

---

## 13. Security Hardening Checklist

> [!IMPORTANT]
> This checklist covers everything needed for a financial-grade deployment. Every item must be ✅ before handling real money.

### 13.1 Network Security

- [ ] VPC with public/private/data subnet tiers
- [ ] No SSH ports open anywhere — use SSM Session Manager
- [ ] Security groups follow least-privilege (Section 3)
- [ ] NACLs on data subnets deny all internet traffic
- [ ] VPC Flow Logs enabled (S3 destination, 7-year retention)
- [ ] NAT Gateway for outbound only — no inbound from internet to private subnets
- [ ] VPC endpoints for all AWS services (no traffic leaves VPC)
- [ ] AWS WAF attached to ALB with:
  - Rate limiting (1,000 requests/5 min per IP)
  - SQL injection rule group
  - Known bad inputs rule group
  - Geo-blocking (allow only relevant countries initially)
  - Bot control (optional)

### 13.2 Identity & Access Management (IAM)

- [ ] Root account has MFA hardware key, no access keys
- [ ] No IAM users — use IAM Identity Center (SSO) for humans
- [ ] ECS tasks use **task roles** (not access keys in env vars)
- [ ] GitHub Actions uses OIDC federation (no long-lived credentials)
- [ ] Separate IAM roles for: engine task, backend task, migration runner, monitoring
- [ ] All IAM policies follow least privilege
- [ ] IAM Access Analyzer enabled — flag over-permissive policies
- [ ] Service Control Policies (SCPs) prevent:
  - Disabling CloudTrail
  - Disabling GuardDuty
  - Modifying VPC flow logs
  - Creating public S3 buckets

### 13.3 Data Protection

- [ ] RDS encrypted at rest (KMS CMK)
- [ ] RDS encrypted in transit (TLS 1.3, `sslmode=verify-full`)
- [ ] ElastiCache encrypted at rest (KMS)
- [ ] ElastiCache encrypted in transit (TLS)
- [ ] ElastiCache AUTH token enabled
- [ ] S3 buckets: block all public access, SSE-KMS encryption, versioning
- [ ] EBS volumes encrypted (Fargate handles this automatically)
- [ ] Secrets Manager for all credentials (never in env files, never in code)
- [ ] KMS key rotation enabled (annual for AWS-managed, manual for wallet CMK)
- [ ] No secrets in Docker images (verified by `.dockerignore` excluding `.env`)
- [ ] Application-level log redaction (Pino redact paths for secrets — Rule R5)

### 13.4 Application Security (Already Implemented — Verify in Production)

- [ ] HMAC-SHA256 authentication on all API routes (constant-time comparison)
- [ ] Absolute timestamp window on HMAC (past AND future bounded)
- [ ] Single-use HMAC signatures (Redis SET NX, fails closed)
- [ ] Admin authentication via separate `X-Admin-Key` (constant-time comparison)
- [ ] Webhook signature verification over raw body bytes
- [ ] Rate limiting on all write endpoints (keyed on IP, not idempotency key)
- [ ] Zod validation on all request bodies
- [ ] Integer-only arithmetic for all monetary values (Rule R3)
- [ ] Provider credentials in env only, never in DB
- [ ] Distributed lock with SET NX + token-safe release
- [ ] Database UNIQUE constraint on `executions.quote_id`
- [ ] Append-only ledger enforced by DB triggers + REVOKE
- [ ] Hash chain on ledger operations
- [ ] Payout velocity limits (R17)
- [ ] Error responses never leak stack traces in production
- [ ] `helmet` middleware enabled (HTTP security headers)

### 13.5 Container Security

- [ ] Multi-stage Dockerfile (no source code in runtime image)
- [ ] Runtime container runs as `USER node` (non-root)
- [ ] No `latest` tag in production — use git SHA tags
- [ ] ECR image scanning enabled, block on CRITICAL/HIGH CVEs
- [ ] Base image (`node:20-slim`) updated monthly
- [ ] No shell access to production containers
- [ ] Read-only root filesystem (`readonlyRootFilesystem: true` in task def)

### 13.6 Audit & Compliance

- [ ] AWS CloudTrail enabled (management + data events)
- [ ] CloudTrail log file integrity validation enabled
- [ ] AWS Config enabled for resource compliance tracking
- [ ] AWS GuardDuty enabled (threat detection)
- [ ] Application audit logs (every admin action, every state transition)
- [ ] Ledger reconciliation runs nightly (`ReconciliationJob`)
- [ ] Ledger hash chain verification runs nightly
- [ ] All logs archived to S3 Glacier (7-year retention for financial compliance)
- [ ] VPC Flow Logs for network forensics

### 13.7 Redis-Specific Security

- [ ] Eviction policy is `noeviction` (engine refuses to boot otherwise)
- [ ] AUTH token required (from Secrets Manager)
- [ ] TLS in transit
- [ ] No public accessibility
- [ ] Security group allows only engine/backend tasks
- [ ] Backup snapshots enabled

---

## 14. Wallet & Key Custody on AWS

> [!CAUTION]
> **The private key IS the treasury. If lost, the funds are gone. If stolen, the funds are gone.**

### MVP: Self-Managed HD Wallet + AWS KMS

```
┌───────────────────────────────────────────────────────────────┐
│                    KEY CUSTODY (MVP)                           │
│                                                               │
│   HD Wallet Seed ──► Encrypted by KMS CMK (alias/xanepay-    │
│                      wallet) ──► Stored in Secrets Manager    │
│                                                               │
│   At Runtime:                                                 │
│     1. Engine task role calls Secrets Manager                 │
│     2. Gets encrypted seed                                    │
│     3. KMS decrypts (engine task role has kms:Decrypt)        │
│     4. Seed held in memory ONLY — never written to disk       │
│     5. BIP-44 derivation generates per-user deposit addresses │
│                                                               │
│   Controls:                                                   │
│     ● KMS key policy: ONLY engine task role can decrypt       │
│     ● CloudTrail logs every KMS Decrypt call                  │
│     ● Alarm on KMS Decrypt from unexpected principal          │
│     ● Seed backup: encrypted copy in a SEPARATE AWS account   │
│       (break-glass recovery — requires 2 humans + MFA)        │
│                                                               │
│   Limits:                                                     │
│     ● Hot wallet holds MAX $50,000 equivalent                 │
│     ● Excess swept to cold storage (hardware wallet, offline) │
│     ● Multi-sig required for withdrawals > $5,000             │
└───────────────────────────────────────────────────────────────┘
```

### Growth: Institutional Custody (Fireblocks / BitGo)

When daily volume exceeds $50k, migrate to managed MPC custody:

| Provider | Integration | Cost | Recommendation |
|---|---|---|---|
| Fireblocks | REST API + SDK | $1,000+/mo | Best for multi-chain |
| BitGo | REST API + SDK | $1,500+/mo | Best for compliance |

The `ITreasuryPort` interface is already defined — migration is a new adapter, zero core changes.

---

## 15. Backup & Disaster Recovery

### RPO / RTO Targets

| Component | RPO (max data loss) | RTO (max downtime) | Strategy |
|---|---|---|---|
| PostgreSQL (ledger) | **0 seconds** (Multi-AZ sync) | < 2 minutes (auto-failover) | Multi-AZ + PITR (30 days) |
| Redis (queues) | < 1 hour (snapshot) | < 5 minutes (auto-failover) | Multi-AZ + daily snapshots |
| Application (ECS) | N/A (stateless) | < 2 minutes (replacement task) | Multi-AZ + auto-scaling |
| Secrets | N/A (managed service) | < 1 minute | Secrets Manager HA |
| Wallet seed | **0** (offline backup) | < 30 minutes (manual) | Cross-account encrypted backup |

### Disaster Recovery Procedures

```
SCENARIO: RDS Primary Failure
  ─► Automatic: Multi-AZ failover (< 120s)
  ─► Engine reconnects automatically (connection pooling handles DNS change)
  ─► No manual intervention required

SCENARIO: Full Region Failure
  ─► Restore RDS from cross-region snapshot (pre-configured)
  ─► Deploy ECS tasks in DR region
  ─► Update Route 53 failover record
  ─► RTO: < 4 hours (manual process)

SCENARIO: Wallet Key Compromise
  ─► Immediately: Pause all payouts via admin API
  ─► Transfer remaining treasury to new wallet
  ─► Rotate KMS key
  ─► Generate new HD seed
  ─► Update Secrets Manager
  ─► Resume payouts with new wallet address
  ─► Post-mortem: how was the key accessed?
```

---

## 16. Compliance & Regulatory

### Financial Data Handling

| Requirement | Implementation |
|---|---|
| **Data retention** | 7-year minimum for all financial records (ledger, audit logs) |
| **Immutable audit trail** | Append-only ledger with DB triggers, hash chain, REVOKE |
| **Reconciliation** | Nightly automated reconciliation with alerts on drift |
| **Access logging** | CloudTrail + application audit logs for every admin action |
| **Encryption** | At rest (KMS) + in transit (TLS 1.3) for all data |
| **PII handling** | User IDs only in engine (no names, no account numbers stored permanently) |
| **Incident reporting** | Defined playbook with < 24hr notification requirement |

### Nigerian Regulatory Considerations

| Regulation | Status | Action Required |
|---|---|---|
| CBN guidelines on crypto | Monitor actively | Ensure operations comply with current CBN position |
| NDPR (Data Protection) | Applicable | Data processing agreement, encryption, access controls |
| AML/CFT | Applicable | KYC enforcement (XaneApp side), transaction monitoring |
| CAC registration | Required for Bank Model B/C | Founders must complete |

### GDPR / Data Protection

| Requirement | Implementation |
|---|---|
| Data minimization | Engine stores only transaction data — no PII beyond user IDs |
| Right to erasure | Ledger is append-only by design — use anonymization, not deletion |
| Data processing records | Audit log + CloudTrail provide full processing records |
| Breach notification | CloudWatch alarms + incident response playbook |

---

## 17. Cost Estimation

### Monthly Estimate (Production — Moderate Traffic)

| Service | Configuration | Monthly Cost (USD) |
|---|---|---|
| ECS Fargate (Engine) | 2 tasks × 0.5 vCPU × 1 GB × 730 hrs | ~$30 |
| ECS Fargate (Backend) | 2 tasks × 0.5 vCPU × 1 GB × 730 hrs | ~$30 |
| RDS PostgreSQL | db.r6g.large, Multi-AZ, 100 GB gp3 | ~$350 |
| ElastiCache Redis | cache.r6g.large, Multi-AZ | ~$300 |
| ALB | 1 ALB + moderate traffic | ~$25 |
| NAT Gateway | 2 gateways × 730 hrs + data | ~$70 |
| Route 53 | 1 hosted zone + health checks | ~$5 |
| Secrets Manager | ~10 secrets | ~$5 |
| KMS | 2 CMKs + API calls | ~$5 |
| CloudWatch | Logs + alarms + metrics | ~$30 |
| ECR | Image storage | ~$5 |
| WAF | Basic rules | ~$10 |
| S3 (logs/backups) | ~50 GB | ~$2 |
| **TOTAL** | | **~$867/month** |

### Cost Optimization Opportunities

| Optimization | Savings | When |
|---|---|---|
| Reserved Instances (RDS + Redis) — 1 year | ~30% on RDS/Redis | After 3 months of stable usage |
| Savings Plans (Fargate) | ~20% on compute | After 3 months |
| Single NAT Gateway (accept AZ risk) | ~$35/month | MVP only |
| Smaller RDS instance (db.t4g.medium) | ~$200/month | MVP (upgrade when needed) |

### MVP Cost (Reduced)

For MVP, use smaller instances and accept slightly reduced resilience:

| Change | Savings |
|---|---|
| `db.t4g.medium` instead of `db.r6g.large` | -$250 |
| `cache.t4g.medium` instead of `cache.r6g.large` | -$200 |
| 1 NAT Gateway instead of 2 | -$35 |
| **MVP Total** | **~$380/month** |

---

## 18. Deployment Runbook — Step by Step

### Phase 1: AWS Account Setup (Day 1)

```
1. Create dedicated AWS account (NOT your personal account)
   ─► Enable MFA on root (hardware key preferred)
   ─► Set up IAM Identity Center (SSO)
   ─► Enable CloudTrail (management + data events)
   ─► Enable GuardDuty
   ─► Enable AWS Config
   ─► Enable IAM Access Analyzer
   ─► Set account-level S3 Block Public Access

2. Select region
   ─► eu-west-1 (Ireland) — good latency to Nigeria, full service availability
   ─► OR af-south-1 (Cape Town) — closest, but check service availability

3. Request service quota increases (if needed)
   ─► Fargate vCPU limit (default: 6, request: 20)
   ─► Elastic IPs (for NAT Gateways)
```

### Phase 2: Network Infrastructure (Day 1-2)

```
1. Create VPC (Section 3 design)
2. Create subnets (public, private, data — 2 AZs)
3. Create Internet Gateway, NAT Gateways
4. Create route tables
5. Create VPC endpoints (ECR, Secrets Manager, CloudWatch, S3, KMS, STS)
6. Create security groups (Section 3)
7. Enable VPC Flow Logs
```

### Phase 3: Data Layer (Day 2-3)

```
1. Create KMS CMKs (master + wallet)
2. Create RDS subnet group (data subnets)
3. Create RDS parameter group (Section 5)
4. Launch RDS PostgreSQL 16 Multi-AZ
5. Create database users/roles (Section 5)
6. Create ElastiCache subnet group (data subnets)
7. Create ElastiCache parameter group (noeviction!)
8. Launch ElastiCache Redis 7 with replication
9. Store connection strings in Secrets Manager
```

### Phase 4: Application Setup (Day 3-4)

```
1. Create ECR repositories (engine, backend)
2. Build and push Docker images
3. Create ECS cluster
4. Create task definitions (Section 4)
5. Create ALB + target groups + listener rules
6. Request ACM certificate, validate via Route 53
7. Create ECS services (engine, backend)
8. Run migrations (docker run --target builder)
9. Verify health checks pass
10. Attach WAF to ALB
```

### Phase 5: CI/CD & Monitoring (Day 4-5)

```
1. Configure GitHub Actions OIDC provider in AWS
2. Create GitHub Actions IAM role
3. Set up deployment workflow (Section 11)
4. Create CloudWatch alarms (Section 12)
5. Create SNS topics for alerts
6. Configure Sentry project
7. Test full deployment pipeline
8. Run engine test suite against production infrastructure
9. Verify ledger reconciliation runs clean
```

### Phase 6: Security Hardening (Day 5-6)

```
1. Complete Security Hardening Checklist (Section 13)
2. Run AWS Trusted Advisor security checks
3. Run AWS Security Hub findings
4. Verify all secrets are in Secrets Manager
5. Verify no public access to any resource
6. Test failover (stop primary RDS → verify auto-failover)
7. Test scaling (load test → verify auto-scaling)
8. Document all access credentials and who holds them
```

### Phase 7: Go-Live (Day 6-7)

```
1. Final ledger reconciliation: debits === credits
2. Final hash chain verification: clean
3. Test provider connectivity (sandbox → production keys)
4. Switch DNS to production ALB
5. Monitor for 24 hours
6. Enable auto-scaling policies
7. Enable nightly reconciliation job
8. ✅ LIVE
```

---

## 19. Incident Response Playbook

### Severity Levels

| Level | Definition | Response Time | Examples |
|---|---|---|---|
| **P1 — Critical** | Money at risk, system down, data breach | < 15 min | Ledger drift, double payout, key compromise, total outage |
| **P2 — High** | Degraded service, potential money impact | < 1 hour | Provider down (fallback working), high error rate, Redis near full |
| **P3 — Medium** | Non-critical degradation | < 4 hours | Slow queries, non-critical alert, single task failure |
| **P4 — Low** | Informational | Next business day | Log anomaly, minor config issue |

### P1 Response: Ledger Drift Detected

```
1. ALERT fires: reconcileLedgerBalances() reports drift
2. IMMEDIATE: Pause all payouts (POST /admin/payouts/pause)
3. DIAGNOSE: 
   ─► Check reconciliation report: which accounts drifted?
   ─► Check hash chain verification: is the chain broken?
   ─► If chain broken: someone modified ledger history — ESCALATE TO SECURITY INCIDENT
   ─► If chain clean: operational drift (missed webhook, stuck conversion)
4. RESOLVE:
   ─► Identify stuck conversions: query executions in non-terminal state
   ─► For each: poll provider for actual status
   ─► Post corrective ledger entries (REVERSAL or COMPLETION)
   ─► Re-run reconciliation — must pass clean
5. RESUME: Unpause payouts
6. POST-MORTEM: Within 24 hours
```

### P1 Response: Suspected Key Compromise

```
1. ALERT: Unexpected KMS Decrypt call OR unauthorized payout detected
2. IMMEDIATE:
   ─► Pause ALL engine operations (POST /admin/operations/halt)
   ─► Revoke the compromised IAM role/credentials
   ─► Rotate KMS CMK (schedule key deletion for old key)
3. CONTAIN:
   ─► Check CloudTrail: what was accessed and when?
   ─► Check wallet balance: has anything been drained?
   ─► If funds stolen: engage law enforcement, notify affected users
4. RECOVER:
   ─► Generate new wallet seed
   ─► Store in Secrets Manager under new KMS CMK
   ─► Transfer remaining treasury to new wallet
   ─► Update all provider webhook URLs if needed
5. RESUME: Gradual restart with monitoring
6. POST-MORTEM: Within 24 hours, mandatory for all stakeholders
```

### P1 Response: Double Payout Detected

```
1. ALERT: reconciliation or monitoring detects double payout
2. IMMEDIATE: Pause payouts
3. DIAGNOSE:
   ─► Which conversion was double-paid?
   ─► Did the Redis lock fail? (check logs for lock acquisition)
   ─► Did the DB constraint fail? (check executions table)
   ─► Did the provider dedupe fail? (check provider dashboard)
4. RECOVER:
   ─► Contact provider to reverse duplicate payout
   ─► Post REVERSAL ledger entries for the duplicate
   ─► If unrecoverable: log as loss, update treasury position
5. FIX:
   ─► Identify which safety layer failed
   ─► Write regression test that reproduces the failure
   ─► Deploy fix through normal CI/CD
6. POST-MORTEM: Mandatory within 24 hours
```

---

## Appendix A: Environment Variables (Production)

> [!WARNING]
> **These values come from Secrets Manager at container launch time — they are NEVER in `.env` files, NEVER committed to git, NEVER in Docker images.**

| Variable | Source | Description |
|---|---|---|
| `NODE_ENV` | Task definition (hardcoded) | `production` |
| `PORT` | Task definition (hardcoded) | `3001` (engine) / `8080` (backend) |
| `DATABASE_URL` | Secrets Manager | Full PostgreSQL connection string with `sslmode=verify-full` |
| `REDIS_URL` | Secrets Manager | Full Redis connection string with TLS (`rediss://`) |
| `HMAC_SECRET` | Secrets Manager | ≥32 char HMAC signing key |
| `ADMIN_API_KEY` | Secrets Manager | ≥32 char admin authentication key |
| `ONSWITCH_SERVICE_KEY` | Secrets Manager | Provider API key |
| `SENTRY_DSN` | Secrets Manager | Error tracking endpoint |
| `DEFAULT_FEE_BPS` | Task definition | `200` (2% — fallback fee) |
| `BULLMQ_CONCURRENCY` | Task definition | `5` (NEVER higher — proven by load tests) |
| `LOG_LEVEL` | Task definition | `info` (never `debug` in production) |

---

## Appendix B: AWS Services Quick Reference

| What You Need | AWS Service | Why Not The Alternative |
|---|---|---|
| Run containers | ECS Fargate | EC2 = patching burden; Lambda = cold starts, no persistent connections |
| PostgreSQL | RDS | Self-managed EC2 = operational nightmare for a financial DB |
| Redis | ElastiCache | Self-managed = same; Redis Cloud = vendor lock-in |
| Secrets | Secrets Manager | Parameter Store = no auto-rotation for DB creds; .env files = insecure |
| Encryption keys | KMS | Self-managed HSM = cost prohibitive at this scale |
| DNS | Route 53 | External DNS = no auto-validation for ACM certs |
| TLS certs | ACM | Let's Encrypt = manual renewal; paid certs = unnecessary cost |
| Load balancer | ALB | NLB = no path-based routing; CloudFront = not for APIs |
| Container images | ECR | Docker Hub = rate limits, public by default |
| Monitoring | CloudWatch + Sentry | Datadog = $$$; Grafana Cloud = another vendor to manage |
| WAF | AWS WAF | Cloudflare = requires DNS proxy (fine if already using CF) |
| VPN/access | SSM Session Manager | SSH = key management nightmare, open ports |

---

*This document is the authoritative deployment and security reference for XanePay.*
*No component may be deployed to production without satisfying the requirements defined here.*
*All changes to infrastructure must be reviewed against this document before implementation.*
