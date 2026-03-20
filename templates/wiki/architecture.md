---
title: Architecture
last_updated: {{DATE}}
status: template
---

# Architecture

## Overview

<!-- One paragraph: what the system does, who uses it, what problem it solves -->

## System Architecture Diagram

<!-- High-level system context showing external actors and systems -->

```mermaid
graph TB
    subgraph External
        User[User / Browser]
        Admin[Admin Portal]
        ExtAPI[External API]
    end

    subgraph Application
        Frontend[Frontend App]
        API[API Layer]
        Workers[Background Workers]
    end

    subgraph Data
        DB[(Database)]
        Cache[(Cache)]
        Storage[File Storage]
    end

    User --> Frontend
    Admin --> Frontend
    Frontend --> API
    API --> DB
    API --> Cache
    API --> ExtAPI
    Workers --> DB
    Workers --> ExtAPI
```

<!-- Replace with actual system components -->

## Application Layers

```mermaid
graph LR
    subgraph Presentation
        Pages[Pages / Routes]
        Components[UI Components]
        Layouts[Layouts]
    end

    subgraph Business
        Services[Services]
        Hooks[Hooks / Logic]
        Validators[Validation]
    end

    subgraph Data_Layer[Data Access]
        Repositories[Repositories / DAL]
        Models[Models / Entities]
        Migrations[Migrations]
    end

    subgraph Infrastructure
        API_Routes[API Routes]
        Middleware[Middleware]
        Config[Configuration]
    end

    Pages --> Hooks
    Components --> Hooks
    Hooks --> Services
    Services --> Repositories
    API_Routes --> Services
    Middleware --> API_Routes
    Repositories --> Models
```

<!-- Replace with actual layer structure -->

## Layer Responsibilities

| Layer | Responsibility | Key Patterns | Location |
|-------|---------------|-------------|----------|
| **Presentation** | UI rendering, user interaction, routing | Server/Client components, layouts | `src/app/`, `src/components/` |
| **Business Logic** | Domain rules, data transformation, orchestration | Services, hooks, validation | `src/lib/`, `src/hooks/` |
| **Data Access** | Database queries, ORM operations, caching | Repository pattern, Prisma/EF Core | `src/models/`, `prisma/` |
| **Infrastructure** | API routes, middleware, auth, config | Route handlers, middleware chain | `src/app/api/`, `src/middleware.ts` |

<!-- Replace with actual layers -->

## Component Hierarchy

```mermaid
graph TD
    RootLayout[Root Layout] --> PageLayout[Page Layout]
    PageLayout --> Navbar[Navbar]
    PageLayout --> PageContent[Page Content]
    PageLayout --> Footer[Footer]

    PageContent --> Section1[Hero Section]
    PageContent --> Section2[Content Section]
    PageContent --> Section3[Feature Section]

    Section1 --> UIComp1[Button]
    Section1 --> UIComp2[Badge]
    Section2 --> UIComp3[Card]
    Section2 --> UIComp4[Modal]
```

<!-- Replace with actual component tree -->

## Data Flow

```mermaid
sequenceDiagram
    participant U as User
    participant F as Frontend
    participant A as API
    participant D as Database

    U->>F: User action (click, form submit)
    F->>A: HTTP request (GET/POST/PUT/DELETE)
    A->>A: Validate & authorize
    A->>D: Query / mutation
    D-->>A: Result
    A-->>F: JSON response
    F-->>U: Updated UI
```

<!-- Replace with actual data flow for key operations -->

## Technology Stack

| Component | Technology | Version | Purpose |
|-----------|-----------|---------|---------|
| Runtime | Node.js | v20.x | Server-side execution |
| Framework | Next.js | 14.x | Full-stack React framework |
| Language | TypeScript | 5.x | Type-safe development |
| Styling | Tailwind CSS | 3.x | Utility-first CSS |
| Database | PostgreSQL | 15.x | Relational data store |
| ORM | Prisma | 5.x | Database access & migrations |
| Testing | Vitest + Playwright | latest | Unit + E2E testing |

<!-- Replace with actual stack -->

## Key Patterns & Conventions

| Pattern | Where Used | Example |
|---------|-----------|---------|
| Server Components | Default for all pages/layouts | `src/app/page.tsx` |
| Client Components | Interactive UI only | `'use client'` directive |
| Repository Pattern | Data access layer | `src/lib/repositories/` |
| Middleware Chain | Auth, logging, CORS | `src/middleware.ts` |
| Error Boundaries | Graceful error handling | `error.tsx` per route |

<!-- Replace with actual patterns -->

## External Dependencies

| Service | Purpose | Integration Point | Docs |
|---------|---------|-------------------|------|
| Azure DevOps | CI/CD, work items | Pipeline + REST API | `docs/wiki/deployment.md` |

<!-- Add actual external services -->

## Security Architecture

```mermaid
flowchart LR
    Request[Incoming Request] --> CORS[CORS Check]
    CORS --> Auth[Authentication]
    Auth --> Authz[Authorization]
    Authz --> RateLimit[Rate Limiting]
    RateLimit --> Handler[Route Handler]
    Handler --> Response[Response]

    Auth -->|Failed| R401[401 Unauthorized]
    Authz -->|Failed| R403[403 Forbidden]
    RateLimit -->|Exceeded| R429[429 Too Many Requests]
```

<!-- Replace with actual security flow -->
