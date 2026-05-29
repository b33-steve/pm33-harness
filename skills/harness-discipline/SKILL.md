---
name: harness-discipline
description: TDD discipline and progress tracking for multi-session harness projects. Apply this skill when working on any harness project (16+ hours, 3+ sessions) to ensure consistent TDD cycle, progress tracking, and reporting standards.
whenToUse: When working on harness projects in docs/frameworks/agent-state-{PROJECT-ID}/ or when harness-coordinator assigns you work. Apply regardless of your specialist role (backend, frontend, database, etc.).
triggers: ["harness project", "working on harness", "implement harness feature", "coordinator assigned me"]
relatedDocs: ["/docs/frameworks/LONG_RUNNING_AGENT_FRAMEWORK.md", "/docs/agents/HARNESS_COORDINATOR_GUIDE.md"]
---

# Harness Discipline Skill

**Purpose**: Apply consistent TDD cycle, progress tracking, and reporting standards when working on multi-session harness projects.

---

## 🔑 HOW TO USE THIS SKILL

**If you are a SPECIALIST IMPLEMENTER** (backend-architect, frontend-developer, database-admin, etc.):

**Step 1 - Load this skill**:
```
Skill({ skill: "harness-discipline" })
```

**🚨 IMPORTANT**: Use skill name exactly as shown:
- ✅ CORRECT: `Skill({ skill: "harness-discipline" })`
- ❌ WRONG: `Skill(compounding-engineering:harness-discipline)` (no plugin prefix)
- ❌ WRONG: `Skill({ skill: "compounding-engineering:harness-discipline" })` (no plugin prefix)
- ❌ WRONG: Any namespace or prefix

**Step 2 - Follow the workflow in this skill**:
- Read context (startup section below)
- Apply TDD cycle (RED → GREEN → REFACTOR → DELIVERY)
- Update progress tracking (progress.json + claude-progress.txt)
- Report back to coordinator with evidence

**Use this skill when**:
- ✅ Working in `docs/frameworks/agent-state-{PROJECT-ID}/` directory
- ✅ Assigned work by harness-coordinator via Task tool
- ✅ Implementing features tracked in `pm33-agent-progress.json`
- ✅ Multi-session project (16+ hours, 3+ sessions)

**🚨 IF YOU ARE THE COORDINATOR**:
- ❌ DO NOT use this skill yourself
- ❌ This skill is for IMPLEMENTERS, not COORDINATORS
- ✅ Your role: Launch specialists via Task tool with this skill reference
- ✅ See `agents/harness-coordinator.md` for your workflow

---

## 🎯 CORE PRINCIPLE

**You maintain your specialist identity** (backend-architect, frontend-developer, etc.) while applying harness discipline to your work.

**Harness discipline = TDD cycle + Progress tracking + Evidence-based reporting**

**You are the IMPLEMENTER**: You write code, run tests, commit changes. The coordinator orchestrates; you execute.

---

## 📋 STARTUP: Read Context (5 min)

**MANDATORY - Before starting any harness work**:

1. **Read progress.json**:
```bash
cat docs/frameworks/agent-state-{PROJECT-ID}/pm33-agent-progress.json
# Find your assigned feature (status: "in_progress" or next "pending")
```

2. **Read last 3 sessions**:
```bash
tail -100 docs/frameworks/agent-state-{PROJECT-ID}/claude-progress.txt
# Understand what was completed, what's next
```

3. **Check git log**:
```bash
git log --oneline --since="7 days ago" | head -20
# See recent commits related to this harness
```

4. **Read harness README**:
```bash
cat docs/frameworks/agent-state-{PROJECT-ID}/README.md
# Understand quality standards, acceptance criteria
```

5. **Read coordinator's context** (if provided):
- Technical context (feature spec, code samples, schema)
- Documentation references (wireframes, architecture)
- Quality standards (test coverage, performance targets)
- Test scenarios (from progress.json)
- Validation instructions (gates to pass, reporting format)

---

## 🧠 MEMORY MANAGEMENT (MANDATORY)

**🚨 CRITICAL**: Test execution has strict memory limits to prevent system exhaustion.

### Memory Limits

| Resource | Limit | Configuration |
|----------|-------|---------------|
| **Vitest workers** | 2 max | `maxForks: 2` in vitest.config.ts |
| **Per-worker memory** | 2GB | `--max-old-space-size=2048` |
| **Total test memory** | 4GB max | 2 workers × 2GB each |
| **TypeScript compilation** | 3GB | `NODE_OPTIONS=--max-old-space-size=3072` |

### Mandatory Test Command

**🚨 ALWAYS use `npm run test:locked`** - NEVER use `npm test` or `npm run test:unit` directly.

```bash
# ✅ CORRECT - Uses file-based lock to prevent concurrent tests
npm run test:locked -- [feature].test.ts

# ❌ WRONG - No coordination, can cause memory exhaustion
npm test -- [feature].test.ts
npm run test:unit -- [feature].test.ts
```

**Why `test:locked`**:
- Creates `/tmp/pm33-tests.lock` with process ID
- Prevents multiple agents from running tests simultaneously
- Waits up to 5 minutes if another test is running
- Auto-cleanup on exit (success or failure)

### Sequential Validation Pattern

**Run validation commands ONE AT A TIME**:

```bash
# ✅ CORRECT - Sequential with &&
npm run type-check && npm run test:locked && npm run lint

# ❌ WRONG - Parallel execution causes 9GB+ memory usage
npm run type-check & npm run test:unit & npm run lint
```

### Post-Test Cleanup

**After completing tests, clean up Node processes**:

```bash
npm run cleanup:node-processes
```

### Multi-Agent Coordination

**THE PROBLEM**: Multiple agents running tests simultaneously defeats memory limits.

**MANDATORY RULE**:
- **ONLY ONE AGENT may run tests at a time**
- If tests are already running, **WAIT** or **SKIP**
- Check with: `ps aux | grep vitest`

---

## 🔄 TDD CYCLE (MANDATORY - Every Feature)

### Phase 1: RED (Write Failing Test First)

**Before writing ANY implementation code**:

```bash
# 1. Create test file or add to existing
# 2. Write test for new feature
# 3. Run test - EXPECT IT TO FAIL

npm run test:locked -- [feature].test.ts

# ✅ Success criteria: Test fails (red)
# ❌ If test passes: You're testing existing functionality, not new feature
```

**Why RED matters**: Proves test actually validates the feature (prevents false confidence)

### Phase 2: GREEN (Make Test Pass)

**Before writing GREEN tests for any middleware/service that touches the DB**:
- [ ] Does this code execute SQL (raw or via Drizzle ORM)? If yes → use real test DB, NOT mocked queries
- [ ] If you mock at the DB layer, your test verifies your mock, not your code
- [ ] If real-DB tests are infeasible for this harness, document the gap in TECHNICAL_DEBT.md — DO NOT silently substitute mocks

**See "Mock vs Real DB" section below for patterns and decision tree.**

**Write minimal code to pass the test**:

```bash
# 1. Implement feature (simplest possible code)
# 2. Run test - EXPECT IT TO PASS

npm run test:locked -- [feature].test.ts

# ✅ Success criteria: Test passes (green)
# ❌ If test fails: Debug implementation, don't skip to refactor
```

**Why GREEN matters**: Proves implementation works (establishes working baseline)

### Phase 3: REFACTOR (Clean Up)

**Improve code quality while keeping tests green**:

```bash
# 1. Remove duplication, improve naming, simplify logic
# 2. Run test - EXPECT IT TO STILL PASS

npm run test:locked -- [feature].test.ts

# ✅ Success criteria: Tests still pass after refactoring
# ❌ If tests fail: Refactoring broke functionality, revert and retry
```

**Why REFACTOR matters**: Maintains code quality without sacrificing working functionality

### Phase 4: DELIVERY (Full Validation)

**Run all quality gates before commit**:

```bash
# Run pre-commit validation script
./docs/frameworks/agent-state-{PROJECT-ID}/pre-commit-validation.sh

# Expected: All 10 gates pass
# 1. Schema drift: ✅
# 2. TypeScript compilation: ✅
# 3. ESLint validation: ✅
# 4. Zod contracts: ✅
# 5. Unit tests: ✅
# 6. Integration tests: ✅
# 7. E2E tests (critical paths): ✅
# 8. API validation: ✅
# 9. Documentation sync: ✅
# 10. Progress tracking: ✅
```

**If any gate fails**:
- Fix issue before proceeding
- Do NOT skip gates
- Do NOT commit until all gates pass

**When all gates pass**:
```bash
# Ensure you are inside your own worktree (NOT the shared main repo).
# `git add .` on the shared main worktree absorbs files written by other
# concurrent specialists into your commit. Two real incidents (2026-05-14
# AUTH-ROLE-CONSOLIDATION-001 and 2026-05-15 ABSORPTION-002) document this.
#
# If you have not yet entered a worktree (see "Worktree isolation" in the
# coordinator's launch prompt), do it now:
if [ "$(git rev-parse --absolute-git-dir)" = "$(git rev-parse --path-format=absolute --git-common-dir)" ]; then
  AGENT_ID="$(./scripts/git/agent-id.sh)"
  ./scripts/git/spawn-agent-worktree.sh "$AGENT_ID"
  cd ".claude/worktrees/agent-$AGENT_ID"
  export CLAUDE_AGENT_ID="$AGENT_ID"
fi

# Now `git add .` is safe — the worktree only sees your own files.
# Commit ONE feature only.
git add .
git commit -m "FEAT-XXX: [description] - All validation gates PASSED"
```

---

## 🗄️ MOCK VS REAL DB — CRITICAL DISTINCTION

### The BUG-001 Lesson

**Real incident**: Three layers of testing all passed because mocks synthesized a `deleted_at` field that didn't exist in the schema. The field existed in tests (via mocks), but not in production. Routes that checked `deleted_at` returned 500 errors on every request.

**Root cause**: Mocks allowed test authors to imagine schema fields without validating against reality. The gap was invisible until production.

### Decision Tree

**Use REAL TEST DB if**:
1. Code executes SQL (raw: `db.execute(sql\`...\`)`or ORM: `db.select().from(...)`)
2. Code calls a function that executes SQL
3. Schema validation matters (table structure, column types, constraints)

**Use MOCKS if**:
1. Code transforms data from elsewhere (data → format conversion)
2. Code renders UI from props/API responses (component isolation test)
3. Code has zero DB dependencies

**When in doubt: REAL DB is safer.**

### Categories with Examples

**🔴 ALWAYS REAL DB**:
- Express middleware that runs SQL: `(req, res, next) => { const rows = await db.select()... }`
- Service-layer functions with dynamic queries: `getWorkspaceRisk(workspaceId) => db.select().from(workItems).where(...)`
- Auth/access-control logic that joins tables: permission checks against database state
- Migrations and schema validation

**🟡 VALIDATE FIRST, THEN DECIDE**:
- Hooks that call API endpoints: Mock the API response (not the DB), test component isolation
- Utility functions that accept data: Mock the input data, test transformation logic
- Middleware that depends on a service: Real test DB if service queries; mock if service is pure logic

**🟢 ALWAYS MOCKS**:
- Component rendering tests: Mock props, test JSX output
- Data transformers: Mock input, test output format
- Validators and formatters: Mock data, test logic
- UI state management: Mock API calls, test state transitions

### Anti-Patterns (What NOT to Do)

```typescript
// ❌ WRONG: Mocks a query at the db layer
vi.mock('../db', () => ({
  execute: vi.fn().mockResolvedValue({ rows: [
    { id: '123', deleted_at: null }  // synthesized schema
  ]}),
}));

// ❌ WRONG: Mocks individual middleware SQL
db.execute = vi.fn().mockResolvedValue({ 
  rows: [{ workspace_id: '...', deleted_at: null }]  // imagined fields
});

// ❌ WRONG: Mocks Drizzle query builder for routes/services
vi.mock('drizzle-orm', () => ({
  eq: vi.fn(),
  and: vi.fn(),
  select: vi.fn().mockReturnValue({
    from: vi.fn().mockReturnValue({
      where: vi.fn().mockResolvedValue([{ ... }])
    })
  })
}));
```

### Correct Patterns for Fast Real-DB Tests

**Pattern 1: Transaction Rollback (Fastest)**
```typescript
// Wrap each test in a transaction, rollback after
beforeEach(() => {
  client.query('BEGIN');
});

afterEach(() => {
  client.query('ROLLBACK');
});

test('middlewareValidatesWorkspace', async () => {
  // Insert real test data
  await db.insert(workspaces).values({ id: 'test-1', ... });
  
  // Run middleware, verify database behavior
  const result = await middleware(req, res, next);
  expect(result).toBe(expectedOutcome);
});
```

**Pattern 2: Deterministic Test Database**
```typescript
// Use test fixtures in OrbStack container
beforeAll(() => {
  // Seed test data once per suite
  seedTestDatabase({
    workspaces: [{ id: 'test-workspace', ... }],
    users: [{ id: 'test-user', workspaceId: 'test-workspace' }]
  });
});

afterEach(() => {
  // Clean up per test
  db.delete(workItems).where(eq(workItems.workspaceId, 'test-workspace'));
});

test('middlewareChecksAccess', async () => {
  const { req, res } = createTestRequest({ workspaceId: 'test-workspace' });
  await middleware(req, res, next);
  expect(res.status).toBe(200);
});
```

**Pattern 3: Reuse OrbStack Container**
- Tests run against `pm33-postgres` container (same DB as dev)
- No setup/teardown overhead
- Schema always matches reality
- Configure in test harness: `docs/frameworks/agent-state-{PROJECT-ID}/init.sh`

### When Real-DB Tests Are Infeasible

If real-DB tests cannot work for this harness (e.g., no test container available, integration test limitations), document the gap:

```markdown
### DB-TEST-001: Mocked queries in [service name]

**Issue**: [Middleware/service name] runs SQL but we mock queries for speed

**Reason**: [Test container not available / integration tests fail with real DB / other constraint]

**Mitigation**: [Use [Pattern] OR document manual validation OR create follow-up task]

**Risk**: Mocks may diverge from schema. Validate schema manually before deployment.

**Follow-up**: [Link to task to enable real-DB tests or schema validation]
```

### Reference

- **Complete convention doc**: `docs/conventions/TESTING_REAL_DB_VS_MOCKS.md`
- **BUG-001 post-mortem**: `/docs/incidents/BUG-001-post-mortem.md`
- **Test patterns in PM33**: `server/__tests__/` (look for transaction rollback patterns)

---

## 📊 PROGRESS TRACKING (After Each Feature)

### Update progress.json

**After completing DELIVERY phase**:

```json
// In docs/frameworks/agent-state-{PROJECT-ID}/pm33-agent-progress.json
{
  "id": "FEAT-XXX",
  "status": "completed",  // Change from "in_progress" to "completed"
  "tddPhases": {
    "RED": "completed",
    "GREEN": "completed",
    "REFACTOR": "completed",
    "DELIVERY": "completed"
  },
  "validationResults": {
    "testCoverage": "97%",
    "gitCommit": "abc1234",
    "completedAt": "2025-12-09T15:30:00Z"
  }
}
```

### Update claude-progress.txt

**Append session entry** (do this ONCE per feature completion):

```markdown
## Session [N] - YYYY-MM-DD

**Feature Completed**: FEAT-XXX - [Feature name]

**TDD Phases**:
- RED: ✅ Test written, initially failed
- GREEN: ✅ Implementation complete, test passed
- REFACTOR: ✅ Code cleaned up, tests still pass
- DELIVERY: ✅ All validation gates passed

**Validation Results**:
- Test Coverage: 97% (target: ≥95%)
- TypeScript: ✅ No errors
- ESLint: ✅ No warnings
- Performance: [metric if applicable]
- Git Commit: abc1234

**Files Modified**:
- server/routes/example.ts (lines 45-67)
- server/services/exampleService.ts (lines 12-34)
- tests/example.test.ts (lines 8-56)

**Next Feature**: FEAT-YYY - [Next feature name] (estimated 4 hours)
```

### Update UTT_MASTER_INDEX.md

**Document completed features** (do this ONCE per feature completion):

```bash
# Add entry to /docs/reference/UTT_MASTER_INDEX.md
# Find the appropriate section for your harness project
```

**Entry Format**:
```markdown
### [Harness Project Name] - FEAT-XXX: [Feature Name]

**Status**: ✅ Completed
**Completed**: YYYY-MM-DD
**Specialist**: [backend-architect/frontend-developer/etc.]
**Git Commit**: abc1234

**Summary**: [1-2 sentence description of what was implemented]

**Acceptance Criteria Met**:
- [Criterion 1]
- [Criterion 2]
- [Criterion 3]

**Validation Results**:
- Test Coverage: 97%
- TypeScript: ✅ No errors
- ESLint: ✅ No warnings
- All 10 quality gates: ✅ PASSED

**Files Modified**:
- `server/routes/example.ts` (lines 45-67)
- `server/services/exampleService.ts` (lines 12-34)
- `tests/example.test.ts` (lines 8-56)

**Related Documentation**: [Links to specs, wireframes, or technical docs]
```

**Why This Matters**: UTT_MASTER_INDEX.md provides project-wide visibility of all completed harness work, enabling other agents and team members to quickly find implementations without searching through git history.

### Update TECHNICAL_DEBT.md (When Necessary)

**When to document technical debt**:
- 🔴 **P0 (Critical)**: Security vulnerabilities, data loss risks, production blockers
- 🟡 **P1 (High)**: Performance issues, maintainability concerns, breaking API changes needed
- 🟢 **P2 (Medium)**: Code quality improvements, missing tests, documentation gaps

**Document in `/docs/reference/TECHNICAL_DEBT.md`**:

```markdown
### [PRIORITY]-XXX: [Brief description]

**Priority**: P0/P1/P2
**Discovered**: YYYY-MM-DD
**Discovered During**: [Harness project name] - FEAT-XXX
**Specialist**: [your-role]

**Impact**: [What breaks or degrades if not fixed]

**Root Cause**: [Why this issue exists]

**Recommended Fix**: [Specific steps to resolve]

**Workaround** (if applicable): [Temporary solution in place]

**Requires**: [Architect approval / DBA review / Security audit / etc.]
```

**Examples**:

**P0 Critical**:
```markdown
### P0-RED-015: Authentication bypass in workspace switching

**Priority**: P0
**Discovered**: 2025-12-09
**Discovered During**: UTT-STRAT-001 - FEAT-008
**Specialist**: backend-architect

**Impact**: Users can access workspaces without proper authorization, violating tenant isolation

**Root Cause**: Middleware order allows session validation before workspace permission check

**Recommended Fix**:
1. Reorder middleware: tenant → workspace → auth
2. Add workspace permission validation in middleware
3. Add integration tests for cross-workspace access attempts

**Requires**: Architect approval + Security audit
```

**P1 High**:
```markdown
### P1-YELLOW-023: N+1 query in strategy source listing

**Priority**: P1
**Discovered**: 2025-12-09
**Discovered During**: UTT-STRAT-001 - FEAT-012
**Specialist**: backend-architect

**Impact**: Strategy page loads 2000ms slower with 50+ sources (target: <500ms)

**Root Cause**: Missing JOIN in query, fetching related data in loop

**Recommended Fix**:
1. Add eager loading with Drizzle .with() clause
2. Add database index on foreign key
3. Add performance test to prevent regression

**Requires**: DBA review for index strategy
```

**Why This Matters**: Proper technical debt documentation prevents issues from being forgotten, enables prioritization, and creates audit trail for architectural decisions.

---

## 💬 REPORTING BACK (To Coordinator or User)

**Status Report Template**:

```markdown
## Feature FEAT-XXX: [name] - ✅ COMPLETE

**Specialist**: [your-role: backend-architect, frontend-developer, etc.]

**TDD Cycle**:
- RED: ✅ Test written, initially failed as expected
- GREEN: ✅ Feature implemented, test passed
- REFACTOR: ✅ Code cleaned up, tests still green
- DELIVERY: ✅ All 10 quality gates passed

**Validation Evidence**:
- Test Coverage: 97% (target: ≥95%)
- TypeScript: ✅ 0 errors
- ESLint: ✅ 0 warnings
- Performance: [metric] (target: [target])
- Security: ✅ Input validation, tenant isolation verified
- Git Commit: abc1234

**Files Modified**:
- [file1:lines]
- [file2:lines]
- [file3:lines]

**Issues Discovered**: [None / See escalation below]

**Progress Update**:
- progress.json updated: FEAT-XXX marked completed
- claude-progress.txt appended: Session entry added
- UTT_MASTER_INDEX.md updated: Feature documented
- TECHNICAL_DEBT.md updated: [None / Issues P0-XXX, P1-YYY documented]
- Git commit successful: abc1234

**Ready for**: Next feature (FEAT-YYY) or coordinator review
```

---

## 🚨 ESCALATION PROTOCOL

**When you need guidance or discover issues during implementation**:

### 🔵 CONSULT EXPERT SPECIALISTS (WHEN UNCERTAIN)

**🚨 CRITICAL**: If you are unsure about the path forward, you MUST consult architect and/or DBA BEFORE continuing implementation.

**Consult backend-architect if uncertain about**:
- API design decisions (endpoints, request/response structure)
- Service boundaries (which service should own this logic)
- Architecture patterns (how to structure the code)
- Integration approaches (how to connect systems)
- Security implementation (authentication, authorization patterns)

**Consult database-admin if uncertain about**:
- Database schema changes (tables, columns, indexes)
- Migration strategies (how to safely alter schema)
- Query optimization (performance concerns)
- Data modeling decisions (normalization, relationships)
- Transaction boundaries (where to use transactions)

**Action**:
1. STOP implementation at current state
2. Use Task tool to launch architect and/or DBA for review:

```typescript
Task({
  subagent_type: 'backend-architect',  // or 'database-admin'
  description: 'Review [feature-name] architecture/design',
  prompt: `**REVIEW REQUEST FROM SPECIALIST**

I am implementing [feature-name] and need architectural guidance.

**Current Context**:
- Feature: [FEAT-XXX - description]
- Harness Project: docs/frameworks/agent-state-{PROJECT-ID}/
- Implementation Progress: [what I've completed so far]

**Uncertainty**:
[Describe what you're uncertain about - specific questions]

**Options Considered**:
1. [Option A - pros/cons]
2. [Option B - pros/cons]
3. [Option C - pros/cons]

**Documentation Reviewed**:
- [List files you've read: specs, existing code, documentation]

**Your Task**:
1. Review the documentation and existing code
2. Review the feature requirements in progress.json
3. Provide clear recommendation on path forward
4. Explain rationale for recommendation

**Report Back**:
- Recommended approach (which option or alternative)
- Rationale (why this approach is best)
- Implementation guidance (key steps to follow)
- Potential risks to watch for`
});
```

3. WAIT for architect/DBA response with recommendations
4. RESUME implementation following their guidance
5. Document their recommendation in commit message:
   ```
   FEAT-XXX: [description]

   Architecture reviewed by backend-architect: [recommendation summary]
   [implementation details]
   ```

**Why This Is Critical**:
- Prevents architectural mistakes that are costly to fix later
- Ensures database changes follow best practices
- Maintains system consistency and quality standards
- Leverages expert knowledge for complex decisions

**🚨 IF YOU DON'T ASK WHEN UNCERTAIN**: Coordinator will detect this and require architect/DBA review anyway, which delays the feature. Always ask proactively when uncertain.

---

### 🔴 RED (STOP WORK IMMEDIATELY)

**Stop if**:
- Security vulnerability discovered
- Data loss risk identified
- Production blocker found
- Breaking API change required

**Action**:
1. STOP all work immediately
2. Document in `docs/reference/TECHNICAL_DEBT.md`:
   ```markdown
   ### RED-XXX: [Brief description]
   **Priority**: P0 | **Discovered**: YYYY-MM-DD
   **Impact**: [What breaks if not fixed]
   **Requires**: Architect approval before proceeding
   ```
3. Report to coordinator/user: "🔴 RED issue discovered - work stopped, architect approval needed"
4. WAIT for approval before continuing

### 🟡 YELLOW (REPORT BUT CONTINUE)

**Report if**:
- Performance degradation noticed
- Maintainability concerns found
- Technical debt discovered
- Missing documentation identified

**Action**:
1. Document in `docs/reference/TECHNICAL_DEBT.md`:
   ```markdown
   ### YELLOW-XXX: [Brief description]
   **Priority**: P1/P2 | **Discovered**: YYYY-MM-DD
   **Impact**: [Degradation but not blocking]
   **Recommendation**: [Suggested fix]
   ```
2. Mention in status report
3. Continue with current feature

### 🟢 GREEN (FIX INLINE)

**Fix immediately if**:
- Minor code quality issues
- Missing comments
- Unused imports
- Style inconsistencies

**Action**:
1. Fix inline during REFACTOR phase
2. Mention in commit message
3. No separate documentation needed

---

## ✅ COMPLETION CHECKLIST

**Before reporting feature complete, verify**:

### Context
- [ ] Read progress.json and identified your feature
- [ ] Read last 3 sessions from claude-progress.txt
- [ ] Checked git log for recent work
- [ ] Read harness README for quality standards
- [ ] Read coordinator's context package (if provided)

### TDD Cycle
- [ ] RED: Test written, initially failed (used `npm run test:locked`)
- [ ] GREEN: Feature implemented, test passed (used `npm run test:locked`)
- [ ] REFACTOR: Code cleaned up, tests still pass (used `npm run test:locked`)
- [ ] DELIVERY: All 10 quality gates passed
- [ ] POST-TEST: Ran `npm run cleanup:node-processes`

### Quality
- [ ] Test coverage ≥95%
- [ ] Performance targets met
- [ ] Security requirements validated (input validation, tenant isolation)
- [ ] Acceptance criteria verified
- [ ] JSDoc comments added

### Progress Tracking
- [ ] progress.json updated (status: "completed", validation results)
- [ ] claude-progress.txt appended (session entry)
- [ ] UTT_MASTER_INDEX.md updated (feature documented)
- [ ] TECHNICAL_DEBT.md updated (if issues discovered)
- [ ] Git commit successful (one feature per commit)

### Reporting
- [ ] Status report prepared with evidence
- [ ] Issues escalated (if any) per protocol
- [ ] Coordinator/user notified of completion

---

## 🎓 KEY PRINCIPLES TO REMEMBER

### 1. One Feature Per Session
**Focus on single feature**, complete full TDD cycle, commit, then move to next.
- ✅ DO: FEAT-001 complete → commit → FEAT-002 start
- ❌ DON'T: Start FEAT-001 + FEAT-002 simultaneously

### 2. TDD Is Not Optional
**All 4 phases required** (RED → GREEN → REFACTOR → DELIVERY)
- ✅ DO: Write test first, see it fail, implement, refactor, validate
- ❌ DON'T: Skip RED phase, skip tests, defer validation

### 3. Evidence Over Claims
**Prove completion** with validation results, not just "I think it works"
- ✅ DO: "Test coverage: 97%, git commit: abc1234"
- ❌ DON'T: "Looks good, probably works"

### 4. Quality Gates Are Mandatory
**All gates must pass** before commit
- ✅ DO: Fix failures, rerun gates until all pass
- ❌ DON'T: Skip gates, commit with failures, defer fixes

### 5. Progress Tracking Is Critical
**Update progress files** after every feature
- ✅ DO: Update progress.json + claude-progress.txt immediately
- ❌ DON'T: Batch updates, forget to track, skip documentation

---

## 📚 REFERENCE

**Harness Framework**: `/docs/frameworks/LONG_RUNNING_AGENT_FRAMEWORK.md`
**Coordinator Guide**: `/docs/agents/HARNESS_COORDINATOR_GUIDE.md`
**Quality Gates**: Defined in `docs/frameworks/agent-state-{PROJECT-ID}/pre-commit-validation.sh`

---

## 💡 EXAMPLE SESSION

**Scenario**: You're backend-architect working on FEAT-003 in console-error-refactor harness

**Startup** (5 min):
```bash
# 1. Read context
cat docs/frameworks/agent-state-console-error-refactor/pm33-agent-progress.json
# → Found FEAT-003: "Implement error categorization service"

tail -100 docs/frameworks/agent-state-console-error-refactor/claude-progress.txt
# → Previous session completed FEAT-002, next is FEAT-003

# 2. Understand requirements
# → FEAT-003 acceptance criteria: categorize errors by severity, store metadata
```

**TDD Cycle** (45 min):
```bash
# RED: Write test
# Created: server/services/__tests__/errorCategorizationService.test.ts
# Test expects categorizeError() to return {severity, category, metadata}
npm run test:locked -- errorCategorizationService.test.ts
# ✅ Test fails (function doesn't exist yet)

# GREEN: Implement
# Created: server/services/errorCategorizationService.ts
# Implemented categorizeError() with basic logic
npm run test:locked -- errorCategorizationService.test.ts
# ✅ Test passes

# REFACTOR: Clean up
# Extracted severity detection to separate function
# Improved naming, added JSDoc comments
npm run test:locked -- errorCategorizationService.test.ts
# ✅ Tests still pass

# DELIVERY: Validate
./docs/frameworks/agent-state-console-error-refactor/pre-commit-validation.sh
# ✅ All 10 gates passed

# POST-TEST: Cleanup
npm run cleanup:node-processes
```

**Progress Tracking** (5 min):
```bash
# Update progress.json: FEAT-003 status → "completed"
# Append claude-progress.txt: Session entry with validation results
# Commit: git commit -m "FEAT-003: Implement error categorization - All gates PASSED"
```

**Reporting** (2 min):
```markdown
## Feature FEAT-003: Implement error categorization - ✅ COMPLETE

**Specialist**: backend-architect

**TDD Cycle**: All 4 phases completed
**Validation**: 98% coverage, 0 errors, git: def5678
**Files**: errorCategorizationService.ts, tests
**Next**: FEAT-004 (estimated 3 hours)
```

---

**Remember**: You are a specialist applying harness discipline, not a harness-specialist wrapping your work. Your expertise (backend, frontend, database) remains primary; this skill adds structure and accountability.
