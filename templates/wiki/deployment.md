---
title: Deployment
last_updated: YYYY-MM-DD
status: template
---

# Deployment

## Environments

| Environment | Branch | URL | Deploy Target |
|-------------|--------|-----|--------------|
| Development | `develop` | localhost | Local |
| Staging | `staging` | {{STAGING_URL}} | {{DEPLOY_TARGET}} |
| Production | `master` | {{PRODUCTION_URL}} | {{DEPLOY_TARGET}} |

## CI/CD Pipeline

<!-- Pipeline stages: build → test → deploy -->
<!-- Trigger conditions per branch -->

## Manual Deployment

<!-- Steps for manual deployment if needed -->

## Rollback Procedure

<!-- How to rollback a bad deployment -->
