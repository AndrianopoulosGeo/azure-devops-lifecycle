---
title: Deployment
last_updated: {{DATE}}
status: template
---

# Deployment

## Overview

<!-- Deployment strategy, zero-downtime approach, infrastructure summary -->

## Environments

| Environment | Branch | URL | Deploy Target | Auto-Deploy |
|-------------|--------|-----|--------------|-------------|
| Development | `develop` | `localhost:3000` | Local | — |
| Staging | `staging` | {{STAGING_URL}} | {{DEPLOY_TARGET}} | On merge |
| Production | `master` | {{PRODUCTION_URL}} | {{DEPLOY_TARGET}} | On merge |

## CI/CD Pipeline

```mermaid
flowchart TD
    subgraph Trigger
        Push[Git Push / Merge]
    end

    subgraph Build_Stage[Build Stage]
        Install[Install Dependencies]
        Lint[Lint & Type Check]
        UnitTest[Unit Tests]
        Build[Build Application]
    end

    subgraph Test_Stage[Test Stage]
        IntTest[Integration Tests]
        E2ETest[E2E Tests]
        Security[Security Scan]
    end

    subgraph Deploy_Staging[Deploy to Staging]
        StgDeploy[Deploy Application]
        StgHealth[Health Check]
        StgSmoke[Smoke Tests]
    end

    subgraph Deploy_Production[Deploy to Production]
        ProdApproval{Manual Approval?}
        ProdDeploy[Deploy Application]
        ProdHealth[Health Check]
        ProdSmoke[Smoke Tests]
        ProdMonitor[Monitor 15min]
    end

    Push --> Install
    Install --> Lint
    Lint --> UnitTest
    UnitTest --> Build
    Build --> IntTest
    IntTest --> E2ETest
    E2ETest --> Security
    Security -->|staging branch| StgDeploy
    Security -->|master branch| ProdApproval
    StgDeploy --> StgHealth
    StgHealth --> StgSmoke
    ProdApproval -->|Approved| ProdDeploy
    ProdDeploy --> ProdHealth
    ProdHealth --> ProdSmoke
    ProdSmoke --> ProdMonitor
```

<!-- Replace with actual pipeline -->

### Pipeline Triggers

| Branch | Pipeline | Action |
|--------|----------|--------|
| `feature/*` | CI only | Build + test, no deploy |
| `staging` | CI + CD | Build + test + deploy to staging |
| `master` | CI + CD | Build + test + deploy to production |

## Deployment Process

```mermaid
sequenceDiagram
    participant Dev as Developer
    participant Git as Git / Azure DevOps
    participant CI as CI Pipeline
    participant Env as Target Environment
    participant Mon as Monitoring

    Dev->>Git: Merge to staging/master
    Git->>CI: Trigger pipeline
    CI->>CI: Build & test
    CI->>Env: Deploy new version
    Env->>Env: Health check
    Env-->>CI: Healthy
    CI->>Mon: Notify deployment
    Mon->>Mon: Watch metrics 15min

    alt Metrics OK
        Mon-->>Dev: Deployment successful
    else Metrics degraded
        Mon-->>Dev: Alert — consider rollback
        Dev->>CI: Trigger rollback
        CI->>Env: Deploy previous version
    end
```

<!-- Replace with actual deployment process -->

## Infrastructure

```mermaid
graph TB
    subgraph DNS
        Domain[Domain / CDN]
    end

    subgraph Compute
        AppServer[App Server / Container]
        Worker[Background Workers]
    end

    subgraph Data
        DB[(Database)]
        Cache[(Redis Cache)]
        Blob[Blob Storage]
    end

    Domain --> AppServer
    AppServer --> DB
    AppServer --> Cache
    Worker --> DB
    Worker --> Blob
```

<!-- Replace with actual infrastructure -->

## Rollback Procedure

### Automated Rollback

```bash
# Revert to previous deployment
# (command depends on deploy target)
```

### Manual Rollback

1. Identify the last working commit: `git log --oneline -5`
2. Create hotfix branch: `git checkout -b hotfix/rollback master`
3. Revert the problematic commit: `git revert <sha>`
4. Push and let pipeline deploy: `git push origin hotfix/rollback`
5. Merge to master after verification

### Rollback Decision Matrix

| Symptom | Severity | Action |
|---------|----------|--------|
| 5xx error rate > 1% | Critical | Immediate rollback |
| Response time > 2x baseline | High | Rollback within 15 min |
| Feature bug, no data loss | Medium | Hotfix forward |
| UI cosmetic issue | Low | Fix in next release |

## Environment-Specific Configuration

| Config | Staging | Production |
|--------|---------|-----------|
| Debug mode | Enabled | Disabled |
| Log level | Debug | Warning |
| Rate limiting | Relaxed | Strict |
| SSL | Required | Required |

<!-- Replace with actual environment config -->
