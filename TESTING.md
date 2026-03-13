# Testing Skills

This repo has no executable code — "testing" means running skills in real Claude
conversations and verifying they follow instructions and produce quality output.

## When to Test

Test after any change to:
- A SKILL.md file (process changes, new steps, removed steps)
- A reference file (new patterns, changed checklists, updated examples)
- The Router SKILL.md (routing logic, intent detection, continuation logic)
- Example artifacts (they inform users and may be referenced by skills)

## How to Test

### 1. Pick Test Prompts

Each skill has **example test prompts** below. Use at least one prompt per changed
skill. For significant changes, test 2-3 prompts covering different project types.

### 2. Run the Skill

Start a new Claude conversation with the plugin installed. Give the test prompt.
Observe whether Claude:
- Loads the correct SKILL.md and references
- Follows every step in order (no skips)
- Produces all required output artifacts
- Meets the output quality criteria (see per-skill criteria below)

### 3. Before/After Comparison

When evaluating a skill change:
1. **Baseline** — run the test prompt on the old version, save the output
2. **Changed** — run the same prompt on the new version, save the output
3. **Compare** — check quality criteria. Did the change improve, maintain, or degrade?
4. **Regression check** — verify unchanged behaviors still work (use the regression
   markers below to identify what to check)

### 4. Model Matrix

Test each skill on its `recommended_model`. If time permits, also test on one level
down to verify graceful behavior.

| Skill | Recommended | Also test on |
|-------|-------------|-------------|
| Brainstorm | opus | sonnet |
| Investigation | opus | sonnet |
| Planning | sonnet | — |
| Implementation | opus | sonnet |
| Code Review | opus | sonnet |
| Testing | sonnet | — |
| Documentation | sonnet | — |
| Router | sonnet | — |

---

## Test Prompts by Skill

### Router
| Prompt | Tests |
|--------|-------|
| "Build me a URL shortener API" | Routes to brainstorm (Rule 2) |
| "This API is returning 500 errors" | Routes to investigation (Rule 3) |
| "Continue" (with pipeline-state.json) | Continuation logic table |
| "Review my code" | Direct phase routing (Rule 4) |
| "Check health" | Pipeline health check |
| "Start over" | Pipeline reset |

### Brainstorm
| Prompt | Tests |
|--------|-------|
| "Build me a URL shortener" | Full flow: context → research → questions → diverge-converge → spec |
| "Build me a CLI tool for converting images" | Different project type adaptation, YAGNI |
| "I have an idea for a real-time chat app" | Web app type, security/threat modeling depth |

### Investigation
| Prompt | Tests |
|--------|-------|
| "This endpoint is returning stale data" | Bug investigation: reproduce → hypotheses → root cause |
| "This code is messy, help me refactor" | Refactoring analysis, scope assessment |
| "Our API is slow under load" | Performance investigation, metrics-based analysis |

### Planning
| Prompt | Tests |
|--------|-------|
| (After brainstorm produces spec.md) "Let's plan" | Full flow: read spec → decompose → task breakdown → test modes |
| Provide a hand-written spec.md, say "Plan this" | Handles external spec, asks about gaps |

### Implementation
| Prompt | Tests |
|--------|-------|
| (After planning produces plan.md) "Start implementing" | Full flow: plan review → batches → quality gates → security gate → self-review |
| "Continue" (with last_completed_task in state) | Resume mode detection |
| (With fix-plan.md) "Fix the review issues" | Fix mode, finding outcome tracking |

### Code Review
| Prompt | Tests |
|--------|-------|
| (After implementation) "Review this code" | Full review: security pass → file-by-file → report |
| "Review my code" (standalone, existing codebase) | Works without spec/plan (graceful degradation) |
| (After fix cycle) "Re-review" | Re-review protocol, finding outcomes, statistics |

### Testing
| Prompt | Tests |
|--------|-------|
| (After code review approved) "Write tests" | Full flow: detect stack → test plan → write → run → report |
| "Write tests for this" (standalone) | Works without pipeline context |
| (After fix-test-plan.md) "Continue" | Re-test mode, regression check |

### Documentation
| Prompt | Tests |
|--------|-------|
| (After testing) "Generate docs" | Full flow: research → generate → verify → coverage → report |
| "Update the docs" (with existing docs) | Update mode detection, drift analysis |
| "Are the docs up to date?" | Drift detection (standalone) |

---

## Output Quality Criteria

### Brainstorm → spec.md
- [ ] Every feature has behavior description + inputs/outputs
- [ ] Threat model exists with abuse scenarios and data classification
- [ ] Out-of-scope section exists (YAGNI applied)
- [ ] Tech stack specified with reasoning
- [ ] Open questions section lists genuinely unresolved items (not vague placeholders)
- [ ] Design reference links to the full design doc

### Planning → plan.md
- [ ] Every task has files, dependencies, acceptance criteria
- [ ] Test mode assigned to every task (tdd/test-after/no-test)
- [ ] TDD tasks have test case sketches
- [ ] Dependency graph is acyclic (no circular task dependencies)
- [ ] Security Requirements section exists
- [ ] File structure matches spec's architecture

### Implementation → code
- [ ] All tasks completed with acceptance criteria verified
- [ ] Quality gates pass (types + build + lint + tests)
- [ ] Security gate pass (npm audit + manual checks)
- [ ] Self-review completed (no unaddressed findings)
- [ ] TDD tasks have test files alongside source files
- [ ] Checkpoint commits exist (one per task)
- [ ] pipeline-state.json updated with all fields

### Code Review → review.md + fix-plan.md
- [ ] Every finding has severity + file:line + code snippet + impact + suggested fix
- [ ] No vague findings ("this could be better" — must say what and why)
- [ ] Security pass results documented
- [ ] Plan deviations classified (justified/acceptable/problematic)
- [ ] Fix plan is ordered (critical → major → minor) with verify + regression steps
- [ ] Verdict matches finding severity (critical findings = block, not approve)
- [ ] Doc impacts listed in cross-step dependencies

### Testing → test-report.md
- [ ] Test plan presented before writing
- [ ] Existing TDD tests not duplicated
- [ ] All tests pass (or failures documented with root cause)
- [ ] Coverage reported with per-file breakdown
- [ ] Production bugs found are documented with fix
- [ ] Review cross-reference classifies bugs as review-validated or review-miss
- [ ] Security tests exist (for projects with HTTP endpoints)

### Documentation → docs + doc-report.md
- [ ] README has working commands (verified by running them)
- [ ] Every env var in docs exists in code
- [ ] Every endpoint in docs exists in routes
- [ ] Doc coverage analysis completed (public exports, endpoints, env vars)
- [ ] No aspirational content (only documents what exists)
- [ ] Cross-reference validation traces claims to file:line

---

## Regression Markers

When changing a skill, check which quality criteria are affected:

| Change area | Check criteria |
|-------------|---------------|
| Security (gate, patterns, review pass) | Implementation #3-4, Code Review #1-3, Testing #7 |
| TDD (test modes, red-green-refactor) | Planning #2-3, Implementation #5, Testing #2 |
| Context management (session split, resume) | Implementation #7, all skills' state handling |
| Review learning (outcomes, statistics) | Code Review #4-6, Testing #6 |
| Documentation sync (coverage, drift) | Documentation #3-6 |
| Pipeline robustness (validation, recovery) | All skills' prerequisite and state handling |
| Creativity techniques (diverge-converge) | Brainstorm #1-5 |
| Reference file changes | Corresponding skill's full quality criteria |