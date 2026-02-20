# {Project Name} — Design Document

> **Template**: During actual usage, this file will be created as
> `docs/plans/YYYY-MM-DD-{topic}-design.md` (e.g., `2026-02-13-user-auth-design.md`)
> in the target project directory by the brainstorm phase.

## Overview

{One-paragraph summary of what's being built and why.}

## Problem Statement

{What problem this solves. Who has this problem. Why existing solutions fall short.}

## Approach

### Option A: {Approach Name}
- **Pros**: {list}
- **Cons**: {list}
- **Trade-offs**: {list}

### Option B: {Approach Name}
- **Pros**: {list}
- **Cons**: {list}
- **Trade-offs**: {list}

### Chosen Approach
{Which option was selected and why.}

## Architecture

{High-level architecture description. Components, layers, boundaries.}

### Components
- **{Component A}**: {responsibility}
- **{Component B}**: {responsibility}

### Data Flow
{How data moves through the system. Request lifecycle.}

## Data Model

{Database schema, entities, relationships.}

### {Entity Name}
| Field | Type | Description |
|-------|------|-------------|
| id | UUID | Primary key |
| ... | ... | ... |

## API Design

{Endpoints, contracts, request/response shapes.}

### {Endpoint}
- **Method**: GET/POST/PUT/DELETE
- **Path**: `/api/v1/{resource}`
- **Request**: {shape}
- **Response**: {shape}

## Security Considerations

{Authentication, authorization, input validation, data protection.}

## Edge Cases

{Known edge cases and how they're handled.}

## User Stories

- As a {user}, I want to {action} so that {benefit}

## Open Questions

{Anything unresolved that planning needs to address.}

## Decision Log

| Decision | Rationale | Date |
|----------|-----------|------|
| {what} | {why} | {when} |
