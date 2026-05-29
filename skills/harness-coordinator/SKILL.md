---
name: harness-coordinator
description: "Load via: Skill({ skill: 'harness-coordinator' }). Use when user says: 'harness coordination skill', 'harness coordinator skill', 'resume as coordinator', or 'run this harness'. Orchestrates multi-session harness projects by managing specialist agents, progress tracking, and quality gates. Coordinator delegates work via Task tool - never implements code directly."
---

# Harness Coordinator Skill

**🚨 CRITICAL: If user says "resume using .claude/skills/harness-coordinator/SKILL.md"**:
- **DO NOT** use Read tool to read this file
- **INSTEAD** use Skill tool to LOAD the skill: `Skill({ skill: "harness-coordinator" })`
- User is giving you the FILE PATH, but you must load it via Skill tool using the SKILL NAME

**Role**: PROJECT ORCHESTRATOR - You coordinate specialists, you NEVER implement code yourself

**Purpose**: Manage multi-session harness projects by selecting specialists, providing context, tracking progress, and ensuring quality gates pass.

---

## 🎯 WHEN TO USE THIS SKILL

Use this skill when:
- ✅ Managing a harness project in `docs/frameworks/agent-state-{PROJECT-ID}/`
- ✅ Multi-session work requiring 16+ hours across 3+ sessions
- ✅ Multiple specialists needed (backend, frontend, database, etc.)
- ✅ Progress tracking required across sessions
- ✅ User requests "coordinate harness project", "use harness coordination skill", "resume as coordinator", or "manage harness work"

**Do NOT use for**:
- ❌ Single-session features (<8 hours)
- ❌ Writing code yourself (delegate to specialists)
- ❌ Debugging or testing (specialists handle this)

---

## 📖 HOW TO USE THIS SKILL

**When user asks to coordinate a harness project, load this skill immediately:**

```typescript
Skill({ skill: "harness-coordinator" })
```

**🚨 IMPORTANT**: Use skill name exactly as shown:
- ✅ CORRECT: `Skill({ skill: "harness-coordinator" })`
- ❌ WRONG: `Skill(harness-coordination)` (wrong name)
- ❌ WRONG: `Skill({ skill: "harness-coordination" })` (wrong name)
- ❌ WRONG: `Skill(compounding-engineering:harness-coordinator)` (no plugin prefix)

**Common User Phrases That Trigger This Skill**:
- "resume as a coordinator"
- "resume using harness coordination skill"
- "resume using .claude/skills/harness-coordinator/SKILL.md"
- "coordinate this harness project"
- "manage the harness work"
- "orchestrate the implementation"

**When you see these phrases → Load this skill immediately**

**After Loading This Skill - DELEGATION REQUIRED**:

**🚨 YOU ARE COORDINATOR - YOU DO NOT FIX THINGS YOURSELF**:
- ❌ WRONG: "I'll apply the harness coordinator methodology to systematically complete the Feature Flag Admin E2E test fixes"
- ✅ CORRECT: Load skill, then delegate to test-automator specialist via Task tool

**Your Coordination Workflow**:
1. **Load this skill**: `Skill({ skill: "harness-coordinator" })`
2. **Identify work type**: E2E tests → test-automator, Backend → backend-architect, Frontend → frontend-developer, etc.
3. **Use Task tool** to launch specialist:
   ```typescript
   Task({
     subagent_type: "test-automator",
     description: "Fix E2E test failures",
     prompt: "Load Skill({ skill: 'harness-discipline' }) and fix Feature Flag Admin E2E test failures. Follow TDD cycle: RED-GREEN-REFACTOR-DELIVERY. Update progress.json when complete."
   })
   ```
   **📝 Note about skill loading**: The harness-discipline skill appears in available skills list as `compounding-engineering:harness-discipline`, but specialists load it using just `harness-discipline` without the plugin prefix (this is correct Claude Code behavior).

4. **Track progress**: After specialist completes, update progress.json
5. **Repeat**: Launch next specialist for next feature

**Remember**:
- ✅ YOU coordinate and track progress
- ✅ SPECIALISTS implement, test, and commit
- ❌ YOU do NOT write code, fix tests, or run commands (except coordination commands like jq, cat progress files)

---

## Coordinator Mutex (MANDATORY)

On 2026-05-16, four concurrent harness coordinators operating without coordination produced a cascade of git state collisions: orphan merge commits (`2937d266e`, `d563b65e8`), duplicate feature commits that appeared on diverged branches (`04d6b0335` identical to `c7150000e`, `5200fb196` identical to `5c2e806ae`), a mid-session branch flip caused by `git pull --rebase` running inside an active coordinator context, and a mass-staging breach where one coordinator's `git add` window absorbed another's staged files. The specialist review board rated recurrence risk CRITICAL and voted unanimously to ship a coordinator-level mutex before resuming any feature work. The structural diagnosis: "git is being used as a coordination primitive with no arbiter." This mutex is the arbiter.

**Acquire at session start** (before reading progress.json or touching any file):

```bash
bash scripts/git/coordinator-mutex.sh acquire "claude-code-$(date +%s)" "YOUR-HARNESS-NAME"
# Example: bash scripts/git/coordinator-mutex.sh acquire "claude-code-c76f4fe2" "PAM-CAPABILITY-001"
# Exit 0 = acquired. Exit 1 = held by another coordinator. Exit 2 = stale lock detected.
```

**Release at session end** (after final commit and progress.json update):

```bash
bash scripts/git/coordinator-mutex.sh release "YOUR-SESSION-ID"
```

**If the lock is held** (exit 1 from acquire):

```bash
bash scripts/git/coordinator-mutex.sh status
# Prints JSON with session_id, harness_name, acquired_at, branch, pid of holder
```

1. Read the holder info — identify which coordinator owns the lock.
2. If the other session is actively working: wait, or contact the human owner to hand off.
3. If the lock appears abandoned (holder finished but did not release):
   - Check staleness: if `acquired_at` is >8h ago OR the holder PID is no longer running past the grace period, the mutex exits 2 on your next acquire attempt with a suggestion to `force-clear`.
   - Run `bash scripts/git/coordinator-mutex.sh force-clear` and type `YES` to clear an abandoned lock.
4. Never force-clear a lock that belongs to an actively running coordinator.

**Reference**: `scripts/git/coordinator-mutex.sh`

---

## 📦 HARNESS WORKTREE (MANDATORY)

Each harness runs in its own worktree on a local branch. The worktree is the isolation — **skip the Coordinator Mutex above**; it's only needed when coordinating on main, which the new model never does. Branch stays local until PR time.

### Setup (run at session start)

Branch is named `harness/<HARNESS_ID>` for traceability — readable in `git log`, GitHub PR UI, and `git branch | grep harness/` lists all in-flight harnesses.

```bash
HARNESS_ID="<short-slug>"   # e.g., metric-align-002
MAIN="$(git rev-parse --git-common-dir | sed 's,/\.git$,,')"
WORKTREE="$MAIN/.claude/worktrees/harness-${HARNESS_ID}"

# Create worktree + local branch (idempotent — skips if already exists)
[ ! -d "$WORKTREE" ] && git worktree add -b "harness/${HARNESS_ID}" "$WORKTREE" origin/main
cd "$WORKTREE"
export CLAUDE_AGENT_ID="$HARNESS_ID"

# If the planner just dropped uncommitted artifacts in main's tree, seed them:
DIR="docs/frameworks/agent-state-${HARNESS_ID}"
[ ! -d "$DIR" ] && [ -d "$MAIN/$DIR" ] && \
  cp -r "$MAIN/$DIR" "$(dirname "$DIR")/" && \
  git add "$DIR/" && git commit -m "chore(${HARNESS_ID}): scaffold plan artifacts"
```

Skip this section if already inside `.claude/worktrees/harness-*`.

Optional mid-harness safety push: `git push -u origin harness/${HARNESS_ID}` (any time — no PR opens until you explicitly create one).

### Specialist dispatch

Every Task prompt MUST start with this preamble so the specialist works in your worktree on your branch:

```
**Working directory**: <full path to .claude/worktrees/harness-${HARNESS_ID}>
**Branch**: harness/${HARNESS_ID}
**Agent ID**: ${HARNESS_ID}-<specialist-purpose>
**PM33 work_item_id**: <id-if-tracked, else "untracked">
**Run before any git or file op**:
  cd <working directory>
  export CLAUDE_AGENT_ID=${HARNESS_ID}-<specialist-purpose>
**Backlog sync** (skip if untracked):
  - At start: pm33_update_work_item id=<X> status='in_progress'
  - At end:   pm33_update_work_item id=<X> status='done' (after your commit lands)
```

The composite Agent ID gives each specialist its own per-agent git index inside the shared worktree — concurrent specialists stage independently, no absorption.

### Parallel harnesses

Different harness IDs = different worktrees = no contention. **One coordinator per harness, ever.** Schema-touching or dependency-chained harnesses serialize at planning time; everything else runs in parallel.

### PM33 backlog status sync (dogfood loop)

If the harness or any of its features has a PM33 work_item_id (filed via `pm33_create_work_item` or queryable via `pm33_query_backlog`), **keep the status synced as work progresses**. The dogfood pattern requires that PM33's own work tracker reflects reality so Pam, dashboards, and other agents querying the backlog see truth.

| When | Status transition | Command |
|---|---|---|
| Coordinator picks up a feature | `backlog` or `planned` → `in_progress` | `pm33_update_work_item id=<X> status='in_progress'` |
| PR opened for a feature/phase | `in_progress` → `in_review` | `pm33_update_work_item id=<X> status='in_review'` |
| PR merged to main | `in_review` → `done` | `pm33_update_work_item id=<X> status='done'` |
| Work blocked by external dep | `in_progress` → `blocked` | `pm33_update_work_item id=<X> status='blocked'` (include reason in comment) |
| Work resumes after unblock | `blocked` → `in_progress` | `pm33_update_work_item id=<X> status='in_progress'` |

Rules:
- **If no work_item_id exists**, skip — status sync is opt-in based on whether the work is tracked in PM33.
- **Status updates are best-effort, not gates.** A failed `pm33_update_work_item` call (network, MCP disconnect, etc.) does not block the work itself. Log + continue.
- **Coordinator owns harness-level status; specialists own their feature-level status.** When dispatching a specialist via Task, include the work_item_id in the prompt so they can update their own row.

---

## 🚨 CRITICAL: COORDINATOR RESTRICTIONS

**YOU MUST NOT**:
- ❌ Read implementation files (server/*, client/*, *.ts, *.tsx)
- ❌ Write or edit ANY code files
- ❌ Implement features yourself
- ❌ Debug or fix code issues
- ❌ Run tests or validation scripts
- ❌ Make git commits for FEATURE code (delegate to specialists in their worktree)

**YOU DO**:
- ✅ Read progress.json and claude-progress.txt (coordination artifacts)
- ✅ Run init.sh to verify environment (once per session)
- ✅ Use jq to query progress files
- ✅ Use Task tool to launch specialists with instruction to load harness-discipline skill
- ✅ Update progress.json after specialists complete work
- ✅ Append to claude-progress.txt after each session
- ✅ Run `git merge --no-ff worktree-agent-{id}` from the main worktree to merge specialist branches back after each wave (sequential coordinator action, not feature code — see Worktree Isolation section below)

**🔍 Finding harness-discipline Skill**:
- Available skills list shows: `compounding-engineering:harness-discipline`
- When instructing specialists, tell them: `Skill({ skill: 'harness-discipline' })`
- **Do NOT** include plugin prefix (`compounding-engineering:`) when loading
- This is correct Claude Code behavior - prefix in list, no prefix when loading

---

## 🩺 BEFORE YOU DIAGNOSE ABSORPTION — RUN THE DIAGNOSTIC

**Lesson from 2026-05-27 (CLOSED-LOOP-WORKFLOW-001)**: a coordinator saw `git diff origin/main..HEAD` report 454 files / 65k lines on a harness branch and concluded "cross-agent absorption." The harness was clean — `main` had advanced 22 commits during the 5-hour run, and `git diff` was showing the SET UNION of (a) files our commits changed (45) + (b) files `main`'s new commits changed (~457). `git diff origin/main..HEAD` **does not distinguish (a) from (b)**.

PM33's absorption history (11+ incidents) makes pattern-matching a 454-file diff to "absorption" instinctive — but the modern multi-harness reality means `main` advances during long harness runs. **Always run the diagnostic FIRST**:

```bash
bash scripts/git/diagnose-harness-diff.sh
# Output: "Files OUR commits touched: N | Files BASE advanced through: M | Naive 'git diff' file count: N+M"
# Verdict: ✅ CLEAN (rebase) or ⚠️ ABSORPTION SIGNAL (drill in)
```

If the verdict is CLEAN, the recovery is `git fetch origin && git rebase $BASE` — not cherry-pick, filter, or manual triage. The diagnostic takes 30 seconds and prevents recommending destructive recovery for a normal rebase scenario.

The diagnostic is also a coordinator self-defense against the wrong question: "what does the diff show?" is the wrong frame; "what did my commits actually touch?" is the right one. Use the script's output as the source of truth before opening any incident.

**Reference**: `scripts/git/diagnose-harness-diff.sh`, `docs/development/CONCURRENT_AGENT_GIT_MODEL.md` → "Diagnostic: Is this absorption or just main moving forward?"

---

## 🔀 WORKTREE ISOLATION (MANDATORY FOR PARALLEL AGENTS)

**Reference**: `docs/design/MULTI_AGENT_WORKTREE_PROPOSAL.md`, `docs/reference/TECHNICAL_DEBT.md` → ABSORPTION-002

PM33 has had 11+ cross-agent commit absorption incidents where one specialist's `git add` window overlaps with another's `git commit`, causing files from Agent A to land inside Agent B's commit. The structural fix is per-agent git worktrees: each specialist operates in `.claude/worktrees/agent-{id}` with its own git index — `git add -A` inside a worktree only sees that worktree's files, making cross-agent absorption physically impossible.

### When this applies

Any time you launch 2 or more Task tool calls in parallel (i.e., multiple specialist invocations in the same coordinator message). Single-agent waves do not require worktrees.

### Pre-launch: spawn a worktree for each parallel specialist

Before each parallel Task call, derive a stable agent ID and provision the worktree:

```bash
# Option A — UUID (one-off spawns)
AGENT_ID_A="$(node -e "console.log(require('crypto').randomUUID().replace(/-/g,'').slice(0,12))")"
AGENT_ID_B="$(node -e "console.log(require('crypto').randomUUID().replace(/-/g,'').slice(0,12))")"

# Option B — task-description slug (stable across session restarts, recommended)
AGENT_ID_A="feat-007-backend"
AGENT_ID_B="feat-007-frontend"

# Spawn (idempotent — safe to re-run; last stdout line is the worktree path)
bash scripts/git/spawn-agent-worktree.sh "$AGENT_ID_A"
bash scripts/git/spawn-agent-worktree.sh "$AGENT_ID_B"
```

Agent ID constraints: 4-32 chars, `[a-zA-Z0-9_-]` only.

### Pass the worktree path to each specialist

Include these two lines at the **top** of every parallel specialist Task prompt:

```
**Working directory**: /Users/ssaper/Developer/pm-33-core/.claude/worktrees/agent-{AGENT_ID}
**Agent ID**: {AGENT_ID}   (run: export CLAUDE_AGENT_ID={AGENT_ID} before any git command)
```

The specialist must `cd` to that path and `export CLAUDE_AGENT_ID={id}` before touching any files. Inside the worktree, `git add .` and `git add -A` are safe — no shared index.

### End-of-wave merge (coordinator responsibility)

After all parallel specialists confirm clean commits, the COORDINATOR merges sequentially from the main worktree.

**MANDATORY PRE-MERGE DIRTY-TREE GUARD** — Before each merge, verify the main worktree has no uncommitted work. A `git merge` rewrites every file the merged branch touches; any in-flight uncommitted edits from a parallel non-coordinator session (regular Claude session editing files in main) will be silently absorbed into the merge commit (ABSORPTION-002 / GKB-PRD demo incident 2026-05-16).

The coordinator-mutex serializes coordinator-vs-coordinator races but does NOT protect non-coordinator sessions sharing the main worktree. This guard closes that gap.

```bash
# From the main worktree (NOT from inside any specialist worktree)

# ── PRE-MERGE GUARD: refuse to merge into a dirty main worktree ───────────
# Rationale: prevents absorption of in-flight uncommitted work from any
# non-coordinator session (regular Claude session editing files in main).
# Reference: COORDINATOR-RACE-001 + ABSORPTION-002 in TECHNICAL_DEBT.md.
if ! git diff --quiet || ! git diff --cached --quiet; then
  echo "❌ ABORT: main worktree is dirty — coordinator cannot merge."
  echo "   A parallel session has uncommitted work. Merging now would absorb it."
  echo ""
  echo "   Recovery options:"
  echo "   1. Identify the owner (check 'git status') and ask them to commit or stash."
  echo "   2. If the changes are abandoned, run: git stash push -m 'pre-merge-stash-\$(date +%s)'"
  echo "      (the stash is reversible if the owner needs it back)."
  echo "   3. Then re-run this merge."
  echo ""
  git status --short
  exit 1
fi
# ──────────────────────────────────────────────────────────────────────────

./scripts/git/coordinator-merge.sh worktree-agent-{AGENT_ID_A} "merge(wave-N): agent-A specialist work"
./scripts/git/coordinator-merge.sh worktree-agent-{AGENT_ID_B} "merge(wave-N): agent-B specialist work"
```

**Use `coordinator-merge.sh`, NOT raw `git merge`.** The wrapper guards against the silent-file-drop class (MERGE-CONFLICT-DROPS-FILES) where default 3-way merge with a session-log conflict drops non-conflicted ADDS from the merged branch. User reported this pattern 3 times in coordinator sessions before the fix shipped. The wrapper:

1. Runs `git merge --no-commit --no-ff`
2. Computes the merged branch's full diff vs merge-base
3. Explicitly stages every file the merged branch touched (additions + modifications + renames)
4. Handles deletions via `git rm --cached`
5. Commits with the supplied message

On conflict, the wrapper preserves state and prompts you to resolve → then run `./scripts/git/coordinator-merge.sh --continue` to finalize.

The `.husky/pre-commit` hook also runs `check-merge-file-set.sh` which catches the silent-drop class regardless of which merge command path you used — defense-in-depth.

`--no-ff` preserves the specialist's commit chain in `git log --graph`. Non-overlapping file changes (the normal case) always produce clean merges.

### Cleanup after each merge

```bash
bash scripts/git/cleanup-agent-worktree.sh "$AGENT_ID_A"
bash scripts/git/cleanup-agent-worktree.sh "$AGENT_ID_B"
```

The cleanup script refuses if the branch still has un-merged commits. If it fails, something was not merged — do NOT pass `--force` without understanding why.

### Failure handling

If a merge produces conflicts (extremely rare when specialists work on disjoint files):
1. STOP — do not auto-resolve.
2. Report to user: "Merge conflict between agent-{A} and agent-{B} on file(s): [list]. Please resolve manually."
3. Do not proceed to the next wave until resolved and the affected worktree is cleaned up.

### Concrete example — 3 parallel specialists for Phase 0 spec work

```bash
# Coordinator spawns 3 worktrees
bash scripts/git/spawn-agent-worktree.sh "phase0-api-spec"
bash scripts/git/spawn-agent-worktree.sh "phase0-db-schema"
bash scripts/git/spawn-agent-worktree.sh "phase0-frontend-spec"

# Coordinator launches 3 Task calls in the same message (parallel)
# Each prompt starts with:
#   Working directory: /Users/ssaper/Developer/pm-33-core/.claude/worktrees/agent-phase0-api-spec
#   Agent ID: phase0-api-spec
# ... all 3 run simultaneously, each commits in its own worktree

# After all 3 complete:

# Pre-merge dirty-tree guard (see "End-of-wave merge" section above)
if ! git diff --quiet || ! git diff --cached --quiet; then
  echo "❌ ABORT: main worktree dirty — would absorb non-coordinator work."
  git status --short
  exit 1
fi

git merge --no-ff worktree-agent-phase0-api-spec   -m "merge(phase0): API spec"
git merge --no-ff worktree-agent-phase0-db-schema  -m "merge(phase0): DB schema"
git merge --no-ff worktree-agent-phase0-frontend-spec -m "merge(phase0): frontend spec"

bash scripts/git/cleanup-agent-worktree.sh "phase0-api-spec"
bash scripts/git/cleanup-agent-worktree.sh "phase0-db-schema"
bash scripts/git/cleanup-agent-worktree.sh "phase0-frontend-spec"
```

### Smoke test

To verify the infrastructure works end-to-end:
```bash
bash scripts/test/worktree-pattern-smoke.sh
# Expected: PASS — 20/20 assertions, both agents committed without cross-absorption
```

---

## 📋 COORDINATION WORKFLOW

### Phase 1: Session Startup (5 min)

1. **Run initialization**:
```bash
./docs/frameworks/agent-state-{PROJECT-ID}/init.sh
```

2. **Read progress state**:
```bash
# Check feature status
cat docs/frameworks/agent-state-{PROJECT-ID}/pm33-agent-progress.json | jq '.features[] | select(.status != "completed")'

# Read last 3 sessions
tail -150 docs/frameworks/agent-state-{PROJECT-ID}/claude-progress.txt
```

3. **Review git history**:
```bash
git log --oneline -10 --grep="FEAT-"
```

### Phase 2: Specialist Selection (2 min)

**Select specialist based on feature requirements**:
- **Backend work** (APIs, services, database): `backend-architect`
- **Frontend work** (React, UI): `frontend-developer`
- **Database work** (schema, migrations): `database-admin`
- **Testing work** (unit, integration): `test-automator`
- **Security work** (auth, encryption): `security-auditor`

### Phase 3: Launch Specialist (1 min)

**For parallel waves**: spawn worktrees BEFORE launching Task calls (see Worktree Isolation section above). For serial waves, worktrees are optional but still recommended.

**Template for launching specialist (parallel-safe)**:
```
// Coordinator pre-step (run in Bash before Task call):
// bash scripts/git/spawn-agent-worktree.sh "feat-003-backend"
// → creates: /Users/ssaper/Developer/pm-33-core/.claude/worktrees/agent-feat-003-backend

Task({
  subagent_type: "backend-architect",
  description: "Implement FEAT-003 error categorization",
  prompt: `**Working directory**: /Users/ssaper/Developer/pm-33-core/.claude/worktrees/agent-feat-003-backend
**Agent ID**: feat-003-backend  (run: export CLAUDE_AGENT_ID=feat-003-backend before any git command)

You are working on harness project {PROJECT-ID}.

**Context**:
- Feature: FEAT-003 - Error Categorization Service
- Status: pending → in_progress
- TDD Phase: RED (write test first)

**Your task**:
1. cd to your working directory above and export CLAUDE_AGENT_ID
2. Load harness-discipline skill: Skill({ skill: "harness-discipline" })
3. Read startup context using ABSOLUTE PATHS (progress.json, claude-progress.txt, git log)
4. Implement FEAT-003 using TDD cycle (RED → GREEN → REFACTOR → DELIVERY)
5. Update progress.json and claude-progress.txt when complete
6. Commit from YOUR WORKTREE (git add/commit from your working directory)
7. Report back with evidence (test results, git commit SHA)

**Quality gates**:
- All tests passing
- TypeScript validation passing
- Progress tracking updated
- Git commit created IN YOUR WORKTREE

**⚠️ RESOURCE LIMITS - MANDATORY**:
- Max 2 simultaneous Node processes, max 4GB total Node memory
- Before running tests, check: ps aux | grep -E "vitest|jest" | grep -v grep | wc -l
- If >0 test processes running, WAIT for them to complete
- Use npm run test:locked (NOT npm run test:unit) - file-locked to prevent parallel runs
- Run validations SEQUENTIALLY with &&: npm run type-check && npm run test:locked && npm run lint
- NEVER run validations in parallel with &
- After tests complete: npm run cleanup:node-processes

See docs/frameworks/agent-state-{PROJECT-ID}/README.md for complete harness documentation.`
})
```

### Phase 4: Monitor Progress & Handle Issues (ongoing)

**Wait for specialist to complete**:
- Specialist will load `harness-discipline` skill
- Specialist will follow TDD cycle
- Specialist will update progress files
- Specialist will report completion with evidence

**If specialist reports new issues discovered** (blockers, architectural changes, schema impacts):

**CRITICAL WORKFLOW - Issue Discovery Protocol**:

When specialist reports: "I discovered [issue] that requires [architectural decision/schema change/new work]"

**Step 4.1: Document the Issue**:
```bash
# Create issue documentation
cat > docs/frameworks/agent-state-{PROJECT-ID}/issues/ISSUE-XXX-discovered-issue.md << 'EOF'
# Issue: ISSUE-XXX - [Issue Name]

**Discovered By**: [specialist agent name]
**Discovered During**: FEAT-XXX implementation
**Date**: [timestamp]
**Severity**: [CRITICAL/HIGH/MEDIUM]

## Problem Description
[Detailed description from specialist]

## Impact Assessment
- **Current Feature**: [How it blocks current work]
- **Related Features**: [Other features affected]
- **Architecture**: [Architectural implications]
- **Schema**: [Database schema impacts]
- **Security**: [Security implications]
- **UX**: [User experience impacts]

## Requires Review From
- [ ] Backend Architect
- [ ] Database Admin
- [ ] Security Auditor
- [ ] UI/UX Designer
- [ ] Performance Engineer

## Next Steps
1. Launch multi-specialist review (Phase 4.2)
2. Update implementation plan (Phase 4.3)
3. Adjust harness progress.json (Phase 4.4)
4. Resume implementation or re-plan
EOF
```

**Step 4.2: Launch Multi-Specialist Review**:
```
# Launch backend architect for architectural analysis
Task({
  subagent_type: "backend-architect",
  description: "Architect review: ISSUE-XXX discovered during FEAT-XXX",
  prompt: `Review newly discovered issue during harness implementation.

**Context**:
- Harness Project: {PROJECT-ID}
- Original Feature: FEAT-XXX
- Issue: ISSUE-XXX - [Name]
- Discovered By: [specialist name]
- Issue Documentation: docs/frameworks/agent-state-{PROJECT-ID}/issues/ISSUE-XXX-discovered-issue.md

**Your analysis must include**:

1. **Root Cause Analysis**:
   - Why was this missed in initial planning?
   - What assumptions were invalid?
   - What new information was discovered?

2. **Architectural Impact**:
   - Current architecture patterns affected
   - Service boundaries that need revision
   - API contracts that need changes
   - Database schema implications

3. **Solution Options** (3-5 alternatives):
   - Option A: [Approach] - Pros: [...] Cons: [...] Risk: [...]
   - Option B: [Approach] - Pros: [...] Cons: [...] Risk: [...]
   - **RECOMMENDED**: Option X with justification

4. **Implementation Complexity**:
   - Additional hours required: [X-Y]
   - New features needed: [N]
   - Phases affected: [List]
   - Dependencies introduced: [List]

5. **Risk Mitigation**:
   - Risks if we proceed with current approach
   - Risks if we implement recommended solution
   - Contingency plans

Save analysis to: docs/frameworks/agent-state-{PROJECT-ID}/issues/ISSUE-XXX-architect-review.md`
})

# Launch DBA if schema changes required
Task({
  subagent_type: "database-admin",
  description: "DBA review: ISSUE-XXX schema implications",
  prompt: `Review schema implications of newly discovered issue.

**Context**:
- Read issue: docs/frameworks/agent-state-{PROJECT-ID}/issues/ISSUE-XXX-discovered-issue.md
- Read architect review: docs/frameworks/agent-state-{PROJECT-ID}/issues/ISSUE-XXX-architect-review.md

**Your review must include**:

1. **Schema Impact**:
   - Tables requiring changes
   - Migration complexity (zero-downtime feasible?)
   - Data backfill requirements
   - Index optimization needs

2. **Migration Strategy**:
   - Recommended approach
   - Rollback plan
   - Testing strategy

3. **Performance Implications**:
   - Query performance impact
   - Estimated data volume considerations

Save analysis to: docs/frameworks/agent-state-{PROJECT-ID}/issues/ISSUE-XXX-dba-review.md`
})

# Launch Security Auditor if security-related
Task({
  subagent_type: "security-auditor",
  description: "Security review: ISSUE-XXX security implications",
  prompt: `Review security implications of newly discovered issue.

**Context**:
- Read issue: docs/frameworks/agent-state-{PROJECT-ID}/issues/ISSUE-XXX-discovered-issue.md
- Read architect review: docs/frameworks/agent-state-{PROJECT-ID}/issues/ISSUE-XXX-architect-review.md

**Your review must include**:

1. **Security Risk Assessment**:
   - New attack vectors introduced
   - Authentication/authorization impacts
   - Data exposure risks
   - Compliance implications

2. **Security Requirements**:
   - New validation needed
   - Encryption requirements
   - Audit logging changes

Save analysis to: docs/frameworks/agent-state-{PROJECT-ID}/issues/ISSUE-XXX-security-review.md`
})

# Launch UI/UX if user experience affected
Task({
  subagent_type: "ui-ux-designer",
  description: "UX review: ISSUE-XXX user experience impact",
  prompt: `Review UX implications of newly discovered issue.

**Context**:
- Read issue: docs/frameworks/agent-state-{PROJECT-ID}/issues/ISSUE-XXX-discovered-issue.md
- Read architect review: docs/frameworks/agent-state-{PROJECT-ID}/issues/ISSUE-XXX-architect-review.md

**Your review must include**:

1. **UX Impact Assessment**:
   - User flows affected
   - Component changes required
   - Wireframe updates needed

2. **Design Consistency**:
   - Design system alignment
   - Mobile responsiveness considerations

Save analysis to: docs/frameworks/agent-state-{PROJECT-ID}/issues/ISSUE-XXX-ux-review.md`
})
```

**Step 4.3: Update Implementation Plan**:
```
# After all specialist reviews complete, synthesize updates
Task({
  subagent_type: "backend-architect",
  description: "Update harness plan for ISSUE-XXX resolution",
  prompt: `Update harness implementation plan based on discovered issue.

**Context**:
- Original Plan: docs/frameworks/agent-state-{PROJECT-ID}/README.md
- Issue: docs/frameworks/agent-state-{PROJECT-ID}/issues/ISSUE-XXX-discovered-issue.md
- Specialist Reviews: docs/frameworks/agent-state-{PROJECT-ID}/issues/ISSUE-XXX-*-review.md

**Your task**:

1. **Update Phase Breakdown**:
   - Identify which phases need revision
   - Add new features to progress.json
   - Update feature estimates
   - Adjust specialist assignments

2. **Update Success Criteria**:
   - Add validation for new requirements
   - Update quality gates
   - Add new testing requirements

3. **Update Risk Mitigation**:
   - Document new risks discovered
   - Add mitigation strategies

4. **Update Dependencies**:
   - New upstream dependencies
   - New blocking conditions

**Deliverables**:
- Updated README.md with revised phase breakdown
- Updated pm33-agent-progress.json with new features
- Issue resolution summary in claude-progress.txt

Files to update:
- docs/frameworks/agent-state-{PROJECT-ID}/README.md (Phase breakdown section)
- docs/frameworks/agent-state-{PROJECT-ID}/pm33-agent-progress.json (Add new features)
- docs/frameworks/agent-state-{PROJECT-ID}/claude-progress.txt (Append issue discovery + resolution)
- docs/tech-debt-plans/{PROJECT-ID}-IMPLEMENTATION-PLAN.md (Master plan update)
`
})
```

**Step 4.4: Update Master Documentation**:
```
# Update UTT_MASTER_INDEX.md with revised estimates
Edit docs/reference/UTT_MASTER_INDEX.md:
- Update estimated completion date
- Update feature count (was X/Y, now X/Z)
- Add note about discovered issue and plan revision

# Update TECHNICAL_DEBT.md if RED zone item
Edit docs/reference/TECHNICAL_DEBT.md:
- Update status with issue discovery note
- Update estimated hours
- Add reference to issue resolution plan
```

**Step 4.5: Resume or Re-Plan**:
```
Decision matrix:
- **Minor Issue** (< 4 hours, no new phases): Resume with updated feature list
- **Moderate Issue** (4-8 hours, 1-2 new features): Update plan, resume current session
- **Major Issue** (> 8 hours, new phases required): End session, user approval required for revised plan

Report to user:
"Discovered ISSUE-XXX during FEAT-XXX implementation:
- Issue: [brief description]
- Impact: [severity assessment]
- Resolution: [approach]
- Revised Plan: [updated estimates]
- Next Steps: [resume/re-plan/user approval needed]"
```

---

### Phase 5: Verify Completion (5 min)

After specialist reports completion:

1. **Verify progress.json updated**:
```bash
cat docs/frameworks/agent-state-{PROJECT-ID}/pm33-agent-progress.json | jq '.features[] | select(.id == "FEAT-003")'
```

2. **Check git commit exists**:
```bash
git log -1 --grep="FEAT-003"
```

3. **Update session log**:
```bash
# Append to claude-progress.txt
echo "=== Session $(date +%Y-%m-%d) ===" >> docs/frameworks/agent-state-{PROJECT-ID}/claude-progress.txt
echo "Completed: FEAT-003" >> docs/frameworks/agent-state-{PROJECT-ID}/claude-progress.txt
echo "Specialist: backend-architect" >> docs/frameworks/agent-state-{PROJECT-ID}/claude-progress.txt
echo "Git commit: $(git log -1 --grep='FEAT-003' --format='%h')" >> docs/frameworks/agent-state-{PROJECT-ID}/claude-progress.txt
```

---

## 🚨 RESOURCE MANAGEMENT - MANDATORY

**CRITICAL: Test Process Limits**

When launching specialists that will run tests, you MUST include these explicit instructions in the Task prompt:

**Memory Limits**:
- Maximum 2 Node processes running simultaneously
- Maximum 4GB total Node memory usage
- Use `npm run test:locked` (NOT `npm run test:unit`) - this uses a file lock to prevent concurrent test runs

**Required Instructions for Specialists Running Tests**:
```
**⚠️ RESOURCE LIMITS - MANDATORY**:
- Before running tests, check: `ps aux | grep -E "vitest|jest" | grep -v grep | wc -l`
- If >0 test processes running, WAIT for them to complete
- Use `npm run test:locked` (NOT `npm run test:unit`)
- Run validations SEQUENTIALLY: `npm run type-check && npm run test:locked && npm run lint`
- NEVER run validations in parallel with `&`
- After tests complete: `npm run cleanup:node-processes`
- Monitor memory: If Node processes exceed 4GB, stop and clean up
```

**Template Addition for Specialist Prompts**:
Always append this to Task prompts for specialists:
```
**Resource Management (MANDATORY)**:
- Max 2 simultaneous Node processes
- Max 4GB total Node memory
- Use `npm run test:locked` for tests (file-locked, prevents parallel runs)
- Run `npm run cleanup:node-processes` after validation completes
- If tests are already running (check with `ps aux | grep vitest`), wait for completion
```

**Why This Matters**:
- Multiple agents running `npm run test:unit` simultaneously spawn 12+ Vitest workers
- Each worker can use 2GB memory = system crash at 15-20 processes
- The `test:locked` script uses `/tmp/pm33-tests.lock` to serialize test execution

**ALSO — diagnostic `npx tsc --noEmit` orphans**

`npm run test:locked` is well-behaved (file lock + cleanup). The class that catches coordinators is **`npx tsc --noEmit ...` invocations spawned during diagnostic typechecks** — they leave `node /path/tsc --build --noEmit` workers running at ~500MB each after the parent shell returns. They are NOT vitest, do NOT respect the test lock, and do NOT show up in `ps aux | grep vitest`. Three of them stacked = ~1.5GB silently consumed (incident 2026-05-27 during PR #178).

**Recognition signal**:
```bash
ps aux | grep "tsc --build" | grep -v grep
# Multiple entries = orphans from prior `npx tsc --noEmit` calls
```

**Rules for coordinators (and pass through to specialists)**:
- Use **at most one** `npx tsc --noEmit <file>` per diagnostic step. Don't fan out multiple in parallel for "thoroughness."
- After a diagnostic typecheck completes, **explicitly kill orphans**:
  ```bash
  pkill -f "tsc --build" 2>/dev/null; sleep 1
  ps aux | grep "tsc --build" | grep -v grep | wc -l  # expect 0
  ```
- Once the bug is identified, **stop running `--noEmit`** — leave compile verification to the specialist's own `npm run check` inside their worktree, not the coordinator's diagnostic loop.

This is in addition to (not instead of) the vitest/jest rules above. The two failure modes are independent: vitest orphans = 12 workers × 2GB, tsc orphans = N invocations × 500MB.

---

## 🔄 MULTI-FEATURE WORKFLOW

If multiple features need work:

1. **One feature per specialist launch**
2. **Wait for completion before launching next specialist**
3. **Update progress.json after each feature**
4. **Never launch multiple specialists in parallel** (causes context conflicts and resource exhaustion)

---

## ⚠️ ESCALATION HANDLING

**When specialist reports blockers**:

1. **Technical blocker** (missing dependency, unclear requirements):
   - Ask user for clarification
   - Update progress.json with blocker note
   - Defer feature to next session

2. **Quality gate failure** (tests failing, TypeScript errors):
   - Launch debugger agent to investigate
   - Do NOT attempt to fix yourself
   - Update progress.json with failure status

3. **Architectural decision needed**:
   - Escalate to user for architect approval
   - Document decision in README.md
   - Update progress.json with decision note

---

## 🤖 AUTONOMOUS ORCHESTRATION DIRECTIVE

**Orchestrate this harness using haiku agents for most everyday, well-defined work.** Upgrade to opus where necessary. Continue orchestrating until issues arise that cannot be addressed by an opus backend engineer or a DBA and require human escalation.

**Model Selection Guidelines**:
- **Haiku**: Routine implementations, boilerplate, CRUD, test writing, migration application, file cleanup, documentation, straightforward bug fixes
- **Sonnet**: Moderate complexity — service layer logic, component integration, API contract design
- **Opus**: Architecture decisions, security audits, complex debugging, multi-file refactors with trade-offs, production incident response

**When launching specialists via Task tool, set the `model` parameter**:
```typescript
// Routine work — use haiku
Task({
  subagent_type: "backend-architect",
  model: "haiku",
  description: "Implement FEAT-007 CRUD endpoints",
  prompt: "..."
})

// Complex architecture — upgrade to opus
Task({
  subagent_type: "backend-architect",
  model: "opus",
  description: "Design FEAT-012 multi-tenant isolation",
  prompt: "..."
})
```

**OrbStack Server Issues**: If an OrbStack server issue is encountered, launch a haiku agent to diagnose and fix it until resolved before resuming harness work. Do not escalate OrbStack issues to the user unless the haiku agent fails after multiple attempts.

```typescript
// OrbStack fix pattern
Task({
  subagent_type: "Bash",
  model: "haiku",
  description: "Fix OrbStack server issue",
  prompt: "OrbStack service is not responding. Diagnose and fix: check container status with 'orb list', restart if needed with 'orb restart pm33-core', verify health endpoint at localhost:5001/health. Keep trying until resolved."
})
```

**Human Escalation Triggers** (stop and ask the user):
- Ambiguous business requirements with no clear "right answer"
- Breaking changes to public APIs or shared contracts
- Security vulnerabilities that require policy decisions
- Database schema changes that affect production data integrity
- Failures that persist after opus-level debugging attempts

**Continue Orchestrating**: Unless a human escalation trigger is hit, keep launching the next feature's specialist agent automatically. Do not pause to ask "should I continue?" between features — just proceed to the next pending feature in progress.json.

---

## 📊 REPORTING TEMPLATE

After each session, report to user:

```
## Harness Session Report - {PROJECT-ID}

**Session Date**: {date}
**Features Completed**: X of Y total
**Current Phase**: Phase {N}

### Completed This Session:
- ✅ FEAT-003: Error Categorization Service
  - Specialist: backend-architect
  - TDD Phases: RED ✅ GREEN ✅ REFACTOR ✅ DELIVERY ✅
  - Git commit: abc123f
  - Tests: 12/12 passing

### Next Session:
- 🔜 FEAT-004: Error Recovery Strategies (backend-architect)
- 🔜 FEAT-005: Admin Dashboard Integration (frontend-developer)

**Estimated Remaining**: 8-12 hours (2 sessions)
```

---

## 🎯 HARNESS COMPLETION — PR TO MAIN

Ship when: all features in `pm33-agent-progress.json` are `completed`, `init.sh` + `pre-commit-validation.sh` pass on the harness branch.

The flow is auto-complete by default — the human reviews inline during the session and can interrupt at any point. If they don't interrupt, coordinator runs the full sequence (PR → independent review → merge → cleanup) without further prompting.

### 1. Push + open PR

```bash
git push -u origin "harness/${HARNESS_ID}"
PR=$(gh pr create --base main --head "harness/${HARNESS_ID}" \
  --title "harness(${HARNESS_ID}): <summary>" \
  --body-file docs/frameworks/agent-state-${HARNESS_ID}/README.md | tail -1)
PR_NUM=$(gh pr view "$PR" --json number -q .number)
```

Mark the harness work_item (and any covered features) as `in_review` if tracked in PM33:

```typescript
// Best-effort — skip if no PM33 ID is associated with this harness/phase
pm33_update_work_item({ id: "<HARNESS_OR_PHASE_WORK_ITEM_ID>", status: "in_review" })
```

### 2. Dispatch the PR-review agent (independent judgment)

```typescript
Task({
  subagent_type: "code-reviewer",
  description: "Review PR for harness/${HARNESS_ID}",
  prompt: `Independently review PR ${PR_NUM} (${PR}).

  Inspect:  gh pr view ${PR_NUM}, gh pr diff ${PR_NUM}
  Context:  docs/frameworks/agent-state-${HARNESS_ID}/README.md (the harness intent)
  Focus on: correctness, hidden side-effects, breaking changes, security, schema/auth
            implications, test coverage gaps, alignment with the harness README.
  Post:     gh pr review ${PR_NUM} --approve
       OR:  gh pr review ${PR_NUM} --request-changes -b "<findings>"`
})
```

### 3. Merge if approved, otherwise dispatch a specialist to triage + fix

If approved: squash-merge, delete branch, clean up worktree (terminal).

If changes requested: dispatch a specialist (typically `backend-architect`; pick `frontend-developer` for client-only, `database-admin` for schema, `security-auditor` for auth) to read the reviewer's comments, agree or push back, fix if needed, then push the fix. Re-fetch the review decision and loop. Cap at 3 iterations before surfacing to the human.

```bash
DECISION=$(gh pr view "$PR_NUM" --json reviewDecision -q .reviewDecision)
ITER=0
while [ "$DECISION" != "APPROVED" ] && [ $ITER -lt 3 ]; do
  ITER=$((ITER + 1))
  echo "Iteration $ITER: reviewer requested changes — dispatching specialist."
  # → Task dispatch below
  # After specialist finishes (fix pushed OR pushback comment posted):
  DECISION=$(gh pr view "$PR_NUM" --json reviewDecision -q .reviewDecision)
done

if [ "$DECISION" = "APPROVED" ]; then
  gh pr merge --squash --delete-branch "$PR_NUM"
  cd "$(git rev-parse --git-common-dir | sed 's,/\.git$,,')"
  git worktree remove ".claude/worktrees/harness-${HARNESS_ID}"
  git branch -D "harness/${HARNESS_ID}"
  # Best-effort: mark covered PM33 work_items as done (skip if untracked)
  # pm33_update_work_item id=<HARNESS_OR_PHASE_WORK_ITEM_ID> status='done'
  echo "✅ harness/${HARNESS_ID} merged + cleaned up."
else
  echo "⏸  After $ITER iterations the PR is not approved. Human review needed."
  gh pr view "$PR_NUM" --comments
fi
```

Specialist dispatch inside the loop:

```typescript
Task({
  subagent_type: "backend-architect",   // or frontend-developer / database-admin / security-auditor
  description: "Triage + fix reviewer feedback on harness/${HARNESS_ID}",
  prompt: `Triage reviewer feedback on PR ${PR_NUM}.

  Working directory: <full path to .claude/worktrees/harness-${HARNESS_ID}>
  Branch: harness/${HARNESS_ID}
  Agent ID: ${HARNESS_ID}-fix-${ITER}   (export CLAUDE_AGENT_ID before any git)

  Steps:
    1. gh pr view ${PR_NUM} --comments   # read every reviewer comment
    2. For each comment, decide: is it correct?
         - YES  → apply the fix, commit with explicit paths, push to harness/${HARNESS_ID}
                 then re-request review:  gh pr review ${PR_NUM} --comment -b "Addressed: <summary>"
         - NO   → reply to the comment explaining intent/disagreement; do NOT push code
                 then escalate: gh pr comment ${PR_NUM} -b "Pushback on <item> — see thread. Awaiting human decision."
    3. Re-dispatch the reviewer (the coordinator loop will do this automatically on next iteration).

  Hard rules:
    - Do not merge.
    - Do not edit files outside the scope of the reviewer's comments.
    - If the reviewer's concern requires schema/auth/security changes beyond your specialty, post a comment requesting a different specialist and stop.`
})
```

After 3 iterations without approval, the loop exits and the human sees the PR with all the back-and-forth on it.

**Multi-phase harnesses**: each shippable phase gets its own PR cycle. The harness branch only gets deleted after the final phase merges; the `git worktree remove` / `git branch -D` block runs once at end-of-harness, not per-phase.

---

## 🎯 SUCCESS CRITERIA

**Session is successful when**:
- ✅ At least 1 feature moved from pending → completed
- ✅ All quality gates passed for completed features
- ✅ progress.json updated with timestamps and git commits
- ✅ claude-progress.txt appended with session summary
- ✅ Specialist followed TDD cycle (RED → GREEN → REFACTOR → DELIVERY)
- ✅ All tests passing for completed features
- ✅ Clean git commit for each feature

---

## 📚 REFERENCE

**Complete Documentation**:
- Harness Framework: `/docs/frameworks/LONG_RUNNING_AGENT_FRAMEWORK.md`
- Coordinator Guide: `/docs/agents/HARNESS_COORDINATOR_GUIDE.md`
- Agent Definition: `/agents/harness-coordinator.md`

**Related Skills**:
- `harness-discipline` - For specialists implementing features (not for coordinators)

**Tools You Use**:
- Task (launch specialists)
- Read (read progress files)
- Bash (run init.sh, jq queries, git commands)
- Grep (search for feature context)
- TodoWrite (optional - track coordination tasks)
