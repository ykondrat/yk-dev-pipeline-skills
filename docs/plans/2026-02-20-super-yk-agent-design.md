# Super-YK Agent Design

> Deep dive analysis and design for a strict orchestrator layer that ensures no steps are skipped across the entire yk-dev-pipeline.

**Date**: 2026-02-20
**Status**: Draft — pending brainstorm/planning phases

---

## Deep Dive Analysis

### What's Already Strong

The current pipeline is well-architected:
- **Artifact chaining** is clean: `spec.md → plan.md → code → review.md → test-report.md → docs`
- **Progressive loading** prevents context bloat (references loaded on demand)
- **Failure recovery loops** are thoughtful (code-review → fix-plan → implementation, testing → fix-test-plan → implementation)
- **Quality gates** after every 3-task batch enforce discipline
- **Severity system** in code review is production-grade

### The Core Problem: Why Steps Get Skipped

After reading every SKILL.md line by line, 5 root causes identified:

| # | Root Cause | Example |
|---|-----------|---------|
| 1 | **Phase-level state, not step-level** | `pipeline-state.json` tracks "brainstorm: completed" but not "brainstorm step 3 of 6: completed". If conversation breaks mid-phase, Claude doesn't know where it left off. |
| 2 | **No verification after each step** | Steps say "do X" but never say "verify X was done before moving to Y". Claude can silently skip. |
| 3 | **Context pressure** | When context fills up, Claude compresses earlier messages and "forgets" it was on step 4, jumping to step 6. |
| 4 | **Steps are prose, not state machine transitions** | The instructions read like a guide, not an enforced protocol. There's no guard preventing `step N → step N+2`. |
| 5 | **No self-audit mechanism** | Claude never checks "did I actually do all steps?" before marking a phase complete. |

---

## Super-YK Agent Design

### Core Concept

A **strict orchestrator layer** on top of the existing pipeline that makes every step an **explicit, verifiable, state-tracked transition** — not a suggestion.

### Architecture: 3 Layers

```
┌─────────────────────────────────────────────────┐
│           SUPER-YK ORCHESTRATOR                  │
│  (Enhanced Router — strict state machine)        │
│  • Step-level state tracking                     │
│  • Pre/post verification per step                │
│  • Context window management                     │
│  • Self-audit before phase completion            │
├─────────────────────────────────────────────────┤
│           PHASE EXECUTORS                        │
│  (Enhanced phase SKILLs — each step is a         │
│   named transition with inputs/outputs/guards)   │
│  • brainstorm[1..6]  planning[1..7]             │
│  • implementation[1..8]  code-review[1..6]      │
│  • testing[1..7]  documentation[1..6]           │
├─────────────────────────────────────────────────┤
│           REFERENCE LAYER                        │
│  (Unchanged — loaded on demand per step)         │
│  • js-ts-best-practices, databases-*,           │
│  • frameworks-*, review-checklists,             │
│  • test-patterns, doc-templates                 │
└─────────────────────────────────────────────────┘
```

### Key Innovation 1: Step-Level State in `pipeline-state.json`

**Current** (phase-level only):
```json
{
  "brainstorm": { "status": "completed" },
  "planning": { "status": "in-progress" }
}
```

**Super-YK** (step-level):
```json
{
  "version": "2.0",
  "project": "url-shortener",
  "currentPhase": "planning",
  "currentStep": 3,
  "phases": {
    "brainstorm": {
      "status": "completed",
      "completedAt": "2026-02-20T10:30:00Z",
      "steps": {
        "1_understand_context": { "status": "completed", "verified": true },
        "2_ask_questions": { "status": "completed", "verified": true, "meta": { "questionsAsked": 8 } },
        "3_explore_approaches": { "status": "completed", "verified": true },
        "4_present_design": { "status": "completed", "verified": true },
        "5_generate_output": { "status": "completed", "verified": true, "artifacts": ["spec.md", "docs/plans/2026-02-20-url-shortener-design.md"] },
        "6_handoff": { "status": "completed", "verified": true }
      }
    },
    "planning": {
      "status": "in-progress",
      "startedAt": "2026-02-20T10:35:00Z",
      "steps": {
        "1_read_spec": { "status": "completed", "verified": true },
        "2_identify_blocks": { "status": "completed", "verified": true },
        "3_define_structure": { "status": "in-progress", "verified": false },
        "4_task_breakdown": { "status": "pending" },
        "5_present_plan": { "status": "pending" },
        "6_generate_plan": { "status": "pending" },
        "7_handoff": { "status": "pending" }
      }
    }
  },
  "artifacts": {
    "spec.md": { "exists": true, "checksum": "a1b2c3", "updatedAt": "..." },
    "plan.md": { "exists": false }
  },
  "qualityGates": [],
  "recoveryHistory": []
}
```

This means if the conversation breaks at planning step 3, the next session reads state and resumes **exactly** at step 3 — not from scratch, not from step 1.

### Key Innovation 2: Step Verification Protocol

Every step in every phase gets wrapped with this protocol:

```
╔══════════════════════════════════════╗
║  STEP EXECUTION PROTOCOL            ║
╠══════════════════════════════════════╣
║                                      ║
║  1. PRE-CHECK                        ║
║     - Previous step verified? ✓      ║
║     - Required inputs exist?  ✓      ║
║     - No blockers in state?   ✓      ║
║     → If any ✗ → HALT, report why    ║
║                                      ║
║  2. ANNOUNCE                         ║
║     "Starting [Phase] Step N/M:      ║
║      [step name]"                    ║
║                                      ║
║  3. EXECUTE                          ║
║     (Follow phase SKILL.md exactly)  ║
║                                      ║
║  4. POST-CHECK                       ║
║     - Expected outputs created?  ✓   ║
║     - Quality criteria met?      ✓   ║
║     → If any ✗ → retry or escalate   ║
║                                      ║
║  5. RECORD                           ║
║     - Mark step completed in state   ║
║     - Set verified: true             ║
║     - Log artifacts produced         ║
║                                      ║
║  6. TRANSITION                       ║
║     - If last step → phase audit     ║
║     - Else → next step PRE-CHECK     ║
║                                      ║
╚══════════════════════════════════════╝
```

### Key Innovation 3: Phase Completion Audit

Before marking any phase "completed", the orchestrator runs a **mandatory self-audit**:

```
PHASE COMPLETION AUDIT: [brainstorm]
─────────────────────────────────────
Step 1 (understand_context):  ✓ completed, ✓ verified
Step 2 (ask_questions):       ✓ completed, ✓ verified (8 questions asked)
Step 3 (explore_approaches):  ✓ completed, ✓ verified
Step 4 (present_design):      ✓ completed, ✓ verified
Step 5 (generate_output):     ✓ completed, ✓ verified
  → spec.md exists?           ✓
  → design doc exists?        ✓
Step 6 (handoff):             ✓ completed, ✓ verified
  → pipeline-state updated?   ✓

ALL STEPS VERIFIED. Phase [brainstorm] → completed.
```

If **any** step shows unverified, the orchestrator refuses to mark the phase complete and reports what's missing.

### Key Innovation 4: Step Definitions per Phase

Each phase gets a machine-readable step registry. Example for brainstorm:

```json
{
  "phase": "brainstorm",
  "totalSteps": 6,
  "steps": [
    {
      "id": "1_understand_context",
      "name": "Understand Context",
      "inputs": ["package.json?", "tsconfig.json?", "existing code?"],
      "outputs": ["context_understood"],
      "verification": "Claude summarized existing project state",
      "references": []
    },
    {
      "id": "2_ask_questions",
      "name": "Ask Questions",
      "inputs": ["context_understood"],
      "outputs": ["requirements_gathered"],
      "verification": "At least 5 questions asked across categories",
      "references": [],
      "minQuestions": 5
    },
    {
      "id": "3_explore_approaches",
      "name": "Explore Approaches",
      "inputs": ["requirements_gathered"],
      "outputs": ["approaches_presented"],
      "verification": "2-3 approaches with trade-offs shown to user",
      "references": []
    },
    {
      "id": "4_present_design",
      "name": "Present Design",
      "inputs": ["approaches_presented", "user_choice"],
      "outputs": ["design_validated"],
      "verification": "200-300 word sections presented, user validated",
      "references": []
    },
    {
      "id": "5_generate_output",
      "name": "Generate Output",
      "inputs": ["design_validated"],
      "outputs": ["spec.md", "docs/plans/YYYY-MM-DD-{topic}-design.md"],
      "verification": "Both files exist and contain complete content",
      "references": ["brainstorm/docs/plans/YYYY-MM-DD-{topic}-design.md"]
    },
    {
      "id": "6_handoff",
      "name": "Handoff",
      "inputs": ["spec.md"],
      "outputs": ["pipeline-state.json updated"],
      "verification": "pipeline-state.json shows brainstorm completed",
      "references": []
    }
  ]
}
```

### Key Innovation 5: Smart Context Management

Instead of hoping Claude loads references at the right time, the orchestrator **explicitly manages what's in context**:

```
Phase: Implementation
  Step 1 (load_plan):     LOAD plan.md, fix-plan.md?, fix-test-plan.md?
  Step 2 (detect_stack):  LOAD package.json, tsconfig.json
  Step 3 (setup):         (no new loads)
  Step 4 (execute_batch): LOAD js-ts-best-practices.md
                          LOAD frameworks-web.md      (if web project)
                          LOAD databases-sql.md       (if SQL detected)
  Step 5 (quality_gate):  UNLOAD references (they served their purpose)
  Step 6 (self_review):   LOAD review-checklists.md (quick scan only)
  Step 7 (report):        UNLOAD all, present summary
```

This prevents the "context filled with stale references" problem.

### Key Innovation 6: Recovery From Any Point

```
RESUME PROTOCOL
───────────────
1. Read pipeline-state.json
2. Find currentPhase + currentStep
3. Load ONLY that phase's SKILL.md
4. Verify artifacts from completed steps still exist
   → If any missing: flag, ask user to re-run from that step
5. Load references needed for current step
6. Continue from exact interruption point
7. Announce: "Resuming [phase] at step N/M: [name]"
```

### Key Innovation 7: Cross-Phase Consistency Checks

After each phase transition, verify the chain hasn't broken:

```
TRANSITION CHECK: Planning → Implementation
─────────────────────────────────────────────
✓ spec.md exists and is non-empty
✓ plan.md exists and is non-empty
✓ plan.md references spec.md features
✓ plan.md has task breakdown with acceptance criteria
✓ All tasks have Files, Depends-on, Description fields
✓ No circular dependencies in task graph
→ CLEAR to start Implementation
```

---

## Implementation Strategy

### What Changes

| Component | Current | Super-YK |
|-----------|---------|----------|
| `pipeline-state.json` | Phase-level status | Step-level status + artifacts + verification flags |
| Router `SKILL.md` | Phase dispatcher | Strict state machine with step verification protocol |
| Phase `SKILL.md` files | Prose instructions | Same prose + step registry (JSON/YAML block at top) + pre/post conditions per step |
| Reference loading | "Load when skill instructs" | Explicit load/unload per step in step registry |
| Phase transitions | "Suggest next, wait for user" | Same + mandatory completion audit + transition check |
| Failure recovery | Phase-level (blocked/fix-plan) | Step-level resume + artifact existence verification |

### What Stays the Same

- All 6 phase SKILL.md content (the actual instructions)
- All reference files (unchanged)
- All example artifacts
- The "suggest and wait" interaction pattern
- The severity system and review checklists
- The plugin/marketplace structure

### File Changes Needed

1. **New file**: `plugins/yk-dev-pipeline/skills/orchestrator/SKILL.md` — The super-yk orchestrator (replaces or wraps the current router SKILL.md)
2. **New file**: `plugins/yk-dev-pipeline/skills/orchestrator/step-registry.json` — Machine-readable step definitions for all 6 phases
3. **New file**: `plugins/yk-dev-pipeline/skills/orchestrator/protocols.md` — Step execution protocol, completion audit, transition checks, resume protocol
4. **Modified**: Each phase `SKILL.md` gets a YAML step registry block at the top
5. **Modified**: Router `SKILL.md` delegates to orchestrator for enforcement
6. **Enhanced**: `pipeline-state.json` schema v2 with step-level tracking

### Estimated Scope

- ~1 new SKILL.md (orchestrator): ~300-400 lines
- ~1 step registry JSON: ~200 lines
- ~1 protocols reference: ~200 lines
- ~6 phase SKILL.md modifications: ~20-30 lines each (adding step registry headers)
- Router SKILL.md rewrite: ~100 lines changed

---

## Summary

The super-yk agent isn't a different system — it's **enforcement infrastructure** around the existing pipeline. The current skills are the "what to do." The super-yk agent adds the "prove you did it" layer:

1. **Step-level state** → know exactly where you are
2. **Pre/post verification** → can't skip steps
3. **Completion audits** → can't fake phase completion
4. **Smart context management** → load right references at right time
5. **Resume from any point** → survive conversation interruptions
6. **Transition checks** → artifact chain stays intact