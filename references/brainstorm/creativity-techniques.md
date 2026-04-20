# Brainstorm Creativity Techniques

Reference material for the brainstorm skill. These techniques surface non-obvious
requirements, challenge assumptions, and prevent premature convergence on a solution.

**Load this file when Step 4 (Explore Approaches) instructs you to.**

---

## Technique Selection Matrix

Pick 1-2 techniques based on project type. Don't use all of them — that would overwhelm
the user and slow down the brainstorm.

| Project Type | Recommended Techniques | Why |
|---|---|---|
| API / Backend Service | Pre-mortem + Stakeholder Role-Play | APIs fail in production in predictable ways; role-play surfaces auth/ops gaps |
| UI / Frontend App | Inversion + Stakeholder Role-Play | UX failures are subtle; inversion surfaces bad experiences; role-play covers accessibility |
| CLI Tool / Library | Inversion + Assumption Surfacing | Libraries fail when assumptions about usage context are wrong |
| Data Pipeline / ETL | Pre-mortem + Assumption Surfacing | Pipelines fail on edge case data; assumptions about data shape are often wrong |
| Full-Stack App | Pre-mortem + Stakeholder Role-Play + Assumption Surfacing | Complex systems need multiple lenses |
| Refactoring / Migration | Pre-mortem + Assumption Surfacing | Migrations fail when assumptions about existing state are wrong |

---

## Pre-Mortem

**Purpose:** Surface risks that optimistic planning misses by imagining failure first.

**How to run:**

Present this to the user:
> "Before we commit to an approach, let's do a quick pre-mortem. Imagine it's 6 months
> from now and this project has failed. What's the most likely reason?"

Then think through your own failure scenarios:

1. **Technical failures** — What technology choice could backfire? What dependency could
   become unmaintained? What performance bottleneck could emerge at scale?
2. **Scope failures** — What feature could balloon in complexity? What "simple" requirement
   could turn out to be an iceberg?
3. **Integration failures** — What external API could change? What assumption about data
   format could be wrong?
4. **User failures** — What could confuse users? What workflow could they find unacceptable?
5. **Operational failures** — What could break at 3am? What error could be undebuggable?

**Present 3-5 failure scenarios** ranked by likelihood. For each, note:
- What would go wrong
- How likely it is (high / medium / low)
- What design decision could prevent it

**Example:**
> "Here are the most likely failure modes I see:
> 1. **Redis goes down and takes the whole API with it** (high) — we're using Redis for
>    both caching and rate limiting. Design mitigation: graceful degradation when Redis
>    is unavailable.
> 2. **URL collision at scale** (medium) — 6-char base62 gives ~57B possibilities, but
>    birthday problem hits sooner. Design mitigation: collision retry with exponential
>    backoff.
> 3. **Click analytics becomes a bottleneck** (medium) — fire-and-forget writes could
>    overwhelm Redis on viral links. Design mitigation: batched writes or sampling."

---

## Inversion

**Purpose:** Surface design requirements by thinking about what would make the project terrible.

**How to run:**

Present this to the user:
> "Let's think backwards for a moment. If we wanted to make this project as frustrating
> as possible to use, what would we do?"

Then generate 5-7 "anti-requirements" — things that would make the project awful:

1. Think about the worst possible user experience
2. Think about the worst possible developer experience (for contributors/maintainers)
3. Think about the worst possible operational experience

**Invert each anti-requirement** into a concrete design constraint:

| Anti-requirement | Inverted constraint |
|---|---|
| "Error messages just say 'Something went wrong'" | Every error includes: what failed, why, and what to do next |
| "Configuration requires editing 12 environment variables" | Sensible defaults for everything; only require what's truly required |
| "The CLI takes 30 seconds to start" | Lazy-load heavy dependencies; startup time <500ms |
| "Breaking changes with every minor version" | Semantic versioning; deprecation warnings before removal |

**Present the inversions** as design constraints and ask:
> "These are constraints I'd derive from thinking about what would make this terrible.
> Any of these particularly important to you?"

---

## Stakeholder Role-Play

**Purpose:** Evaluate the design from multiple perspectives to catch blind spots.

**Personas** (pick 3-4 relevant to the project):

### End User
- "I just want to get my task done. Is this intuitive? Can I figure it out without docs?"
- Focus: UX flow, error messages, discoverability, learning curve
- Ask: "What's the first thing a new user sees? What happens when they make a mistake?"

### Attacker
- "How can I abuse this system? Where are the trust boundaries weak?"
- Focus: Auth bypass, injection, data exfiltration, resource exhaustion, privilege escalation
- Ask: "What's the most valuable thing I can steal/break? What's the easiest entry point?"

### On-Call Engineer (3am)
- "Something's broken and I've never seen this codebase. Can I figure out what's wrong?"
- Focus: Logging, error context, health checks, graceful degradation, runbooks
- Ask: "If this service returns 500s, what's my first debugging step? Is there a health endpoint?"

### New Contributor
- "I just cloned this repo. Can I understand the architecture and make a change safely?"
- Focus: Code organization, naming, documentation, test coverage, onboarding friction
- Ask: "Can I run the project locally in <5 minutes? Is the architecture self-documenting?"

### Business Stakeholder
- "Does this solve the actual business problem? Can we ship this to customers?"
- Focus: Feature completeness, compliance, scalability, time-to-market trade-offs
- Ask: "What's the MVP vs the full vision? What are we deferring and why?"

**How to present:**
> "Let me look at this design from a few different angles..."

Share 1-2 key observations per persona. Focus on **new insights** — things that
haven't been discussed yet. Don't repeat what's already covered.

---

## Assumption Surfacing

**Purpose:** Make hidden assumptions explicit so they can be validated or designed around.

**How to run:**

After the approach is tentatively chosen, list every assumption the design relies on:

**Categories:**
1. **User assumptions** — "Users have modern browsers", "Users are technical", "Users
   will read the docs", "Users have reliable internet"
2. **Data assumptions** — "Input data is well-formed", "Records are <1MB", "IDs are
   unique", "Timestamps are UTC"
3. **Infrastructure assumptions** — "Database is always available", "DNS resolves quickly",
   "We have <100ms network latency to the cache"
4. **Scale assumptions** — "We'll have <10K users", "Peak load is <100 RPS", "Dataset
   fits in memory"
5. **Dependency assumptions** — "This library is maintained", "This API is stable",
   "This service has <1s response time"
6. **Business assumptions** — "Requirements won't change significantly", "We don't need
   multi-tenancy yet", "This is internal-only"

**Present assumptions and challenge each:**

| # | Assumption | Confidence | If wrong... |
|---|-----------|------------|-------------|
| 1 | Users have modern browsers (ES2020+) | High | Need polyfills, larger bundle |
| 2 | Input records are <1MB | Medium | Need streaming parser, memory management |
| 3 | Peak load is <100 RPS | Low | Need caching layer, connection pooling |
| 4 | Redis is always available | Medium | Need graceful degradation path |

**For low-confidence assumptions:** Propose a design that works even if the assumption
is wrong, or flag it as an open question for the spec.

> "Here are the assumptions this design relies on. The ones I'm least confident about
> are #3 and #4 — should we design for those being wrong?"

---

## Diverge-Converge Process

**Purpose:** Prevent premature convergence by explicitly separating idea generation
from evaluation.

This isn't a standalone technique — it's the **structure** for Step 4 of the brainstorm.

### Diverge Phase (Generate)
- Generate 3-5 distinct approaches. Don't self-censor — include creative, unconventional
  options.
- For each approach, note the core insight (what makes it different from the others).
- Apply 1-2 creativity techniques (pre-mortem, inversion, role-play) to stretch thinking.
- **Rule: no evaluation during divergence.** Don't say "this one is better" yet.

### Converge Phase (Evaluate)
- Evaluate each approach against the requirements, constraints, and technique findings.
- Use a simple scoring matrix:

| Criterion | Approach A | Approach B | Approach C |
|-----------|-----------|-----------|-----------|
| Meets core requirements | ✓ / partial / ✗ | ... | ... |
| Handles failure modes (pre-mortem) | ... | ... | ... |
| Survives inversion constraints | ... | ... | ... |
| Stakeholder approval (role-play) | ... | ... | ... |
| Assumption robustness | ... | ... | ... |
| Implementation complexity | Low / Med / High | ... | ... |
| Maintenance burden | Low / Med / High | ... | ... |

- **Lead with your recommendation** — rank the approaches and explain why.
- Let the user decide, but be opinionated. "I recommend Approach B because it handles
  the failure modes we identified while keeping implementation simple."

### Assumption Surfacing (Post-Convergence)
- After an approach is chosen, run Assumption Surfacing on the chosen design.
- This is the final validation before moving to design presentation (Step 5).