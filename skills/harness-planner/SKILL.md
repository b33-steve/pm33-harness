---
name: harness-planner
description: "Create structured harness projects for complex multi-phase features (16+ hours, 3+ sessions). Use when planning competitive intelligence, major refactors, AI integrations, or security overhauls requiring systematic progress tracking and quality gates. REQUIRES harness-prep (Phase 0 orchestrator — runs discovery + brainstorming + conditional research) BEFORE this skill, and gauntlet-review (Phase 4) AFTER. Every feature in the plan MUST have an explicit (specialist, llmTier, requiredSkills) assignment — no exceptions. Frontend features MUST include `frontend-design:frontend-design`."
---

# Harness Planner Skill

**Role**: HARNESS PROJECT ARCHITECT - Creates comprehensive project structures for long-running agent work

**Purpose**: Transform complex feature requests into structured harness projects with phases, features, validation gates, and progress tracking.

---

## 🔄 MANDATORY WORKFLOW (updated 2026-05-29)

Harness planning is a **3-skill workflow**: prep → plan → review. Each phase is mandatory.

```
Prerequisite: harness-prep        ← orchestrates discovery + brainstorming + (conditional) research
   ↓ produces docs/dogfood/discovery/<slug>.md with up to 3 sections
Phase 1-3: harness-planner        ← THIS SKILL — draft the plan
   ↓ produces docs/dogfood/<slug>-sprint-plan.md
Phase 4: gauntlet-review          ← parallel specialist review BEFORE shipping
   ↓ integrates findings into plan Appendix A
[then] harness-coordinator        ← execute
```

**Why mandatory**: the 2026-05-21 incident showed that planning without Phase 0 prep produces 5-10x over-estimates (PAM-AUTO-HARNESS estimated at 10-12h, actual was 3-4h; PAM-GITHUB-PUBLISH estimated at 31-45h, actual was 0-22h). The 2026-05-29 architectural correction extracted Phase 0 orchestration into `harness-prep` so the planner's job is unambiguous (draft, given prepared context). harness-prep itself composes harness-discovery + superpowers:brainstorming + harness-research (conditional) — see `harness-prep` SKILL.md for the sequencing. Planning without Phase 4 ships plans with security blockers, performance bugs, and OUTCOMES-001-class test gaps.

**The composition is non-negotiable for multi-workstream plans (3+ workstreams or 16+ hours total).**

---

## 🎯 WHEN TO USE THIS SKILL

Use this skill when user requests:
- ✅ **Large features**: 16+ hours of work across 3+ sessions
- ✅ **Multi-phase projects**: Backend → Frontend → Integration → Testing
- ✅ **Complex integrations**: AI features, OAuth flows, competitive intelligence
- ✅ **Security overhauls**: Zero-trust, MFA, SSO implementations
- ✅ **Major refactors**: Authentication system, data pipeline redesign

**User phrases that trigger this skill**:
- "Plan a harness project for..."
- "Create structured project for competitive intelligence"
- "Need harness structure for AI-powered PRD generation"
- "Set up multi-session project for authentication overhaul"

**Do NOT use for**:
- ❌ Single-session features (<8 hours)
- ❌ Simple bug fixes or minor enhancements
- ❌ Already-started projects (use harness-coordinator instead)
- ❌ Without first invoking harness-prep (Phase 0 orchestrator)

---

## 📖 HOW TO USE THIS SKILL

### Prerequisite (MANDATORY) — Run harness-prep FIRST

Before drafting, invoke `harness-prep` to orchestrate discovery + brainstorming + conditional research:

```typescript
Skill({ skill: "harness-prep" })
```

harness-prep produces a fully-prepared `docs/dogfood/discovery/<slug>.md` with up to 3 sections (audit findings, alternatives considered, optional external research with decay metadata). harness-planner consumes this doc as input.

**Greenfield exception**: if the work is genuinely greenfield (new file in new directory, no neighbors) you may skip harness-prep — but document the decision in the plan's Appendix B with rationale.

### Phase 1-3 (this skill) — Draft the plan

**With discovery findings in hand, load this skill:**

```typescript
Skill({ skill: "harness-planner" })
```

**After loading, invoke harness-planner agent. IMPORTANT: include the discovery doc path in the prompt:**

```typescript
Task({
  subagent_type: "harness-planner-agent",
  description: "Create harness project structure",
  prompt: `Create comprehensive harness project for: [FEATURE DESCRIPTION]

Discovery doc (harness-prep output — MANDATORY INPUT):
- Path: docs/dogfood/discovery/<feature-slug>.md
- Key findings: [SUMMARIZE WHAT ALREADY EXISTS]
- Corrected effort baseline: [FROM DISCOVERY DOC]

Requirements:
- Business value: [WHY THIS FEATURE]
- Technical components: [BACKEND/FRONTEND/DB/INTEGRATION NEEDS]
- Success criteria: [MEASURABLE OUTCOMES]
- Constraints: [PM33-SPECIFIC REQUIREMENTS]

Generate complete harness structure including:
1. README.md with phase breakdown (use discovery doc to scope phases accurately)
2. pm33-agent-progress.json with feature checklist
3. init.sh with quality gates
4. pre-commit-validation.sh
5. claude-progress.txt template

Effort estimates MUST reflect discovery findings — do NOT estimate from scratch
when discovery shows a capability is already built or partially built.

🎯 MANDATORY — Specialist + LLM + Required Skills Assignment Per Feature:

Every feature in pm33-agent-progress.json MUST have ALL THREE:
  - "specialist":     one of [backend-architect, frontend-developer, database-admin,
                              security-auditor, ai-engineer, performance-engineer,
                              test-automator, api-documenter, ui-ux-designer,
                              general-purpose]
  - "llmTier":        one of [haiku, sonnet, opus]
  - "requiredSkills": array of skill names (may be [] if no auto-required skills)

Apply these rules in order (later rules override earlier):

  1. Pick specialist from task type using the matrix in SKILL.md.
  2. Pick llmTier from task complexity using the matrix in SKILL.md.
  3. Populate requiredSkills from the matrix "Required Skills" column for that
     specialist (default [] if matrix row says "(none auto-required)").
  4. POLICY OVERRIDE: if specialist=frontend-developer, force llmTier=opus
     AND ensure requiredSkills includes "frontend-design:frontend-design".
  5. POLICY OVERRIDE: if specialist=ui-ux-designer, ensure requiredSkills
     includes "frontend-design:frontend-design".
  6. POLICY OVERRIDE: if specialist=security-auditor, force llmTier=opus.
  7. POLICY OVERRIDE: if task touches auth/OAuth/RLS/tenant-isolation,
     ensure llmTier=opus AND list security-auditor in gauntlet specialists.

After generating the plan, audit it:
  - Count features by specialist; print the distribution
  - Count features by llmTier; report the actual breakdown for this harness
    (NO target — every harness is different; a CRUD-heavy harness may be 95%
    haiku, a research harness may have more sonnet)
  - Count features missing requiredSkills field (should be 0)
  - Count frontend features missing "frontend-design:frontend-design" (should be 0)
  - For each non-haiku feature, write a one-sentence justification next to
    its spec explaining why higher tier is needed (genuine cross-service
    decision point, architectural judgment, policy override, etc.)
  - If multiple non-haiku features have similar weak justifications ("complex
    logic"), the spec likely needs to be more specific — that's the cheaper
    fix than dispatching opus. Iterate on the spec first.

Report this audit in the README.md "Resource Plan" section so the user sees
the (specialist, llmTier, requiredSkills) breakdown — with justifications
for non-haiku assignments — before approving the harness.

Do NOT use the deprecated "agent" field. Use "specialist" + "llmTier" +
"requiredSkills" only.`
})
```

### Phase 4 (MANDATORY) — Run gauntlet-review BEFORE declaring plan complete

**After drafting the plan, dispatch a 4-5 specialist parallel review:**

```typescript
Skill({ skill: "gauntlet-review" })
// Dispatch parallel review agents per the gauntlet-review workflow
// Minimum specialist set for multi-workstream plans:
//   - backend-architect    (service design, MCP patterns)
//   - security-auditor     (auth, permissions, external surfaces)
//   - test-automator       (acceptance criteria coverage, regression)
// Add when applicable:
//   - ai-engineer          (if any LLM/embedding/AI feature)
//   - performance-engineer (if any cron/worker/scale concern)
//   - frontend-developer   (if any UI surface)
// Integrate findings into plan as Appendix A
```

**Why this is mandatory**: planning without gauntlet ships plans with security/perf/test blockers that cost 5-10x more to fix post-implementation. The 2026-05-21 gauntlet caught a stored-XSS pivot, a CAN-SPAM violation, an 80x perf bug, and an OUTCOMES-001-class test gap — none of which were obvious from the plan alone.

**Plan is NOT complete until Appendix A contains gauntlet findings and the plan has been re-sequenced (if needed) to address blockers.**

---

## 🏗️ WHAT THE HARNESS-PLANNER AGENT CREATES

### Core Artifacts

**1. README.md** - Project overview:
- Feature description and business value
- 3-6 logical phases with time estimates
- Success criteria and validation requirements
- Quick start commands for agent sessions
- Documentation references (PRDs, wireframes, technical guides)
- **Harness branch + worktree** (for traceability): branch will be `harness/<HARNESS_ID>`, worktree at `.claude/worktrees/harness-<HARNESS_ID>/`. The coordinator skill provisions these at session start — planner does NOT create them (separation of concerns: planner produces documents, coordinator owns environment).

**2. pm33-agent-progress.json** - Structured progress tracking:
```json
{
  "project": {
    "id": "utt-task-number",
    "name": "Feature Name",
    "estimatedHours": 32,
    "phases": [
      {
        "id": 1,
        "name": "Backend Foundation",
        "features": [
          {
            "id": "1.1",
            "description": "Create Zod contracts",
            "status": "pending",
            "estimatedHours": 1.5,
            "tddPhases": {
              "red": { "status": "pending" },
              "green": { "status": "pending" },
              "refactor": { "status": "pending" },
              "delivery": { "status": "pending" }
            }
          }
        ]
      }
    ]
  }
}
```

**3. init.sh** - Environment validation:
- Verify OrbStack services (PostgreSQL, Redis, App)
- Schema drift validation
- TypeScript strict mode check
- ESLint validation
- Test baseline establishment
- Blocks session start if critical gates fail

**4. pre-commit-validation.sh** - Quality gates:
- All validation checks from init.sh
- API contract validation
- Unit test coverage (95%+)
- Integration test execution
- Documentation updates verified

**5. claude-progress.txt** - Session log template:
```
Session 1 - [DATE]
Features Completed: 1.1, 1.2
TDD Phases: RED-GREEN-REFACTOR-DELIVERY
Validation Results: All gates passed
Next Feature: 1.3 - API route implementation (1.5h)
```

---

## 🔄 WORKFLOW AFTER HARNESS CREATED

**Once harness-planner agent completes:**

1. **Review the structure** generated in `docs/frameworks/agent-state-{PROJECT-ID}/`
2. **Verify artifacts** are complete:
   - [ ] README.md with clear phases
   - [ ] pm33-agent-progress.json with all features
   - [ ] init.sh with appropriate quality gates
   - [ ] pre-commit-validation.sh configured
   - [ ] claude-progress.txt template ready

3. **Start implementation** using harness-coordinator:
   ```typescript
   Skill({ skill: "harness-coordinator" })
   ```

4. **Coordinator workflow**:
   - Runs init.sh to validate environment
   - Reads pm33-agent-progress.json to find next feature
   - Launches specialist agent (backend-architect, frontend-developer, etc.)
   - Tracks progress and ensures quality gates pass

---

## 📋 PM33-SPECIFIC REQUIREMENTS

**The harness-planner agent ensures ALL harness projects comply with:**

### Zero Mock Data Policy
- No placeholder content or demo data
- Real data integration with proper empty states
- Environment variables for configuration
- CSS custom properties for styling

### Zod Contract-First Development
- API contracts created BEFORE route implementation
- Request/response validation with Zod schemas
- Frontend types mirroring backend contracts
- `@contract` comments on all route handlers

### MECE UI/UX Compliance
- Mutually Exclusive, Collectively Exhaustive interfaces
- Wireframe reference for every UI component (PM33_WIREFRAMES_v14.md)
- No emojis in UI components
- Mobile-first responsive design

### Strategic Context Integration
- PM33's core differentiator: strategy to execution enablement
- Features connect business plans to business outcomes
- Strategic alignment scoring where applicable
- ROI optimization and prioritization frameworks

### Security & Tenant Isolation
- Zero-trust architecture principles
- Tenant middleware on all multi-tenant endpoints
- Authentication validation
- Row-level security for database access

### Schema Change Protocol
- `shared/schema.ts` and migrations committed together
- Schema drift validation before AND after changes
- Multi-environment migration strategy (dev → staging → production)

---

## 🎓 EXAMPLE INVOCATIONS

### Example 1: Competitive Intelligence Feature

**User**: "I need to implement competitive intelligence discovery with data scraping, AI analysis, and strategic recommendations"

**Your response**:
```typescript
// Load harness-planner skill
Skill({ skill: "harness-planner" })

// Invoke harness-planner agent
Task({
  subagent_type: "harness-planner-agent",
  description: "Create competitive intelligence harness",
  prompt: `Create comprehensive harness project for Competitive Intelligence Discovery System.

Business Value:
- Enable PM teams to systematically track competitor products and features
- Provide AI-powered strategic recommendations based on market analysis
- Integrate competitive insights into product roadmap prioritization

Technical Components:
- Backend: Web scraping service, AI analysis endpoints, data aggregation
- Frontend: Competitive dashboard, insight cards, strategic recommendations UI
- Database: competitor_products, competitive_insights, analysis_results tables
- Integrations: OpenAI API, web scraping libraries, strategic alignment scoring

Success Criteria:
- Scrape and analyze 50+ competitor websites successfully
- Generate strategic recommendations with 85%+ relevance score
- Surface insights in unified dashboard with filtering and search
- Complete in 32-40 hours across 4-5 sessions

PM33-Specific Requirements:
- Strategic context integration (competitive insights influence roadmap)
- Zod contracts for all AI endpoints
- Wireframe compliance for dashboard (section TBD)
- Zero mock data - real competitor analysis from session 1`
})
```

### Example 2: Authentication System Overhaul

**User**: "We need to overhaul the authentication system to implement zero-trust with MFA, SSO, and session management"

**Your response**:
```typescript
Skill({ skill: "harness-planner" })

Task({
  subagent_type: "harness-planner-agent",
  description: "Create authentication overhaul harness",
  prompt: `Create comprehensive harness project for Zero-Trust Authentication System.

Business Value:
- Enterprise-grade security for SaaS customers
- SOC2 compliance requirement for enterprise deals
- Reduce unauthorized access incidents to zero

Technical Components:
- Backend: OAuth2.0 flows, MFA service, session management, JWT handling
- Frontend: SSO login flows, MFA setup wizard, session monitoring UI
- Database: mfa_credentials, oauth_tokens, session_logs tables
- Integrations: Google OAuth, Microsoft Azure AD, Twilio (SMS MFA)

Success Criteria:
- Support Google and Microsoft SSO with automatic account linking
- MFA enrollment required for admin users
- Session timeout and activity monitoring
- Zero authentication bypass vulnerabilities (security audit required)
- Complete in 40-48 hours across 5-6 sessions

PM33-Specific Requirements:
- Security-critical: All phases require security-auditor review
- No session/auth changes without architect approval (RED zone)
- Comprehensive audit logging (activity_feed integration)
- Tenant isolation preserved across all auth flows
- Zero downtime migration from current auth system`
})
```

### Example 3: AI-Powered PRD Generation

**User**: "Implement the AI-powered PRD generator with template selection, strategic context integration, and export functionality"

**Your response**:
```typescript
Skill({ skill: "harness-planner" })

Task({
  subagent_type: "harness-planner-agent",
  description: "Create PRD generator harness",
  prompt: `Create comprehensive harness project for AI-Powered PRD Generation System.

Business Value:
- Reduce PRD creation time from 8 hours to 30 minutes
- Ensure strategic alignment through automated context integration
- Generate comprehensive PRDs with competitive analysis and ROI projections

Technical Components:
- Backend: AI prompt engineering, template management, strategic context extraction
- Frontend: PRD editor with WYSIWYG, template library, export controls
- Database: prd_templates, prd_drafts, strategic_context_links tables
- Integrations: OpenAI API (GPT-4), strategy extraction service, export to PDF/Markdown

Success Criteria:
- Generate PRDs with 90%+ completeness score
- Strategic context automatically pulled from company strategy
- Export to PDF, Markdown, Confluence, Jira formats
- Complete in 24-32 hours across 3-4 sessions

PM33-Specific Requirements:
- Strategic context integration (PM33's core differentiator)
- Zod contracts for AI endpoints and template management
- Wireframe compliance for PRD editor (PM33_WIREFRAMES_v14.md lines 666-800)
- Real template library - no mock templates
- Zero emojis in generated PRDs (MECE compliance)`
})
```

---

## 🎯 MANDATORY: Specialist + LLM Assignment Per Feature

**Every feature MUST be assigned BOTH a specialist agent type AND an LLM tier.** A plan is INCOMPLETE if any feature is missing either. The harness-planner-agent prompt and success criteria below enforce this.

### Why two fields, not one

The legacy `"agent": "haiku|sonnet|opus"` field conflated two orthogonal concerns:
- **Specialist** = WHO does the work (which Claude Code agent type — `backend-architect`, `frontend-developer`, etc.)
- **LLM tier** = WITH WHAT MODEL (haiku/sonnet/opus, based on task complexity)

A backend-architect can run on haiku for a well-defined CRUD endpoint OR opus for a novel service skeleton — same specialist, different tier. Conflating them loses signal and forces the planner to pick the wrong axis. Split them.

### Specialist Selection Matrix

| Task Type | Specialist | Required Skills | Notes |
|---|---|---|---|
| Backend service / API route / data model | `backend-architect` | (none auto-required) | Default for server-side work |
| Frontend component / page / UX implementation | `frontend-developer` | **`frontend-design:frontend-design` MANDATORY** | **ALWAYS opus** (PM33 policy) |
| Database schema / migration / RLS policy | `database-admin` | (none auto-required) | |
| Security review / auth / OAuth / scope check | `security-auditor` | (none auto-required; consider `senior-security`) | **Typically opus** (high-stakes) |
| AI / LLM / embedding / RAG / prompt engineering | `ai-engineer` | (none auto-required; consider `llm-application-dev:*`) | Rarely haiku |
| Performance / scaling / load / query optimization | `performance-engineer` | (none auto-required) | |
| Test strategy / coverage / regression | `test-automator` | (none auto-required) | |
| API documentation / contract spec | `api-documenter` | (none auto-required) | |
| UI/UX design / wireframe compliance | `ui-ux-designer` | **`frontend-design:frontend-design` MANDATORY** | Pairs with frontend-developer |
| Generic / cross-cutting / mixed | `general-purpose` | (none auto-required) | Use sparingly — prefer specialists |

The "Required Skills" column lists skills that **MUST be loaded by the implementing agent** before starting work on the feature. The harness-planner-agent populates the feature's `requiredSkills` array based on this matrix; the harness-coordinator ensures they're loaded at dispatch time.

### LLM Tier Selection Matrix

**Core principle: spec specificity drives tier choice. The more pre-decided the specification, the smaller the model needed.** The planner's job is to *distill complexity into the spec*, not leave it for the agent. If a task feels like it needs sonnet or opus, the first question to ask is: "can I write a more specific spec to make this haiku-doable?" — usually the answer is yes.

| Tier | Use when the SPEC contains... | Implication |
|---|---|---|
| **haiku** | Complete algorithm/pseudocode, full type signatures, exact file paths, explicit test cases with inputs+outputs, ALL design decisions pre-baked. Agent is translating, not deciding. | Default for well-specified work |
| **sonnet** | Algorithm outline but 1-2 genuine design decisions left to the agent; OR multi-file coordination where the agent must thread context across services it doesn't have full visibility into | Use when raising spec quality further isn't feasible (genuine cross-service unknowns) |
| **opus** | Interface contracts + design constraints but the implementation requires architectural judgment; OR security-critical work where review-as-you-write matters; OR frontend (per policy override below) | Should be **rare** — reserved for genuine architecture, novel algorithms, security reviews, or PM33's frontend-quality requirement |

### Policy Overrides (NON-NEGOTIABLE)

These overrides come from existing PM33 memory and CLAUDE.md preferences. They **always win** over the matrix:

| Rule | Source | Rationale |
|---|---|---|
| `specialist=frontend-developer` → `llmTier=opus` ALWAYS | Memory: `feedback-frontend-opus-only.md` | PM33 frontend quality bar requires opus output even for "mechanical" UI tasks |
| `specialist=frontend-developer` → `requiredSkills` MUST include `frontend-design:frontend-design` | PM33 frontend quality bar; avoids generic AI aesthetics | The skill produces distinctive, production-grade frontend code; without it, frontend output drifts toward generic AI defaults that fail PM33's UX bar |
| `specialist=ui-ux-designer` → `requiredSkills` MUST include `frontend-design:frontend-design` | Same as above | UX designers using PM33's standards need the same skill that frontend-developers do |
| `specialist=security-auditor` → `llmTier=opus` ALWAYS | High-stakes review; ai-engineer review of plan-time security findings | Security errors compound; cost of opus is dwarfed by cost of a CAN-SPAM violation |
| Any feature touching auth / OAuth / RLS / tenant-isolation → require `security-auditor` review (separate sub-feature OR gauntlet specialist) | CLAUDE.md §4 "SECURITY IS NON-NEGOTIABLE" | All security-critical paths need a specialist pass, not just self-review |

### No distribution targets — pick what's right per task

There is **no target distribution** (no "70% haiku / 20% sonnet / 10% opus" rule). Every harness is different. A well-specified CRUD-heavy harness might be 95% haiku; a research-driven AI harness might be 50% sonnet. The right distribution emerges from the work, not from a quota.

What matters:

- **Each tier assignment must be justified by the spec it sits beside.** If a feature is assigned sonnet, the spec should visibly require multi-file context or have a decision point. If opus, the spec should describe interface contracts requiring architectural judgment. A reviewer should be able to look at any (tier, spec) pair and agree the tier matches the spec quality.
- **Opus should be the exception, not the rule.** Most harness work is well-specified by design — that's the *whole point* of writing a harness. The complexity is supposed to be distilled into the spec by the planner, not handed to the agent as architectural homework. If you find yourself assigning opus to many features, the question is usually: "are my specs detailed enough?" — not "am I picking the right tier?"
- **The only mandatory opus uses are policy overrides**: frontend-developer (always), security-auditor (always), auth/OAuth/RLS work (always). Everything else: justify per-task.

Reference data points for calibration (not targets):
- SCHED-V3 Phase 0: 12 features, 100% haiku — very dense specs, narrow domain
- METRIC-ALIGN-001: 55 features mixed across all three tiers — broader scope, multiple research phases
- A typical CRUD-heavy backend harness with strong specs: 80-95% haiku
- A research/discovery harness with novel algorithms: more sonnet, occasional opus

Use these as ground truth that distributions vary widely. Don't try to match them.

### How the assignment shows up in the plan

Every feature row in `pm33-agent-progress.json` must include three fields explicitly:

```json
{
  "id": "X.Y",
  "specialist": "frontend-developer",
  "llmTier": "opus",
  "requiredSkills": ["frontend-design:frontend-design"],
  ...
}
```

For features where no skill auto-requires (most backend tasks), the array is empty:

```json
{
  "id": "X.Y",
  "specialist": "backend-architect",
  "llmTier": "haiku",
  "requiredSkills": [],
  ...
}
```

The legacy `"agent": "haiku"` field is **deprecated** — new harnesses should not use it. If present for backward compatibility, the coordinator interprets it as `llmTier` only, and `specialist` defaults to `general-purpose` (with a warning logged).

### Mini-decision tree

```
Is this a frontend task?
  YES → specialist=frontend-developer
       → llmTier=opus (policy override)
       → requiredSkills MUST include "frontend-design:frontend-design"
  NO ↓
Does it touch auth/OAuth/RLS/tenant isolation?
  YES → specialist=security-auditor OR add security-auditor to gauntlet review
       → llmTier=opus (policy override)
  NO ↓
Pick specialist from matrix by task type.
  ↓
requiredSkills from matrix "Required Skills" column (default [])
  ↓
LLM tier — driven by SPEC specificity, not a quota:
  Spec has full algorithm/signatures/test cases, all decisions baked in?
    YES → haiku
    No — spec has 1-2 design decision points or genuine cross-service unknowns?
    → can you write a more specific spec? If yes, do that first, re-evaluate.
    → if no (decisions genuinely require runtime context), use sonnet
    Genuine architectural judgment / novel algorithm / security review?
    → write a more specific spec first; if still architectural, use opus
```

The single best question to ask before assigning sonnet/opus: **"can I make this spec specific enough that haiku could handle it?"** Usually yes. Opus should be uncommon.

---

## 🔧 FEATURE SPECIFICATION QUALITY STANDARDS - MANDATORY

Every feature in pm33-agent-progress.json MUST include these fields. This standard was proven on SCHED-V3 Phase 0 (12 features, 266 tests, 100% haiku execution rate with zero escalations).

### 7 Required Specification Fields

Every feature's `specification` object MUST contain ALL of these:

| Field | Purpose | Example |
|-------|---------|---------|
| `files` | Exact file paths to create/modify | `["server/services/roadmap/utils/PriorityBandBuilder.ts"]` |
| `functionSignature` | Full TypeScript signature with types | `"export function buildPriorityBands(epics: BandableEpic[]): PriorityBand[]"` |
| `algorithm` | Step-by-step logic or pseudocode | Complete algorithm the agent can translate directly to code |
| `designDocRef` | Exact line numbers in design doc | `"lines 134-145 (priority bands with 10% tolerance)"` |
| `imports` | Required import statements | `["classifyEpicStatus from './StatusClassification'"]` |
| `testCases` | Specific test scenarios with expected outputs | `["Given [95,92,85], returns 2 bands: [95,92], [85]"]` |
| `acceptanceCriteria` | Measurable completion criteria (top-level field) | `["All 15+ known statuses classified correctly"]` |

### Model-Tiered Specification Depth

> See "MANDATORY: Specialist + LLM Assignment Per Feature" section above for the matrices
> and policy overrides. This section describes the *specification depth* required per tier —
> i.e., how much detail the planner must provide for an agent at that tier to succeed.

**HAIKU features** (routine, well-defined — target 70-80% of features):
- `algorithm` field contains **COMPLETE executable pseudocode** — the agent translates it directly to TypeScript
- ALL architectural decisions pre-made (no choices left to the agent)
- Exact interface/type definitions included in the algorithm
- Every import path specified
- Test cases include specific inputs AND expected outputs
- Example:
```json
{
  "agent": "haiku",
  "specification": {
    "algorithm": "export interface BandableEpic { id: string; compositePriority: number; storyPoints: number; isLocked: boolean; }; function buildPriorityBands(epics: BandableEpic[]): PriorityBand[] { const schedulable = epics.filter(e => !e.isLocked); const bands: Map<number, BandableEpic[]> = new Map(); for (const epic of schedulable) { const bandIndex = Math.floor(epic.compositePriority / 10); if (!bands.has(bandIndex)) bands.set(bandIndex, []); bands.get(bandIndex)!.push(epic); } return sortedBandKeys.map(key => ({ bandIndex: key, minPriority: key * 10, maxPriority: key * 10 + 9, epics: bands.get(key)!.sort((a,b) => b.storyPoints - a.storyPoints) })); }",
    "testCases": [
      "Given epics with priorities [95, 92, 85, 82, 71], returns 3 bands: [95,92], [85,82], [71]",
      "Given all locked epics, returns empty array",
      "Given empty array, returns empty array"
    ]
  }
}
```

**SONNET features** (moderate complexity — multi-file coordination, state machines):
- `algorithm` provides **algorithm outline with key decision points marked**
- Integration points with other services specified
- State transitions documented if applicable
- Performance requirements included
- Example: Service integration, API contract wiring, multi-step data pipelines

**OPUS features** (architecture decisions — service skeletons, complex algorithms):
- `algorithm` provides **interface contracts, design constraints, and architectural patterns**
- Lists all DI dependencies and their roles
- Specifies which patterns to use (strategy, observer, etc.)
- Defines performance bounds and convergence criteria
- Example: HybridSchedulerService skeleton, LocalSearchOptimizer with 5 move types

### Cross-Phase Dependency References

Every feature MUST declare dependencies on prior phase features:
```json
{
  "id": "1.7",
  "dependencies": ["0.4", "0.8", "1.1"],
  "specification": {
    "imports": [
      "classifyEpicStatus from '../utils/StatusClassification'",
      "loadWorkItemHierarchy from '../HierarchyLoader'",
      "buildPriorityBands from '../utils/PriorityBandBuilder'"
    ]
  }
}
```

This enables the coordinator to:
1. Build a dependency graph for wave-based parallel execution
2. Verify all dependencies are complete before launching an agent
3. Include dependency outputs in the agent's context

### Feature JSON Template

```json
{
  "id": "X.Y",
  "description": "Clear description of what this feature produces",
  "status": "pending",
  "estimatedHours": 1.5,
  "dependencies": ["prior.feature.ids"],
  "specialist": "backend-architect",
  "llmTier": "haiku",
  "requiredSkills": [],
  "tddPhases": {
    "red": { "status": "pending", "testFile": "path/to/test.ts" },
    "green": { "status": "pending", "implementation": "path/to/file.ts" },
    "refactor": { "status": "pending", "optimizations": [] },
    "delivery": { "status": "pending", "validationCommands": ["npm run test:locked -- test.ts"] }
  },
  "acceptanceCriteria": [
    "Measurable criterion with specific numbers/behaviors",
    "Edge case handling specified",
    "Performance requirement if applicable"
  ],
  "specification": {
    "files": ["exact/file/paths.ts"],
    "functionSignature": "export function name(params: Types): ReturnType",
    "algorithm": "Complete pseudocode for haiku | outline for sonnet | contracts for opus",
    "designDocRef": "lines X-Y (section name)",
    "imports": ["import { X } from 'module'"],
    "testCases": ["Given X, expect Y"]
  },
  "validationGates": ["npm run test:locked -- test.ts"]
}
```

**Example: frontend feature** (note the mandatory `frontend-design:frontend-design`):

```json
{
  "id": "3.2",
  "description": "Build VOC triage queue UI with filter/sort and propose-action buttons",
  "specialist": "frontend-developer",
  "llmTier": "opus",
  "requiredSkills": ["frontend-design:frontend-design"],
  ...
}
```

**Required fields, no exceptions**: `specialist`, `llmTier`, AND `requiredSkills` (all three; the latter may be `[]` for tasks with no auto-required skills). The legacy combined `"agent": "haiku|sonnet|opus"` field is deprecated — see Specialist + LLM Assignment section above.

**Validation hint**: a quick way to audit a harness plan is:

```bash
# Find features missing required fields
jq '.project.phases[].features[] | select(.specialist == null or .llmTier == null or .requiredSkills == null) | .id'

# Find frontend features missing the mandatory frontend-design skill
jq '.project.phases[].features[]
    | select(.specialist == "frontend-developer" or .specialist == "ui-ux-designer")
    | select((.requiredSkills // []) | index("frontend-design:frontend-design") | not)
    | .id'
```

Both should return zero IDs. The harness-coordinator may refuse to dispatch features missing these fields or violating policy.

---

## ✅ SUCCESS CRITERIA

**A successful harness project includes:**

- [ ] **harness-prep completed**: `docs/dogfood/discovery/<slug>.md` exists with findings, alternatives, and optional external research
- [ ] **Phase 4 gauntlet completed**: Plan's Appendix A contains 4-5 specialist findings, blockers addressed
- [ ] **Clear phases**: 3-6 logical phases with realistic time estimates (informed by discovery)
- [ ] **Granular features**: Each phase has 3-6 features (1-2 hours each)
- [ ] **Quality gates**: init.sh and pre-commit-validation.sh configured
- [ ] **Progress tracking**: pm33-agent-progress.json with TDD phase tracking
- [ ] **7/7 spec fields**: Every feature has all 7 required specification fields
- [ ] **(specialist, llmTier, requiredSkills) trio on EVERY feature**: zero features missing any of the three fields
- [ ] **Specialist matches task type**: frontend tasks → frontend-developer, security tasks → security-auditor, etc.
- [ ] **LLM tier matches complexity**: haiku for pseudocode-translatable, sonnet for multi-file coordination, opus for architectural/security/frontend
- [ ] **Policy overrides applied**: frontend-developer ALWAYS opus; security-auditor ALWAYS opus; auth/OAuth/RLS ALWAYS opus
- [ ] **`frontend-design:frontend-design` on EVERY frontend feature**: any feature with `specialist` in {`frontend-developer`, `ui-ux-designer`} MUST have `frontend-design:frontend-design` in its `requiredSkills` array — zero exceptions
- [ ] **Each non-haiku tier assignment is justified by its spec**: for every sonnet/opus feature, the planner can point to specific spec characteristics (multi-service decision point, architectural judgment, policy override) that warrant the higher tier — NOT a distribution target. Default-haiku is the principle; sonnet/opus needs a reason.
- [ ] **Tier breakdown reported in README.md "Resource Plan"**: not against a target, but visible so user can audit whether each non-haiku assignment is well-justified
- [ ] **Model-tiered depth**: Haiku features have COMPLETE pseudocode; sonnet have outline + decision points; opus have interfaces + constraints
- [ ] **Cross-phase deps**: All features declare dependencies on prior phases
- [ ] **Documentation references**: Links to PRDs, wireframes, technical guides, discovery doc
- [ ] **PM33 compliance**: All mandatory standards incorporated
- [ ] **Success criteria**: Measurable outcomes defined upfront
- [ ] **Decision log (Appendix B)**: Decisions emerging from gauntlet documented with rationale

---

## 🔗 RELATED SKILLS

**Mandatory composition (in order)**:
- **harness-prep** (Prerequisite): orchestrates discovery + brainstorming + conditional external research; produces enriched discovery doc
- **harness-planner** (this skill, Phase 1-3): draft the plan informed by the prep output
- **gauntlet-review** (Phase 4): parallel specialist review BEFORE shipping plan

**After plan is complete**:
- **harness-coordinator**: orchestrate multi-session implementation
- **harness-discipline**: TDD discipline enforcement for specialists

**Skills composed by harness-prep** (you don't invoke these directly when prepping a harness — invoke harness-prep instead):
- harness-discovery (internal audit, always)
- superpowers:brainstorming (alternatives, always)
- harness-research (external context, conditional)

**For specific work types**:
- **utt-worker**: Execute UTT tasks with harness integration
- **technical-debt-sync**: Document architectural issues during harness work

---

## 📚 REFERENCE DOCUMENTATION

**Harness Framework**:
- `/docs/frameworks/LONG_RUNNING_AGENT_FRAMEWORK.md` - Comprehensive guide
- `/docs/frameworks/HARNESS_PROJECT_TEMPLATE.md` - Template for new projects
- `/docs/frameworks/agent-state-utt-strat-001/` - Reference implementation

**PM33 Standards**:
- `@CLAUDE.md` - Mandatory development standards
- `@documents/v15/PRD_EXECUTIVE_SUMMARY.md` - Strategic context
- `@docs/current/design/PM33_WIREFRAMES_v14.md` - UI/UX specifications

**Quality Gates**:
- `/docs/QA_RUNTIME_ERROR_PREVENTION.md` - Validation framework
- `/API_GOVERNANCE_FRAMEWORK.md` - API modification protocols
- `/docs/SCHEMA_CHANGE_WORKFLOW.md` - Schema synchronization

---

## 🚨 IMPORTANT NOTES

**Harness Planner Creates Structure ONLY**:
- ✅ Generates project artifacts (README, progress.json, scripts)
- ✅ Defines phases, features, and validation gates
- ✅ Documents requirements and success criteria
- ❌ Does NOT implement features
- ❌ Does NOT write code or tests
- ❌ Does NOT execute the harness project

**Implementation Happens After Planning**:
1. **Planner** creates structure → outputs to `docs/frameworks/agent-state-{ID}/`
2. **Coordinator** manages sessions → delegates to specialists
3. **Specialists** implement features → TDD cycle, validation, commits

**Use This Skill When**:
- User describes a complex feature requiring structure
- Need to plan multi-phase implementation
- Want systematic progress tracking across sessions
- Feature benefits from quality gates and validation

**Don't Use This Skill When**:
- Harness already exists (use harness-coordinator instead)
- Feature is simple and fits in one session
- Just need to coordinate existing harness work