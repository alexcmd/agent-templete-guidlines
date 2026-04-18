# UI Layer

## Overview

Claude Code's UI is built with [Ink](https://github.com/vadimdemedes/ink), a React renderer for terminal applications. The main REPL component is ~5,000 lines and manages the entire interactive experience.

---

## Technology Stack

| Library | Purpose |
|---------|---------|
| React 18 | Component model + hooks |
| Ink 5 | Terminal renderer (translates React → ANSI escape codes) |
| `react-compiler` | Automatic memoization |
| `chalk` | Colors and text styling |
| `cli-truncate` | Ellipsis truncation for terminal widths |
| `strip-ansi` | Strip ANSI codes for plain text |
| `ansi-escapes` | ANSI control sequences |

---

## Component Hierarchy

```
App (Ink root)
└── REPL.tsx (5000 lines — main conversation UI)
    ├── StatusLine — top bar with model, session info
    ├── MessageList — scrollable conversation history
    │   ├── UserMessage — renders user input
    │   │   ├── Text block
    │   │   ├── Image block (if pasted image)
    │   │   └── Attachment (if file attached)
    │   ├── AssistantMessage — renders model output
    │   │   ├── TextBlock — markdown-rendered text
    │   │   ├── ThinkingBlock — collapsible extended thinking
    │   │   └── ToolUseBlock → per-tool renderers
    │   ├── ToolResultMessage (collapsed tool output)
    │   │   ├── BashTool UI — command + output
    │   │   ├── FileEdit UI — diff view
    │   │   ├── AgentTool UI — sub-agent status
    │   │   ├── MCP UI — MCP tool output
    │   │   └── ProgressMessage — streaming progress
    │   └── SystemMessage — system notifications
    ├── PermissionDialog — modal overlays for permissions
    │   ├── BashPermissionRequest
    │   ├── FileEditPermissionRequest
    │   ├── FileWritePermissionRequest
    │   ├── WebFetchPermissionRequest
    │   ├── AgentPermissionRequest
    │   └── MCPPermissionRequest
    ├── InputArea — text input with vim/readline support
    │   ├── TextInput (Ink-based multiline input)
    │   ├── AutoComplete (command completion)
    │   └── ImagePaste (clipboard image handling)
    └── Footer — status bar with cost, tokens, shortcuts
        ├── CostDisplay
        ├── ModelDisplay
        ├── TasksFooter (active tasks count)
        └── KeybindingHints
```

---

## REPL State Management

The REPL maintains extensive local state alongside the global `AppState`:

```typescript
// REPL.tsx local state (simplified)
type REPLState = {
  // Input
  inputValue: string
  isSubmitting: boolean
  
  // Conversation
  messages: Message[]
  isStreaming: boolean
  
  // UI
  showSystemMessages: boolean
  selectedMessageIndex: number | null
  expandedToolUseIds: Set<string>
  
  // Permission dialogs
  pendingPermissionRequest: PermissionRequest | null
  
  // History navigation
  historyIndex: number
  
  // Modals
  activeModal: 'none' | 'help' | 'history' | 'config' | ...
}
```

---

## Message Rendering

Each message type has a dedicated renderer:

### AssistantMessage Renderer
```tsx
function AssistantMessageRenderer({ message }: { message: AssistantMessage }) {
  const contentBlocks = message.message.content
  
  return (
    <Box flexDirection="column">
      {contentBlocks.map((block, i) => {
        if (block.type === 'text') {
          return <MarkdownText key={i} text={block.text} />
        }
        if (block.type === 'tool_use') {
          return <ToolUseRenderer key={i} toolUse={block} />
        }
        if (block.type === 'thinking') {
          return <ThinkingBlock key={i} thinking={block} />
        }
      })}
      <MessageFooter usage={message.message.usage} cost={message.costUSD} />
    </Box>
  )
}
```

### Tool-Specific Renderers

Each tool (`BashTool`, `FileEditTool`, etc.) exports its own UI component:

```tsx
// tools/BashTool/UI.tsx
export function renderToolUseMessage(toolUse: ToolUseBlock): React.ReactNode {
  const command = toolUse.input.command as string
  return (
    <Box>
      <Text color="cyan">$ </Text>
      <Text>{command}</Text>
    </Box>
  )
}

export function renderToolResultMessage(result: string, isError: boolean): React.ReactNode {
  const lines = result.split('\n').slice(0, MAX_DISPLAY_LINES)
  return (
    <Box flexDirection="column" marginLeft={2}>
      {isError && <Text color="red">Error: </Text>}
      {lines.map((line, i) => <Text key={i}>{line}</Text>)}
      {result.split('\n').length > MAX_DISPLAY_LINES && (
        <Text color="gray">... (truncated)</Text>
      )}
    </Box>
  )
}
```

---

## Text Input System

The input area supports:

### Keybindings (`src/keybindings/`)

```typescript
type KeyBinding = {
  key: string               // 'ctrl+enter', 'escape', 'tab', etc.
  description: string       // Shown in help overlay
  handler: () => void       // Action to perform
}

// Default bindings
const defaultBindings: KeyBinding[] = [
  { key: 'enter',       description: 'Submit message' },
  { key: 'ctrl+enter',  description: 'Insert newline' },
  { key: 'escape',      description: 'Cancel / clear input' },
  { key: 'ctrl+c',      description: 'Interrupt Claude' },
  { key: 'ctrl+l',      description: 'Clear screen' },
  { key: 'ctrl+r',      description: 'Search history' },
  { key: 'up/down',     description: 'Navigate history' },
  { key: 'tab',         description: 'Autocomplete command' },
]
```

### Vim Mode (`src/vim/`)

Full vim modal editing for the input field:
- Normal mode (`Escape` → normal)
- Insert mode (default)
- Visual mode (text selection)
- Supports: `w`, `b`, `0`, `$`, `dd`, `cc`, `yy`, `p`, `/search`, etc.

### Auto-complete

```typescript
// Command auto-complete
function getAutoCompleteOptions(input: string, commands: Command[]): string[] {
  if (!input.startsWith('/')) return []
  
  const partial = input.slice(1)
  return commands
    .filter(c => c.name.startsWith(partial) && !c.isHidden)
    .map(c => `/${c.name}`)
}
```

---

## Markdown Rendering

Claude's text responses are rendered as markdown in the terminal:

```typescript
// MarkdownText component
function MarkdownText({ text }: { text: string }) {
  // Renders:
  // - **bold** → bold text
  // - `code` → highlighted code spans
  // - ```code blocks``` → syntax-highlighted, bordered boxes
  // - # Headers → colored, larger text
  // - - Lists → bullet points with indentation
  // - > Blockquotes → left-border, dimmed
  // - [links](url) → colored URL text
  // - Tables → ASCII table rendering
}
```

---

## Status Line (`src/screens/`)

Top-bar showing:
- Current model name
- Session cost (running total)
- Token count
- Active tasks indicator
- Streaming indicator (spinner)
- Worktree mode indicator
- Plan mode indicator

---

## Spinner and Progress Indicators

```tsx
// Spinner component (src/components/Spinner.tsx)
type SpinnerMode = 
  | 'dots'      // ⠋ ⠙ ⠹ ⠸ ⠼ ⠴ ⠦ ⠧ ⠇ ⠏
  | 'line'      // - \ | /
  | 'arrow'     // → ↗ ↑ ↖ ← ↙ ↓ ↘

function Spinner({ mode, label }: { mode: SpinnerMode, label?: string }) {
  const [frame, setFrame] = useState(0)
  
  useEffect(() => {
    const timer = setInterval(() => {
      setFrame(f => (f + 1) % FRAMES[mode].length)
    }, 80)
    return () => clearInterval(timer)
  }, [mode])
  
  return <Text color="cyan">{FRAMES[mode][frame]} {label}</Text>
}
```

---

## Permission Dialog UI

Shown as a modal overlay when a tool needs permission:

```tsx
function PermissionDialog({ request, onDecision }) {
  return (
    <Box
      borderStyle="round"
      borderColor="yellow"
      flexDirection="column"
      padding={1}
    >
      <Text bold>Claude wants to run a Bash command:</Text>
      <Text> </Text>
      <Box marginLeft={2}>
        <Text color="cyan">$ </Text>
        <Text>{request.command}</Text>
      </Box>
      <Text> </Text>
      <Box gap={2}>
        <KeyOption key="y" label="Allow" onPress={() => onDecision('allow')} />
        <KeyOption key="a" label="Always Allow" onPress={() => onDecision('always-allow')} />
        <KeyOption key="n" label="Deny" onPress={() => onDecision('deny')} />
        <KeyOption key="d" label="Always Deny" onPress={() => onDecision('always-deny')} />
      </Box>
    </Box>
  )
}
```

---

## Task/Agent View

When multiple agents are running, the REPL shows an expandable task view:

```
[2 tasks running] ▶ Press T to expand
```

Expanded:
```
Tasks:
  ✓ [explore-frontend] Completed — Found 47 components
  ⠸ [explore-backend] Running — Analyzing services...
  ✗ [explore-utils] Failed — Permission denied
```

---

## Theming (`src/utils/theme.ts`)

```typescript
type ThemeName = 'dark' | 'light' | 'high-contrast' | 'solarized-dark' | 'solarized-light'

type Theme = {
  // Text colors
  text: string
  textDim: string
  textBright: string
  
  // UI elements
  border: string
  header: string
  
  // Semantic colors
  success: string    // green
  warning: string    // yellow
  error: string      // red
  info: string       // blue
  
  // Tool-specific
  bashCommand: string
  filePath: string
  agentName: string
  costText: string
}
```

---

## Voice Input (`src/voice/`)

Feature-gated voice input support:
- `voiceStreamSTT.ts` — Streaming speech-to-text
- `voice.ts` — Voice activation logic
- `voiceKeyterms.ts` — Keyword detection for activation

---

## Non-Interactive / Bare Mode

When `isBareMode()` is true (piped input or `--no-tty`):
- No React/Ink rendering
- Direct stdout output
- JSON output mode available (`--output json`)
- Streaming chunks written to stdout as they arrive
