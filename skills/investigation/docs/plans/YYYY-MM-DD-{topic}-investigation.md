# {Topic} — Investigation Report

> **Date**: YYYY-MM-DD
> **Type**: Bug fix | Refactoring | Performance optimization | Tech debt | Security fix
> **Severity**: Critical | Major | Minor
> **Status**: Completed

---

## Problem Statement

### Symptoms
- {What the user observes — the visible behavior}
- {Error messages, unexpected output, degraded performance}

### Impact
- **Blast radius**: {who/what is affected}
- **Urgency**: {blocking release | degraded experience | cosmetic}
- **Workaround**: {exists/none — describe if exists}

### Reproduction
```
Steps to reproduce:
1. {step}
2. {step}
3. {observe: ...}

Expected: {what should happen}
Actual: {what happens instead}
```

### Environment
- **Runtime**: {Node.js version, browser, etc.}
- **OS**: {if relevant}
- **Dependencies**: {relevant versions}
- **Commit**: {git SHA where issue was investigated}

---

## Investigation Process

### What Was Checked
1. {Area checked} — {finding or "no issues found"}
2. {Area checked} — {finding or "no issues found"}
3. {Area checked} — {finding or "no issues found"}

### Evidence
| # | File | Line(s) | Finding |
|---|------|---------|---------|
| 1 | `{file}` | {lines} | {what was found} |
| 2 | `{file}` | {lines} | {what was found} |

### Hypotheses Considered
| # | Hypothesis | Verdict | Reasoning |
|---|-----------|---------|-----------|
| 1 | {hypothesis} | **Confirmed** | {evidence} |
| 2 | {hypothesis} | Rejected | {evidence against} |
| 3 | {hypothesis} | Rejected | {evidence against} |

---

## Root Cause Analysis

### Root Cause
{Clear, concise statement of the root cause}

### Mechanism
{How the root cause produces the observed symptoms — the chain of causation}

### Code Path
```
{Entry point}
  → {file:line} — {what happens}
  → {file:line} — {what happens}
  → {file:line} — {ROOT CAUSE: what goes wrong}
  → {file:line} — {resulting symptom}
```

---

## Affected Areas

| File | Impact | Change Needed |
|------|--------|---------------|
| `{file}` | {what's wrong here} | {what needs to change} |
| `{file}` | {ripple effect} | {what needs to change} |

---

## Proposed Fix Approaches

### Option A: {Name} (Recommended)
{Description of the approach}

- **Pros**: {benefits}
- **Cons**: {drawbacks}
- **Effort**: Small | Medium | Large
- **Risk**: Low | Medium | High

### Option B: {Name}
{Description of the alternative approach}

- **Pros**: {benefits}
- **Cons**: {drawbacks}
- **Effort**: Small | Medium | Large
- **Risk**: Low | Medium | High

### Decision
{Which option was chosen and why}

---

## Risk Assessment

- **Regression risk**: {what could break — specific areas to watch}
- **Data impact**: {any data migration, schema changes, cache invalidation}
- **Rollout strategy**: {big bang | phased | feature flagged}
- **Rollback plan**: {how to revert if things go wrong}

---

## Testing Strategy

### Verification Tests
- {Test that proves the fix works — the reproduction case as a test}

### Regression Tests
- {Test that ensures existing behavior isn't broken}
- {Test for edge cases discovered during investigation}

---

## Decision Log

| Decision | Rationale | Alternatives Considered |
|----------|-----------|------------------------|
| {decision} | {why} | {what else was considered} |
