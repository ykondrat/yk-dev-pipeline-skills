# Architecture

> **Template**: During actual usage, this file will be generated as
> `docs/architecture.md` in the target project directory by the
> documentation phase, derived from the actual codebase.

## Overview
{High-level description of the system architecture. Derived from spec.md design
decisions and actual code structure.}

## System Architecture Diagram

```mermaid
graph TB
    Client[Client / Browser] --> API[API Gateway]
    API --> Auth[Auth Middleware]
    Auth --> Routes[Route Handlers]
    Routes --> Services[Service Layer]
    Services --> Repos[Repository Layer]
    Repos --> DB[(Database)]
    Services --> Cache[(Redis Cache)]
    Services --> External[External APIs]
```

## Project Structure
```
src/
├── config/        — {what this contains}
├── types/         — {what this contains}
├── domain/        — {what this contains}
│   └── user/      — {what this contains}
├── infrastructure/ — {what this contains}
├── api/           — {what this contains}
└── utils/         — {what this contains}
```

## Architecture Decisions

{Key decisions made during brainstorm/planning and how they're reflected in code.}

### {Decision 1: e.g., "Layered Architecture"}
**Decision:** {what was decided}
**Rationale:** {why — from design doc}
**Implementation:** {how it's reflected in code structure}

### {Decision 2: e.g., "PostgreSQL over MongoDB"}
**Decision:** {what was decided}
**Rationale:** {why}
**Implementation:** {how}

## Request Lifecycle — Sequence Diagram

```mermaid
sequenceDiagram
    participant C as Client
    participant M as Middleware
    participant R as Router
    participant S as Service
    participant D as Database

    C->>M: HTTP Request
    M->>M: Validate auth token
    M->>R: Authenticated request
    R->>R: Validate input
    R->>S: Call service method
    S->>D: Query / Mutation
    D-->>S: Result
    S-->>R: Domain response
    R-->>C: HTTP Response
```

## Data Flow

{How a request flows through the system — from HTTP to response.
Trace an actual endpoint through the layers.}

1. Request arrives at `src/api/routes/{resource}.routes.ts`
2. Middleware validates auth (`src/api/middleware/auth.ts`)
3. Handler calls service (`src/domain/{resource}/{resource}.service.ts`)
4. Service calls repository (`src/domain/{resource}/{resource}.repository.ts`)
5. Repository executes query (`src/infrastructure/database/...`)
6. Response serialized and returned

## Error Handling Flow

```mermaid
flowchart TD
    A[Incoming Request] --> B{Auth Valid?}
    B -->|No| C[401 Unauthorized]
    B -->|Yes| D{Input Valid?}
    D -->|No| E[400 Validation Error]
    D -->|Yes| F[Execute Business Logic]
    F --> G{Success?}
    G -->|Yes| H[200/201 Response]
    G -->|Domain Error| I[4xx Client Error]
    G -->|Unexpected| J[500 Internal Error + Log]
```

## Database Schema

{If database exists — ER diagram or table descriptions from actual schema/migrations.}

```mermaid
erDiagram
    USER ||--o{ POST : creates
    USER {
        uuid id PK
        string email UK
        string name
        timestamp created_at
    }
    POST {
        uuid id PK
        uuid author_id FK
        string title
        text content
        timestamp created_at
    }
```

## State Machine Diagram

{If the system has entities with lifecycle states.}

```mermaid
stateDiagram-v2
    [*] --> Draft
    Draft --> Published: publish()
    Draft --> Archived: archive()
    Published --> Archived: archive()
    Archived --> Draft: restore()
```

## External Dependencies

{External services, APIs, databases the system connects to.}

| Dependency | Purpose | Config |
|-----------|---------|--------|
| PostgreSQL | Primary data store | `DATABASE_URL` |
| Redis | Caching/sessions | `REDIS_URL` |

## Security Model

{How authentication and authorization work. Derived from actual middleware and
auth implementation.}

## Deployment Architecture

```mermaid
graph LR
    LB[Load Balancer] --> App1[App Instance 1]
    LB --> App2[App Instance 2]
    App1 --> DB[(Primary DB)]
    App2 --> DB
    App1 --> Redis[(Redis)]
    App2 --> Redis
    DB --> Replica[(Read Replica)]
```
