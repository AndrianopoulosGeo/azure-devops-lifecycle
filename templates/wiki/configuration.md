---
title: Configuration
last_updated: {{DATE}}
status: template
---

# Configuration

## Overview

<!-- Configuration management approach, environments, secrets handling -->

## Environment Variables

### Required

| Variable | Type | Description | Example |
|----------|------|-------------|---------|
| `DATABASE_URL` | string | Database connection string | `postgresql://user:pass@host:5432/db` |
| `JWT_SECRET` | string | JWT signing secret (min 32 chars) | `your-secret-key-here` |
| `AZURE_DEVOPS_PAT` | string | Azure DevOps access token | `xxxxxxxxxxxx` |

<!-- Replace with actual required variables -->

### Optional

| Variable | Type | Default | Description |
|----------|------|---------|-------------|
| `PORT` | int | `3000` | Server port |
| `LOG_LEVEL` | string | `info` | Logging level: debug, info, warn, error |
| `CACHE_TTL` | int | `3600` | Cache time-to-live in seconds |

<!-- Replace with actual optional variables -->

### Per-Environment

| Variable | Development | Staging | Production |
|----------|-------------|---------|-----------|
| `NODE_ENV` | `development` | `staging` | `production` |
| `DATABASE_URL` | Local DB | Staging DB | Production DB |
| `LOG_LEVEL` | `debug` | `debug` | `warn` |
| `RATE_LIMIT` | `1000/min` | `1000/min` | `100/min` |

<!-- Replace with actual per-environment values -->

## Configuration Flow

```mermaid
flowchart TD
    subgraph Sources[Configuration Sources]
        EnvFile[.env file<br/>Local development]
        Pipeline[Pipeline Variables<br/>CI/CD secrets]
        KeyVault[Key Vault / Secret Manager<br/>Production secrets]
        ConfigFile[Config Files<br/>appsettings, next.config]
    end

    subgraph Loading[Loading Order - Priority ↓]
        Env[Environment Variables]
        File[Config Files]
        Default[Default Values]
    end

    subgraph Usage
        App[Application Code]
    end

    EnvFile --> Env
    Pipeline --> Env
    KeyVault --> Env
    ConfigFile --> File
    Env --> App
    File --> App
    Default --> App
```

<!-- Replace with actual configuration flow -->

## Configuration Files

| File | Purpose | Environment |
|------|---------|-------------|
| `.env` | Local development variables | Development only |
| `.env.claude` | Lifecycle automation config | All (gitignored) |
| `next.config.ts` | Next.js framework config | All |
| `tsconfig.json` | TypeScript compiler options | All |
| `tailwind.config.ts` | Tailwind CSS configuration | All |

<!-- Replace with actual config files -->

## Secrets Management

### Local Development

- Store secrets in `.env` (gitignored)
- Never commit secrets to the repository
- Use `.env.example` as a template (no real values)

### CI/CD Pipeline

- Secrets stored in Azure DevOps Pipeline Variables (or Library)
- Marked as secret (masked in logs)
- Injected as environment variables during build

### Production

<!-- Document actual secrets management: Azure Key Vault, AWS Secrets Manager, etc. -->

```mermaid
flowchart LR
    Dev[Developer] -->|reads| EnvExample[.env.example]
    Dev -->|creates| EnvLocal[.env]
    Pipeline[CI Pipeline] -->|reads| PipelineVars[Pipeline Variables]
    Production[Production App] -->|reads| KeyVault[Key Vault]

    EnvLocal -.->|gitignored| Git[Git Repo]
    EnvExample -->|committed| Git
```

## Feature Flags (if applicable)

| Flag | Default | Description | Environments |
|------|---------|-------------|-------------|
| `ENABLE_NEW_UI` | `false` | New UI redesign | Staging only |
| `ENABLE_ANALYTICS` | `true` | Analytics tracking | All |

<!-- Replace with actual feature flags or remove section -->

## Adding New Configuration

1. Add variable to `.env.example` with description
2. Add to `docs/wiki/configuration.md` (this file)
3. Add validation in app startup (fail fast if required)
4. Add to CI/CD pipeline variables for staging/production
5. Update `docs/wiki/deployment.md` if it affects deployment
