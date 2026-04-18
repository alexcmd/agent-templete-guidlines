# Reproduction Blueprint: Build a Domain-Specific AI Agent

This guide provides a complete blueprint for building a Claude Code-like agent for a specific business domain, using the same architectural patterns.

---

## Architecture Overview

```
┌─────────────────────────────────────────────────────────┐
│                  YOUR DOMAIN AGENT                       │
│                                                          │
│  1. CLI Entry Point                                      │
│  2. Session Manager (QueryEngine equivalent)             │
│  3. Inner Loop (query.ts equivalent)                     │
│  4. Domain Tools (your business-specific tools)          │
│  5. Permission System                                    │
│  6. Memory System (domain knowledge)                     │
│  7. Configuration                                        │
│  8. UI Layer (terminal or web)                           │
└─────────────────────────────────────────────────────────┘
```

---

## Step 1: Define Your Domain

Before writing code, answer these questions:

**1. What domain are you building for?**
- Legal (contract analysis, case research)
- Healthcare (clinical documentation, diagnosis support)
- Finance (data analysis, report generation)
- DevOps (infrastructure management, deployment)
- Customer Support (ticket resolution, knowledge base)
- Data Science (EDA, model building)
- Manufacturing (process optimization, maintenance)

**2. What actions should your agent take?**
- Read-only (research, analysis, reporting)
- Create/modify (document generation, data entry)
- Execute (run queries, trigger workflows)
- Multi-step workflows (complex processes)

**3. Who are the users?**
- Technical (can approve complex operations)
- Non-technical (need simplified permission model)
- Automated (headless, no permission prompts)

---

## Step 2: Technology Stack Choices

### Option A: Build on Claude Code (Extend)

Add a plugin with domain-specific tools and commands:
```
└── .claude/
    ├── commands/
    │   └── my-domain-skill.md
    └── agents/
        └── domain-expert.md
```

**Pros**: Instant full Claude Code capability, just add domain layer
**Cons**: Tied to Claude Code's release cycle

### Option B: Build Custom Agent (Recommended for production)

Use the same patterns but purpose-built:

**Tech stack:**
```
Runtime:        Bun or Node.js (TypeScript)
UI:             Ink.js (terminal) or Next.js (web)
API client:     @anthropic-ai/sdk
Schema:         Zod
State:          React Context / Redux / Zustand
Config:         JSON files or environment variables
```

---

## Step 3: Core Architecture — Files to Create

```
my-agent/
├── src/
│   ├── main.ts                    Entry point
│   ├── Session.ts                 QueryEngine equivalent
│   ├── loop.ts                    Inner query loop
│   ├── Tool.ts                    Tool interface
│   ├── tools.ts                   Tool registry
│   ├── tools/                     Domain tool implementations
│   │   ├── DomainTool1/
│   │   ├── DomainTool2/
│   │   └── ...
│   ├── permissions/               Permission system
│   │   ├── canUseTool.ts
│   │   └── rules.ts
│   ├── memory/                    Domain knowledge injection
│   │   └── loadMemory.ts
│   ├── config/                    Configuration system
│   │   └── settings.ts
│   ├── commands/                  Slash commands
│   │   └── index.ts
│   ├── prompts/                   System prompt
│   │   └── systemPrompt.ts
│   └── ui/                        UI layer
│       ├── REPL.tsx               Main UI component
│       └── components/
└── .domain-agent/                 Config directory
    ├── settings.json
    ├── MEMORY.md
    └── commands/
```

---

## Step 4: Implement the Tool Interface

```typescript
// src/Tool.ts
import { z } from 'zod'

export type ToolInputJSONSchema = {
  type: 'object'
  properties?: Record<string, unknown>
  required?: string[]
}

export type ToolContext = {
  sessionId: string
  userId?: string
  config: AppConfig
  abortSignal: AbortSignal
  getState(): AppState
  setState(update: Partial<AppState>): void
}

export type Tool = {
  name: string
  description: string          // Shown to model — be very specific
  inputSchema: z.ZodObject<any>
  handler(
    input: Record<string, unknown>,
    context: ToolContext,
  ): Promise<string> | AsyncGenerator<ProgressEvent>
  
  isEnabled?(context: ToolContext): boolean
  requiresPermission?: boolean  // Ask before executing
  category?: 'read' | 'write' | 'execute' | 'network'
}
```

---

## Step 5: Implement Domain Tools

### Tool Design Principles

1. **One action per tool** — Don't make tools that do multiple things
2. **Idempotent where possible** — Same input = same output
3. **Return structured text** — Claude reads the output and decides next steps
4. **Include context in output** — "Updated 3 records" not just "OK"
5. **Fail gracefully** — Return error messages, don't throw unhandled exceptions

### Example: Legal Domain Tools

```typescript
// tools/ContractAnalyzeTool/ContractAnalyzeTool.ts
const ContractAnalyzeTool: Tool = {
  name: 'AnalyzeContract',
  description: `Analyze a legal contract document and extract key terms, 
obligations, and risk factors. Provide the file path or URL of the contract.

When to use: When you need to understand what a contract says before 
advising on it.

Returns: Structured analysis with parties, key dates, obligations, 
risk areas, and unusual clauses.`,

  inputSchema: z.object({
    source: z.string().describe('File path or URL of the contract'),
    focus_areas: z.array(z.string()).optional().describe(
      'Specific areas to focus on: "payment_terms", "liability", "ip_rights", etc.'
    ),
  }),

  async handler(input, context) {
    const { source, focus_areas } = input as { source: string; focus_areas?: string[] }
    
    // Read contract
    const text = source.startsWith('http') 
      ? await fetchDocument(source)
      : await readFile(source)
    
    // Extract key elements (could use Claude API here for sub-analysis)
    const analysis = await analyzeContract(text, focus_areas)
    
    return JSON.stringify(analysis, null, 2)
  },
  
  requiresPermission: false,  // Read-only, safe to auto-allow
  category: 'read',
}
```

### Example: Healthcare Domain Tools

```typescript
const CreateClinicalNotesTool: Tool = {
  name: 'CreateClinicalNote',
  description: `Create a structured clinical note from dictation or unstructured text.

Formats the note in SOAP format (Subjective, Objective, Assessment, Plan).
Does NOT modify any patient records — returns the formatted note for review.

Input: free-form text from physician dictation
Output: structured SOAP note ready for EHR entry`,

  inputSchema: z.object({
    dictation: z.string().describe('Raw physician dictation text'),
    patient_id: z.string().optional().describe('Patient ID for reference (not stored)'),
    note_type: z.enum(['progress', 'discharge', 'procedure', 'consultation']).default('progress'),
  }),

  async handler(input, context) {
    const note = await formatClinicalNote(input.dictation, input.note_type)
    return `# Clinical Note (${input.note_type.toUpperCase()})

## Structured Output
${note.soap}

## Extracted Data
- ICD-10 codes: ${note.icd10Codes.join(', ')}
- Medications mentioned: ${note.medications.join(', ')}
- Follow-up required: ${note.followUpRequired ? 'Yes' : 'No'}

Note: Please review and verify all content before submitting to EHR.`
  },
  
  requiresPermission: true,   // Clinical notes need approval
  category: 'write',
}
```

---

## Step 6: Implement the Query Loop

```typescript
// src/loop.ts
import Anthropic from '@anthropic-ai/sdk'

const client = new Anthropic()

export async function* runAgentLoop(
  systemPrompt: string,
  messages: MessageParam[],
  tools: Tool[],
  context: ToolContext,
): AsyncGenerator<AgentEvent> {

  // Build API params
  const params = {
    model: context.config.model,
    system: systemPrompt,
    messages,
    tools: tools.map(t => ({
      name: t.name,
      description: t.description,
      input_schema: t.inputSchema,
    })),
    max_tokens: 8192,
  }

  // Stream API call
  const stream = client.messages.stream(params)
  let assistantMessage = { role: 'assistant', content: [] }

  for await (const event of stream) {
    if (event.type === 'content_block_delta' && event.delta.type === 'text_delta') {
      yield { type: 'text_delta', text: event.delta.text }
    }
    
    if (event.type === 'message_stop') {
      const finalMessage = await stream.finalMessage()
      assistantMessage = finalMessage
      
      yield { type: 'message', message: finalMessage }
      
      // Handle tool calls
      if (finalMessage.stop_reason === 'tool_use') {
        const toolResults = []
        
        for (const block of finalMessage.content) {
          if (block.type !== 'tool_use') continue
          
          const tool = tools.find(t => t.name === block.name)
          if (!tool) continue
          
          // Check permission
          const permitted = await checkPermission(tool, block.input, context)
          if (!permitted) {
            toolResults.push({
              type: 'tool_result',
              tool_use_id: block.id,
              content: 'Permission denied by user',
            })
            continue
          }
          
          yield { type: 'tool_start', toolName: block.name, input: block.input }
          
          // Execute tool
          try {
            const result = await executeTool(tool, block.input, context)
            toolResults.push({
              type: 'tool_result',
              tool_use_id: block.id,
              content: result,
            })
            yield { type: 'tool_complete', toolName: block.name, result }
          } catch (error) {
            toolResults.push({
              type: 'tool_result',
              tool_use_id: block.id,
              is_error: true,
              content: `Error: ${error.message}`,
            })
            yield { type: 'tool_error', toolName: block.name, error }
          }
        }
        
        // Continue with tool results
        messages = [
          ...messages,
          { role: 'assistant', content: finalMessage.content },
          { role: 'user', content: toolResults },
        ]
        
        // Recurse
        yield* runAgentLoop(systemPrompt, messages, tools, context)
      }
    }
  }
}
```

---

## Step 7: System Prompt Design

The system prompt is critical. Structure it like Claude Code's:

```typescript
// src/prompts/systemPrompt.ts

export function buildSystemPrompt(config: AppConfig, memory: string[]): string {
  return `
# Role
You are an AI assistant specialized in ${config.domain}.
${config.agentDescription}

# Capabilities
You have access to the following tools:
${getToolDescriptions(config.tools)}

# Behavioral Guidelines
${getDomainGuidelines(config.domain)}

# Important Constraints
${getConstraints(config)}

# Working Context
Current user: ${config.userId}
Organization: ${config.orgId}
Date: ${new Date().toISOString()}

# Domain Knowledge
${memory.join('\n\n')}
`
}

function getDomainGuidelines(domain: string): string {
  // Domain-specific guidelines
  const guidelines = {
    legal: `
- Always recommend consulting a licensed attorney for final decisions
- Flag any potentially privileged communications
- Note when contract terms are unusual or potentially problematic
- Do not provide legal advice, only legal information and analysis`,
    
    healthcare: `
- Always prioritize patient safety above efficiency
- Never make diagnostic decisions — support physician workflow only
- Flag any concerning clinical findings for immediate physician review
- Maintain HIPAA compliance — do not log patient identifiers`,
    
    finance: `
- All financial calculations must be verified by appropriate professionals
- Flag any regulatory compliance concerns
- Never make trading or investment recommendations without appropriate disclaimers
- Ensure audit trail for all data modifications`,
  }
  
  return guidelines[domain] || '- Follow professional best practices for this domain'
}
```

---

## Step 8: Permission System (Simplified)

```typescript
// src/permissions/canUseTool.ts

type PermissionResult = 'allow' | 'deny' | 'ask'

export async function canUseTool(
  tool: Tool,
  input: Record<string, unknown>,
  config: AppConfig,
): Promise<PermissionResult> {
  
  // 1. Check explicit deny list
  for (const pattern of config.permissions.deny) {
    if (matchesPattern(pattern, tool.name, input)) {
      return 'deny'
    }
  }
  
  // 2. Check explicit allow list
  for (const pattern of config.permissions.allow) {
    if (matchesPattern(pattern, tool.name, input)) {
      return 'allow'
    }
  }
  
  // 3. Category-based defaults
  if (tool.category === 'read') return 'allow'        // Read-only: auto-allow
  if (tool.category === 'network') return 'ask'       // Network: ask
  if (tool.category === 'execute') return 'ask'       // Execution: ask
  if (tool.category === 'write') {
    return tool.requiresPermission ? 'ask' : 'allow'
  }
  
  return 'ask'  // Default: ask
}
```

---

## Step 9: Memory System

```typescript
// src/memory/loadMemory.ts

export async function loadDomainMemory(options: {
  userId: string
  orgId: string
  projectId?: string
}): Promise<string[]> {
  const memories: string[] = []
  
  // 1. Global domain knowledge
  const globalKnowledge = await loadFile('.domain-agent/knowledge/global.md')
  if (globalKnowledge) memories.push(globalKnowledge)
  
  // 2. Organization-specific context
  const orgContext = await loadFromDB(
    `SELECT content FROM org_memory WHERE org_id = ?`,
    [options.orgId]
  )
  if (orgContext) memories.push(orgContext)
  
  // 3. User preferences
  const userPrefs = await loadFromDB(
    `SELECT content FROM user_memory WHERE user_id = ?`,
    [options.userId]
  )
  if (userPrefs) memories.push(userPrefs)
  
  // 4. Project context
  if (options.projectId) {
    const projectContext = await loadFile(
      `.domain-agent/projects/${options.projectId}/CONTEXT.md`
    )
    if (projectContext) memories.push(projectContext)
  }
  
  return memories
}
```

---

## Step 10: Domain-Specific Customization Examples

### Healthcare Agent

**Tools to build:**
- `ReadPatientRecord` — Query EHR (read-only)
- `CreateClinicalNote` — Format clinical documentation
- `CheckDrugInteractions` — Verify medication safety
- `LookupICD10` — Find diagnostic codes
- `GenerateReferral` — Create referral letters
- `SummarizeVisit` — Summarize encounter notes

**System prompt focus:**
- Patient safety first
- HIPAA compliance instructions
- Always defer to physician
- Format preferences (SOAP, SBAR)

**Permission model:**
- Read patient data: ask with patient ID visibility
- Create notes: ask with preview
- Send/transmit anything: always ask + require double-confirmation

---

### Legal Agent

**Tools to build:**
- `AnalyzeContract` — Extract terms and risks
- `SearchCaseLaw` — Query legal databases
- `CompareContracts` — Diff two contract versions
- `GenerateClause` — Draft contract language
- `CheckCompliance` — Verify regulatory compliance
- `ExtractDates` — Pull key dates and deadlines

**System prompt focus:**
- Jurisdictional awareness
- Attorney-client privilege warnings
- "Information not advice" disclaimers
- Confidentiality reminders

---

### DevOps Agent

**Tools to build:**
- `QueryLogs` — Search application logs
- `GetMetrics` — Pull from monitoring systems
- `RunTerraformPlan` — Preview infrastructure changes
- `DeployService` — Trigger deployment (with approvals)
- `ScaleService` — Adjust resource allocation
- `RollbackDeployment` — Revert to previous version

**Permission model:**
- Read logs/metrics: auto-allow
- Terraform plan: auto-allow (read-only)
- Deploy to staging: ask
- Deploy to production: require explicit "deploy to production" confirmation
- Rollback: ask + explain impact

---

## Step 11: Deployment Architecture

### Single-User CLI Tool

```
npm install -g @company/domain-agent
domain-agent chat
```

- Config in `~/.domain-agent/settings.json`
- API key in environment

### Multi-Tenant Web Service

```
┌─────────────────┐     ┌─────────────────┐     ┌──────────────┐
│   Web Frontend  │────►│  Agent Service  │────►│ Anthropic API │
│   (Next.js)     │     │   (Node.js)     │     └──────────────┘
└─────────────────┘     │  - Session mgmt │     ┌──────────────┐
                         │  - Tool execute │────►│  Your DB     │
                         │  - Auth/authz   │     └──────────────┘
                         └─────────────────┘     ┌──────────────┐
                                              ────►  Ext. APIs   │
                                                   └──────────────┘
```

**Key additions for web:**
- Session storage in database (not filesystem)
- User auth (JWT or session cookies)
- Rate limiting per user
- Tool execution in isolated containers (security)
- Audit logging to database

---

## Step 12: Testing Strategy

### Tool Unit Tests

```typescript
describe('ContractAnalyzeTool', () => {
  it('extracts payment terms', async () => {
    const result = await ContractAnalyzeTool.handler(
      { source: './fixtures/sample-contract.pdf' },
      mockContext
    )
    expect(result).toContain('payment_terms')
    expect(JSON.parse(result).payment_terms).toBeDefined()
  })
  
  it('handles missing files gracefully', async () => {
    const result = await ContractAnalyzeTool.handler(
      { source: './nonexistent.pdf' },
      mockContext
    )
    expect(result).toContain('Error:')
  })
})
```

### Integration Tests (Full Loop)

```typescript
it('analyzes a contract end-to-end', async () => {
  const session = new Session({
    model: 'claude-haiku-4-5-20251001',  // Cheap for testing
    tools: [ContractAnalyzeTool],
    permissions: { mode: 'bypass' },      // Skip prompts in tests
  })
  
  const events = []
  for await (const event of session.submitMessage(
    'Analyze the contract in fixtures/sample.pdf and tell me the payment terms'
  )) {
    events.push(event)
  }
  
  const finalMessage = events.find(e => e.type === 'message')
  expect(finalMessage.text).toContain('payment')
  expect(finalMessage.text).not.toContain('Error')
})
```

---

## Checklist: Production Readiness

**Core functionality**
- [ ] All domain tools implemented and tested
- [ ] System prompt reviewed with domain experts
- [ ] Permission model defined for every tool
- [ ] Error handling for all external API calls

**Security**
- [ ] API key management (never hardcoded)
- [ ] Input validation on all tool inputs
- [ ] Output sanitization (no PII leakage in logs)
- [ ] Permission model reviewed by security team
- [ ] Audit logging for all tool executions

**Reliability**
- [ ] Retry logic for API calls
- [ ] Context window management (compaction)
- [ ] Rate limit handling
- [ ] Graceful degradation when tools fail

**Compliance** (domain-specific)
- [ ] Legal: attorney review of disclaimers
- [ ] Healthcare: HIPAA compliance review, BAA with Anthropic
- [ ] Finance: compliance review, audit trail
- [ ] All: data retention and deletion policies

**UX**
- [ ] Permission dialogs are clear and actionable
- [ ] Progress indicators for long operations
- [ ] Session history and resume capability
- [ ] Help documentation
