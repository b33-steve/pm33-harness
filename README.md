# PM33 Harness for Claude Code

**Put your Claude Code in a harness.** Free harness for any Claude Code
project — outcome-attribution and the rest of the closed loop requires
a [PM33 subscription](https://pm-33.com/pricing).

This repo is the source of the bundle distributed at
[pm-33.com/install](https://pm-33.com/install). Every Monday morning a
weekly auto-rebuilder pulls the latest skills/agents/frameworks from
PM33's internal sources and re-publishes the tarball at
`https://pm-33.com/install/pm33-bundle.tar.gz`. This repo is the
git-browseable mirror of that bundle.

## Two ways to install

### One-line (recommended)

```bash
curl -fsSL https://pm-33.com/install/pm33-init.sh | bash
```

Read the install docs at [pm-33.com/install](https://pm-33.com/install)
before piping to bash if you want to inspect first.

### From this repo (for forks, air-gapped, or contribution)

```bash
git clone https://github.com/b33-steve/pm33-harness.git
cd pm33-harness
./install.sh
```

## What's in this bundle

```
agents/                       → ~/.claude/agents/
  ── PM33-authored ──
  harness-coordinator.md      Orchestrates multi-session harness work
  harness-planner-agent.md    Breaks complex projects into phased tasks

  ── Specialists (wshobson/agents, MIT) ──
  ai-engineer.md              Production LLM apps, RAG, multimodal AI
  api-documenter.md           OpenAPI 3.1, SDK generation, dev portals
  backend-architect.md        REST/GraphQL/gRPC, microservices, scaling
  code-reviewer.md            AI-assisted review, security, performance
  database-admin.md           PostgreSQL, schema, RLS, ops
  frontend-developer.md       React 19, Next.js 15, accessibility
  performance-engineer.md     OpenTelemetry, load testing, Web Vitals
  security-auditor.md         OWASP, OAuth, compliance frameworks
  test-automator.md           AI-powered test automation, CI/CD
  ui-ux-designer.md           Design systems, wireframes, a11y
  LICENSE.wshobson            MIT license covering the 10 specialists

skills/                       → ~/.claude/skills/
  pm33-mcp/                   PM33 MCP tool conventions — LOAD BEFORE any
                              pm33_* tool call. Routing, known gaps,
                              batch patterns, queue-and-execute on MCP
                              instability.

  ── Phase 0: shrink uncertainty before planning ──
  harness-prep/               Orchestrator entry point. Sequences
                              discovery + brainstorming + conditional
                              research into one enriched doc that
                              harness-planner consumes. Run BEFORE
                              harness-planner — skipping it is the root
                              cause of 5-10x over-estimates in
                              multi-workstream planning. Requires the
                              superpowers plugin for brainstorming.
  harness-discovery/          Internal-audit pass — what already exists
                              in the codebase that the plan can reuse?
  harness-research/           Bounded external research (max 1+3 search
                              budget). Use only when the plan can't be
                              scoped without external context. Default
                              is no research.

  ── Phases 1+: planning + execution + review ──
  harness-planner/            Plans the harness from the prepared doc
  harness-coordinator/        Coordinator workflow rules
  harness-discipline/         TDD discipline for specialist agents
  gauntlet-review/            Multi-specialist antagonistic spec review
  feature-enhancements/       Pending-feature triage workflow

frameworks/                   → ~/pm33-frameworks/
  HARNESS_PROJECT_TEMPLATE.md           Template + section-by-section guide
  LONG_RUNNING_AGENT_FRAMEWORK.md       The full framework spec (long read)
  HARNESS_TASK_DEFINITION_TEMPLATE.json Per-task JSON skeleton

CLAUDE.md.starter             Starter PM33-aware CLAUDE.md fragment.
                              Copy into your project root (or merge with
                              an existing CLAUDE.md) to get PM33 conventions.
```

## After install — 5 steps

1. **Restart Claude Code** — newly installed agents and skills load on the
   next session.

2. **Install the `superpowers` plugin** so `harness-prep` can invoke
   `superpowers:brainstorming` for the always-on brainstorming pass:
   ```
   /plugin marketplace add anthropics/claude-plugins-official
   /plugin install superpowers
   ```
   Without `superpowers`, `harness-prep` skips brainstorming and warns —
   usable but degraded.

3. **Connect the PM33 MCP server** so the agents can query/edit PM33
   tracking:
   - In Claude Code: type `/mcp`
   - Pick **"claude.ai PM33"** → complete the OAuth flow
   - You'll see `mcp__pm33-staging__pm33_*` tools appear
   - Before your first `pm33_*` call each session, invoke
     `Skill({ skill: "pm33-mcp" })` to load the conventions

4. **Copy the starter CLAUDE.md** into your project root:
   ```bash
   cp ~/pm33-frameworks/CLAUDE.md.starter ./CLAUDE.md
   ```
   Then edit. Adds PM33 conventions (harness discipline, gauntlet review,
   MCP tracking) to every session in that project.

5. **Try the harness flow** on a non-trivial project:
   - Say *"prep a harness for this"* — `harness-prep` sequences
     discovery + brainstorming + (conditional) research into one
     enriched doc
   - Then *"plan the harness"* — `harness-planner` consumes the prep doc
   - For multi-session execution, say *"resume as coordinator"* — the
     `harness-coordinator` agent takes the role

## What requires a PM33 subscription

The harness is free. **Closed-loop capabilities** require a paid PM33
subscription:

- Outcome attribution (AR(1) recalibration tied back to strategic objectives)
- Strategic-objective scoring + Brief alignment
- Capacity-aware sprint scheduler
- Audit log + tenant-isolated multi-workspace tracking
- The full Pam orchestrator + MCP tool surface
  (`mcp__pm33-staging__pm33_*`)

See [pm-33.com/pricing](https://pm-33.com/pricing).

## Attribution

The 10 specialist agents — `ai-engineer`, `api-documenter`,
`backend-architect`, `code-reviewer`, `database-admin`,
`frontend-developer`, `performance-engineer`, `security-auditor`,
`test-automator`, `ui-ux-designer` — are from
[**wshobson/agents**](https://github.com/wshobson/agents) (MIT). PM33
redistributes this curated subset because every PM33 project leans on
them per `CLAUDE.md` agent-selection guidance. License terms:
`agents/LICENSE.wshobson`.

For the full 75+ agent collection or the latest upstream versions, add
the marketplace inside Claude Code:

```
/plugin marketplace add wshobson/agents
```

The PM33 harness roles (`harness-coordinator`, `harness-planner-agent`)
and all skills are PM33-authored.

## License

PM33-authored content (the 2 harness agents, 9 skills, frameworks, this
README) is **MIT-licensed** — see `LICENSE`.

The 10 wshobson specialist agents retain their original MIT license
(`agents/LICENSE.wshobson`).

## Updating

The bundle on pm-33.com auto-rebuilds every Monday morning from PM33's
internal sources. To pull the latest in your local install:

```bash
curl -fsSL https://pm-33.com/install/pm33-init.sh | bash
```

…or `git pull` this repo + re-run `./install.sh`.

## Contributing

If you found a bug or want to suggest an improvement to one of the
PM33-authored skills/agents, open an issue or PR here. Changes to the
upstream PM33 source can be PR'd at this repo and we'll mirror them
into pm-33-core on accept.

For the wshobson specialist agents, contribute upstream at
[wshobson/agents](https://github.com/wshobson/agents).
