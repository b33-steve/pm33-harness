---
name: harness-prep
description: "Orchestrator entry point for harness Phase 0. Invoke this BEFORE harness-planner whenever you are about to draft a harness plan. Sequences three sub-flows — discovery (always), brainstorming (always), and conditional external research — into a single enriched discovery doc that harness-planner consumes. Skipping this step is the root cause of 5-10x over-estimates in multi-workstream planning."
---

# Harness Prep Skill

**Role**: PHASE 0 ORCHESTRATOR — Shrink uncertainty before planning begins

**Purpose**: Orchestrate the three Phase 0 sub-flows (discovery, brainstorming, conditional research) into a single coherent "shrink uncertainty" phase. Produces a fully-prepared discovery doc that harness-planner consumes as its primary input.

---

## WORKFLOW POSITION

```
harness-prep          ← THIS SKILL — Phase 0 orchestrator (run before harness-planner)
   composes:
     harness-discovery          (always)
     superpowers:brainstorming  (always, unless skip-with-rationale)
     harness-research           (conditional — see Step 3 below)
   ↓ produces docs/dogfood/discovery/<slug>.md with up to 3 sections
harness-planner       ← Phase 1-3 — draft the plan (invoke after harness-prep completes)
   ↓ produces docs/dogfood/<slug>-sprint-plan.md
gauntlet-review       ← Phase 4 — parallel specialist review BEFORE shipping
   ↓ integrates findings into plan Appendix A
harness-coordinator   ← execute
```

Do not invoke harness-discovery, superpowers:brainstorming, or harness-research directly when prepping a harness. Invoke this skill instead — it handles the sequencing.

---

## WHEN TO USE THIS SKILL

**Invoke harness-prep any time you are about to draft a harness plan.** That means: before every invocation of `harness-planner` for a multi-workstream plan (3+ workstreams or 16+ hours total estimated effort).

**Skip conditions** (document the skip in the plan's Appendix B with rationale):
- Pure compliance or correctness fix where only one correct answer exists (e.g., security CVE remediation, schema drift fix)
- Truly greenfield work (new file in a new directory with zero neighbors)
- Re-running a prep that already completed in the same session

---

## STEP-BY-STEP WORKFLOW

### Step 1 — Discovery (ALWAYS)

Invoke harness-discovery to audit what already exists before estimating anything:

```typescript
Skill({ skill: "harness-discovery" })
```

Per harness-discovery's pattern, dispatch 1-3 parallel Explore agents covering the feature's main surface areas. Each agent reads relevant source files and reports what is built, partially built, or missing.

Output: `docs/dogfood/discovery/<slug>.md` — a `## Findings by area` section documenting the infrastructure baseline. This is the primary effort-calibration input for harness-planner.

Do not estimate effort before this file exists.

### Step 2 — Brainstorming (ALWAYS, unless skip-with-rationale)

Invoke brainstorming to generate 2-4 alternative framings of the problem before committing to one approach:

```typescript
Skill({ skill: "superpowers:brainstorming" })
```

Generate alternatives along axes like: scope (narrow vs. broad), architecture (extend existing vs. new service), phasing (all-at-once vs. incremental), and tooling (build vs. adopt library).

Append a `## Alternatives considered` section to the discovery doc with each alternative and a one-sentence rationale for why the chosen approach wins (or why no alternative is clearly better, leaving the decision to harness-planner).

**Why this is mandatory even when the approach seems obvious**: the 2026-05-21 planning incident showed that the "obvious" framing is often the expensive one. Five minutes of brainstorming consistently surfaces a 30-60% cheaper path that discovery alone doesn't reveal. Skipping it "to save time" is a false economy.

**Legitimate skip**: if the work is a compliance fix, CVE patch, or pure-correctness change with no architectural choices (only one way to fix it correctly), skip brainstorming and note the skip in Appendix B.

### Step 3 — Conditional External Research

Evaluate the research trigger checklist below. If triggered, invoke harness-research with a specific scoped question:

```typescript
Skill({ skill: "harness-research" })
```

Append a `## External research` section to the discovery doc. Include decay metadata (research date, sources, confidence level) so findings are not cargo-culted into future plans without re-verification.

**Research trigger checklist — run research if ANY of these are true:**
- The feature evaluates a library or SDK not already in `package.json`
- The feature touches a fast-moving domain: AI API patterns, security advisories, billing rules, compliance standards (SOC2, GDPR, HIPAA)
- The user explicitly asked for outside-codebase context ("what does the industry do for X?", "is there a standard pattern for Y?")
- harness-planner cannot write a credible spec for a sub-feature without knowing how an external API actually works
- The discovery doc flags a gap item as "unclear without external context"

**Skip research (default) if ALL of the following are true:**
- The work is a refactor of existing internal PM33 code with a known cause
- Bug fix where the root cause is already fully understood
- Obvious extension of an existing PM33 pattern (new Zod contract, new route, new migration following an existing template)
- Time-pressured situation where waiting for search results is not feasible
- harness-discovery already resolved all unknowns

**Budget (from harness-research):** default 1 search-specialist agent + 3 WebSearch calls. Hard ceiling: 2 agents + 6 WebSearch calls. Do not exceed the ceiling.

**Anti-pattern**: do not run research "just to be thorough." The default is no research. If you cannot name the specific question you need answered before invoking this step, skip it.

**Decision citation**: record the research decision (triggered or skipped, with the trigger condition or skip reason) in the discovery doc's `## External research` section header or in Appendix B of the eventual plan.

### Step 4 — Hand Off to Planner

With the enriched discovery doc complete, invoke harness-planner:

```typescript
Skill({ skill: "harness-planner" })
```

Pass the discovery doc path explicitly in the planner prompt:

```
Discovery findings (Phase 0 output — MANDATORY INPUT):
- Path: docs/dogfood/discovery/<slug>.md
- Key findings: [summarize what already exists vs. what needs building]
- Corrected effort baseline: [from discovery doc]
- Alternatives evaluated: [from ## Alternatives considered section]
- External research: [if applicable, cite the ## External research section] (omit this line if research was skipped — note skip rationale in plan's Appendix B)
```

harness-planner must use these findings to calibrate effort estimates. Estimates that ignore discovery findings are invalid.

---

## OUTPUT

A fully-prepared `docs/dogfood/discovery/<slug>.md` with up to 3 sections:

| Section | Source | Always present? |
|---|---|---|
| `## Findings by area` | harness-discovery | Yes |
| `## Alternatives considered` | superpowers:brainstorming | Yes (unless skip-with-rationale) |
| `## External research` | harness-research | Conditional — only when trigger checklist fires |

The discovery doc is the handoff artifact. harness-planner treats it as required input. A plan drafted without this doc is considered incomplete.

---

## ANTI-PATTERNS

- **Skipping brainstorming to save time**: the 5-minute cost compounds against the 50-hour savings from discovery. The ratio is 600:1. There is no reasonable case where skipping brainstorming saves net time on a multi-workstream plan.
- **Running external research without a specific scoped question**: broad "let me see what's out there" research produces low-signal noise. Name the question before invoking harness-research.
- **Conflating prep output with planner output**: harness-prep produces a discovery doc. harness-planner produces a sprint plan. They are separate artifacts. Do not write the sprint plan during prep.
- **Invoking harness-discovery, superpowers:brainstorming, or harness-research individually when you want a fully-prepared harness context**: invoke harness-prep. It handles the sequencing, ensures the sections are appended in the right order, and produces a single coherent handoff doc.
- **Starting effort estimates before Step 1 completes**: any number written before the discovery doc exists is fictional. The 5-10x over-estimates that birthed this skill came from planning without running discovery first.

---

## RELATED SKILLS

**This skill is the entry point. It composes:**
- **harness-discovery**: internal codebase audit (Step 1, always)
- **superpowers:brainstorming**: alternative framings (Step 2, default-always)
- **harness-research**: bounded external context (Step 3, conditional)

**This skill hands off to:**
- **harness-planner**: consumes the enriched discovery doc to draft the sprint plan (Phase 1-3)
- **gauntlet-review**: parallel specialist review of the completed plan (Phase 4)
- **harness-coordinator**: executes the approved plan across sessions
