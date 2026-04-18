# Use Cases & Scenarios

## Core Use Cases

### 1. Code Editing & Refactoring

**Scenario**: "Refactor the authentication module to use JWT instead of sessions"

```
User → "Refactor auth to use JWT"
         │
         ▼
Claude reads relevant files:
  Read(src/auth/session.ts)
  Read(src/middleware/auth.ts)
  Glob(src/**/*auth*.ts)
         │
         ▼
Claude analyzes and plans:
  [thinks about approach]
         │
         ▼
Claude edits files:
  Edit(src/auth/session.ts)  [permission: ASK → Allow]
  Write(src/auth/jwt.ts)     [permission: ASK → Allow]
  Edit(src/middleware/auth.ts)
         │
         ▼
Claude verifies:
  Bash(npm test src/auth/)
         │
         ▼
Reports completion with summary of changes
```

**Key tools**: `Read`, `Edit`, `Write`, `Glob`, `Grep`, `Bash`

---

### 2. Bug Investigation & Fix

**Scenario**: "The login is failing with a 401 error — why?"

```
User → "Login failing with 401"
         │
         ▼
Claude investigates:
  Bash(git log --oneline -20)
  Grep(pattern="401|unauthorized", glob="src/**/*.ts")
  Read(src/auth/handler.ts)
  Read(src/middleware/validate.ts)
         │
         ▼
Claude identifies issue:
  "Line 47 in validate.ts — token expiry check is off by 1 day"
         │
         ▼
Claude fixes:
  Edit(src/middleware/validate.ts)
         │
         ▼
Claude tests:
  Bash(npm test src/middleware/)
         │
         ▼
Reports: "Fixed token expiry bug in validate.ts:47"
```

---

### 3. Code Review (via /review skill)

**Scenario**: "Review my PR before I submit"

```
User → /review
         │
         ▼
SkillTool invokes review skill prompt
         │
         ▼
Claude executes review skill:
  Bash(git diff main...HEAD)
  Bash(git log main...HEAD --oneline)
  Read(changed files)
         │
         ▼
Claude analyzes:
  - Code quality
  - Security issues
  - Performance concerns
  - Test coverage
  - Documentation
         │
         ▼
Reports structured review with
  - Summary of changes
  - Issues found (by severity)
  - Suggestions
  - Overall assessment
```

---

### 4. Test Generation

**Scenario**: "Write tests for the PaymentService class"

```
User → "Write tests for PaymentService"
         │
         ▼
Claude reads source:
  Read(src/services/PaymentService.ts)
  Read(src/services/PaymentService.test.ts)  [if exists]
  Glob(src/services/**/*.test.ts)  [check patterns]
         │
         ▼
Claude analyzes:
  - Public interface
  - Edge cases
  - Error scenarios
  - Dependencies to mock
         │
         ▼
Claude writes tests:
  Write(src/services/PaymentService.test.ts)
         │
         ▼
Claude runs tests:
  Bash(npm test src/services/PaymentService.test.ts)
         │
         ▼
Iterates if tests fail
```

---

### 5. Codebase Exploration (Explore agent)

**Scenario**: "How does the authentication system work?"

```
User → "Explain the auth system"
         │
         ▼
Main agent spawns Explore sub-agent:
  AgentTool(
    subagent_type: "explore",
    prompt: "Map the authentication system architecture"
  )
         │
         ▼
Explore agent (restricted tools: Read, Glob, Grep only):
  Glob(src/**/*auth*.ts)
  Read(src/auth/index.ts)
  Read(src/middleware/auth.ts)
  Grep(pattern="requireAuth|isAuthenticated", glob="src/**/*.ts")
  [finds all usages, builds mental map]
         │
         ▼
Explore agent returns:
  "Authentication uses JWT. Flow: 
   1. Login → src/auth/handler.ts:createToken()
   2. Middleware → src/middleware/auth.ts validates on each request
   3. Token refresh → src/auth/refresh.ts
   Used in 23 routes across src/routes/"
         │
         ▼
Main agent synthesizes and explains to user
```

---

### 6. Database Migration

**Scenario**: "Add an 'email_verified' column to the users table"

```
User → "Add email_verified to users table"
         │
         ▼
Claude explores:
  Glob(migrations/**/*.sql)
  Read(src/models/User.ts)
         │
         ▼
Claude creates migration:
  Write(migrations/20260417_add_email_verified.sql)
  Edit(src/models/User.ts)
  Edit(src/types/User.ts)
         │
         ▼
Claude runs migration:
  Bash(npm run db:migrate)
         │
         ▼
Claude updates tests:
  Edit(src/models/User.test.ts)
         │
         ▼
Claude verifies:
  Bash(npm test src/models/)
```

---

### 7. Multi-Agent Parallel Analysis

**Scenario**: "Audit the entire codebase for security vulnerabilities"

```
User → "Security audit the codebase"
         │
         ▼
Main agent (coordinator mode) spawns parallel workers:
  
  AgentTool(name="sql-injection", background=true,
    prompt="Find SQL injection vulnerabilities in src/")
  
  AgentTool(name="xss-check", background=true,
    prompt="Find XSS vulnerabilities in src/")
  
  AgentTool(name="auth-audit", background=true,
    prompt="Audit authentication and authorization")
  
  AgentTool(name="deps-check", background=true,
    prompt="Check for vulnerable npm dependencies")
         │
         ▼
[All four run in parallel — 4x faster than sequential]
         │
         ▼
Main agent receives all results:
  TaskList() → all completed
  Synthesizes findings
         │
         ▼
Reports comprehensive security audit
```

---

### 8. Documentation Generation

**Scenario**: "Document the API endpoints"

```
User → "Generate API docs"
         │
         ▼
Claude explores:
  Glob(src/routes/**/*.ts)
  Read(src/routes/index.ts)
         │
         ▼
Claude reads each route file, extracts:
  - Endpoint path and method
  - Request parameters/body
  - Response format
  - Auth requirements
         │
         ▼
Claude writes documentation:
  Write(docs/api.md)
         │
         ▼
Optionally: 
  Write(openapi.yaml)  [if OpenAPI format requested]
```

---

### 9. CI/CD Pipeline Debugging

**Scenario**: "Why is the CI failing?"

```
User → "CI is failing, fix it"
         │
         ▼
Claude fetches CI output:
  WebFetch(https://github.com/.../actions/runs/...)
  [or reads local CI output file]
         │
         ▼
Claude investigates:
  Bash(npm test -- --verbose 2>&1 | tail -50)
  Read(failing test file)
         │
         ▼
Claude identifies: "Test expects YYYY-MM-DD format, 
  but code now returns ISO 8601 with timezone"
         │
         ▼
Claude fixes:
  Edit(src/utils/formatDate.ts)
  Edit(src/utils/formatDate.test.ts)
         │
         ▼
Claude verifies:
  Bash(npm test src/utils/formatDate.test.ts)
```

---

### 10. New Feature Implementation

**Scenario**: "Add user avatar upload support"

```
User → "Add avatar upload feature"
         │
         ▼
Claude plans (if in plan mode):
  EnterPlanMode()
  [thinks through approach]
  [lists files to create/modify]
  ExitPlanMode() → user approves
         │
         ▼
Claude implements:
  Read(src/routes/user.ts)            [understand existing patterns]
  Read(src/services/storage.ts)       [understand storage service]
  
  Write(src/services/avatarService.ts) [new service]
  Edit(src/routes/user.ts)            [add route]
  Edit(src/models/User.ts)            [add avatarUrl field]
  Write(migrations/add_avatar_url.sql) [DB migration]
         │
         ▼
Claude tests:
  Bash(npm test src/routes/user.test.ts)
  Bash(npm run db:migrate)
         │
         ▼
Reports: "Avatar upload implemented. 
  Upload: POST /api/users/:id/avatar
  Storage: S3 via avatarService.ts
  Migration: applied"
```

---

## Edge Cases & Special Behaviors

### Interruption Handling

User presses `Ctrl+C` while Claude is working:
1. `AbortController.abort()` is called
2. Currently executing tool is cancelled (if it supports abort)
3. Partial results are preserved in message history
4. Claude receives `AbortError` and reports interruption
5. User can continue the conversation

### Long Output Handling

When tool output exceeds display limits:
1. Output is stored in `~/.claude/tmp/<session>/tool-results/`
2. A preview (first N bytes) is shown inline
3. Model receives the full content via reference
4. UI shows "[output stored — showing preview]"

### Image Input

When user pastes an image:
1. Captured as base64 via clipboard API
2. Validated (size, format)
3. Injected as `image` content block in UserMessage
4. Claude can see and describe the image
5. Can reference image in subsequent tool calls

### Long-Running Bash Commands

For commands that take > 2 seconds:
1. Progress indicator shown (spinner + "Running...")
2. Can be moved to background (auto-background feature)
3. User can continue typing while command runs
4. Notification when complete
5. Can be cancelled with `Ctrl+C`

### Session Resume

```bash
claude --resume  # Resume most recent session
claude --resume <session-id>  # Resume specific session
```

1. Loads transcript from `~/.claude/projects/<hash>/sessions/<id>.jsonl`
2. Recreates QueryEngine with prior messages
3. User can continue where they left off
4. New responses are appended to the session

### Headless / Script Mode

```bash
claude -p "Add error handling to all API routes" --output json
```

1. No interactive UI
2. Runs query to completion
3. Outputs result as JSON to stdout
4. Useful for CI/CD automation
5. Can pipe file content: `cat error.log | claude "What's causing this?"`
