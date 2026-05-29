# PM33 Bundle for Claude Code

You're looking at the PM33 starter bundle — installed by
`curl -fsSL https://pm-33.com/install | bash`.

## What's here

```
agents/                       → ~/.claude/agents/
  ── PM33 harness roles ──
  harness-coordinator.md      Orchestrates multi-session harness work
  harness-planner-agent.md    Breaks complex projects into phased tasks

  ── Essential specialists (from wshobson/agents, MIT) ──
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
  harness-coordinator/        Coordinator workflow rules
  harness-discipline/         TDD discipline for specialist agents
  harness-discovery/          Initial discovery for new harness projects
  harness-planner/            Planning skill for harness projects
  gauntlet-review/            Multi-specialist antagonistic spec review
  feature-enhancements/       Pending-feature triage workflow

frameworks/                   → ~/pm33-frameworks/
  HARNESS_PROJECT_TEMPLATE.md           Template + section-by-section guide
  LONG_RUNNING_AGENT_FRAMEWORK.md       The full framework spec (long read)
  HARNESS_TASK_DEFINITION_TEMPLATE.json Per-task JSON skeleton

CLAUDE.md                     Starter PM33-aware CLAUDE.md fragment.
                              Copy into your project root (or merge with
                              an existing CLAUDE.md) to get PM33 conventions.
```

## First steps after install

1. **Restart Claude Code** if it's running. Newly-installed agents and skills
   get picked up on the next session.

2. **Connect the PM33 MCP server** so the agents can query/edit PM33 tracking:
   - In Claude Code: type `/mcp`
   - Pick **"claude.ai PM33"**
   - Complete the OAuth flow in your browser
   - You'll see `mcp__pm33-staging__pm33_*` tools appear in your registry
   - Before your first `pm33_*` call in any session, invoke
     `Skill({ skill: "pm33-mcp" })` to load the conventions skill

3. **Try the harness flow** on a small project:
   - Open a Claude Code session in any project
   - Say *"plan this as a harness project"* — the `harness-planner` skill
     activates
   - For multi-session work, say *"resume as coordinator"* — the
     `harness-coordinator` agent takes the role

4. **Try the gauntlet review** before implementing a non-trivial spec:
   - Have a spec markdown ready
   - Say *"gauntlet review this spec"* — multiple specialist agents run
     adversarial review in parallel

## Updating later

Re-run the same install command:
```
curl -fsSL https://pm-33.com/install | bash
```

The script overwrites the bundled files with the latest versions. Anything
you've edited locally is overwritten — use git to track changes if you
fork an agent or skill.

## Uninstalling

```
rm -rf ~/.claude/agents/harness-coordinator.md \
       ~/.claude/agents/harness-planner-agent.md \
       ~/.claude/skills/pm33-mcp \
       ~/.claude/skills/harness-coordinator \
       ~/.claude/skills/harness-discipline \
       ~/.claude/skills/harness-discovery \
       ~/.claude/skills/harness-planner \
       ~/.claude/skills/gauntlet-review \
       ~/.claude/skills/feature-enhancements \
       ~/pm33-frameworks
```

## Attribution

The 10 specialist agents (ai-engineer, api-documenter, backend-architect,
code-reviewer, database-admin, frontend-developer, performance-engineer,
security-auditor, test-automator, ui-ux-designer) are from
**`wshobson/agents`** — a curated marketplace of Claude Code subagents
distributed under the MIT License. See `agents/LICENSE.wshobson`.

PM33 redistributes this curated subset because every PM33 project leans
on them per `CLAUDE.md` agent-selection guidance. If you want the full
75+ agent collection or the latest upstream version, add the marketplace
directly inside Claude Code:

```
/plugin marketplace add wshobson/agents
```

The PM33 harness roles (`harness-coordinator`, `harness-planner-agent`)
and all skills are PM33-authored.

## Reporting issues

Open an issue at https://github.com/b33-steve/pm33-website-v3-1 (the
install bundle is built from this repo).
