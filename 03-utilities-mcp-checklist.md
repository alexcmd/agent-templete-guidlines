# Reference Implementations — Utilities, MCP & Production Checklist

> Sections: CWD Hashing · Token Estimation · Loop Detection · Hook Runner · Cost Controller · Concurrent Execution · Tool Output Masking · REPL · MCP Integration · Production Checklist

Part of the [Reference Implementations](03-minimal-agents.md) series.

## 5. CWD Hashing for Session Storage

Session files are stored under `~/.agent/projects/<hash>/sessions/<id>.jsonl` where `<hash>` is a 16-character hex string derived from the real absolute path of the working directory. This means reopening the same directory always finds its previous sessions, regardless of symlinks.

### Python

```python
import hashlib, os
from pathlib import Path

def session_dir(cwd: str) -> Path:
    cwd_hash = hashlib.sha256(os.path.realpath(cwd).encode()).hexdigest()[:16]
    d = Path.home() / ".agent" / "projects" / cwd_hash / "sessions"
    d.mkdir(parents=True, exist_ok=True)
    return d
```

### TypeScript

```typescript
import * as crypto from 'crypto';
import * as fs from 'fs';
import * as path from 'path';

function sessionDir(cwd: string): string {
    const cwdHash = crypto
        .createHash('sha256')
        .update(fs.realpathSync(cwd))
        .digest('hex')
        .slice(0, 16);
    const dir = path.join(process.env.HOME!, '.agent', 'projects', cwdHash, 'sessions');
    fs.mkdirSync(dir, { recursive: true });
    return dir;
}
```

### Rust

```rust
use sha2::{Sha256, Digest};
use std::path::PathBuf;

fn session_dir(cwd: &str) -> PathBuf {
    let real = std::fs::canonicalize(cwd).unwrap();
    let hash = sha2::Sha256::digest(real.to_string_lossy().as_bytes());
    let hex: String = hash[..8].iter().map(|b| format!("{:02x}", b)).collect();
    let dir = dirs::home_dir().unwrap()
        .join(".agent").join("projects").join(hex).join("sessions");
    std::fs::create_dir_all(&dir).unwrap();
    dir
}
```

### Go

```go
import (
    "crypto/sha256"
    "fmt"
    "os"
    "path/filepath"
)

func sessionDir(cwd string) string {
    abs, _ := filepath.EvalSymlinks(cwd)
    hash := sha256.Sum256([]byte(abs))
    hex := fmt.Sprintf("%x", hash[:8])  // 16 hex chars
    dir := filepath.Join(os.Getenv("HOME"), ".agent", "projects", hex, "sessions")
    os.MkdirAll(dir, 0755)
    return dir
}
```

---

## 6. Token Estimation

### Rule of Thumb

```
~4 characters per token for English text
~3 characters per token for code (keywords, symbols are short)
~6 characters per token for languages with long words (German)
```

A quick estimate: `len(text) // 4`

When to estimate vs. count:
- **Estimate**: system prompt sizing, rough budget checks, quick UI feedback
- **count_tokens API**: before sending a request that might exceed limits, during compaction decisions, when billing accuracy matters

### Python — Quick Estimate

```python
def estimate_tokens(text: str) -> int:
    """Rough estimate: 4 chars per token (English prose/code)."""
    return len(text) // 4

def estimate_messages_tokens(messages: list[dict]) -> int:
    """Estimate total tokens in a messages array."""
    total = 0
    for msg in messages:
        content = msg.get("content", "")
        if isinstance(content, str):
            total += estimate_tokens(content)
        elif isinstance(content, list):
            for block in content:
                if isinstance(block, dict):
                    total += estimate_tokens(block.get("text", "") or
                                             block.get("content", "") or
                                             str(block.get("input", "")))
    return total + len(messages) * 4  # per-message overhead
```

### Python — Exact Count via count_tokens API

```python
import anthropic

client = anthropic.Anthropic()

def count_tokens_exact(messages: list[dict], system: str = None, model: str = "claude-opus-4-6") -> int:
    """Call count_tokens API for an exact count. No inference is performed."""
    kwargs = {
        "model": model,
        "messages": messages,
    }
    if system:
        kwargs["system"] = system
    
    response = client.messages.count_tokens(**kwargs)
    return response.input_tokens

# Decision: when to use count_tokens
CONTEXT_LIMIT = 200_000  # tokens
WARNING_THRESHOLD = 0.80  # 80% full → warn
COMPACTION_THRESHOLD = 0.90  # 90% full → compact

def should_compact(messages: list[dict]) -> bool:
    estimated = estimate_messages_tokens(messages)
    if estimated / CONTEXT_LIMIT < WARNING_THRESHOLD:
        return False  # Well under limit, skip API call
    exact = count_tokens_exact(messages)
    return exact / CONTEXT_LIMIT >= COMPACTION_THRESHOLD
```

---

## 7. Loop Detection

### Concept

An agent can loop when a tool call always returns the same error and the model keeps retrying the same approach. Two detection strategies:

1. **Heuristic**: if the last N turns all contain identical tool call fingerprints (same tool name + same input), assume a loop.
2. **LLM-based**: after turn 30, ask a cheap model (Haiku) "Is this agent stuck in a loop?" with a confidence threshold.

A practical tuning: heuristic threshold=5 same-tool turns, LLM check every 5–15 turns after turn 30.

### Python — LoopDetector

```python
import hashlib
import json

class LoopDetector:
    def __init__(self, heuristic_threshold: int = 5, llm_check_after_turn: int = 30,
                 llm_check_interval: int = 10):
        self.heuristic_threshold  = heuristic_threshold
        self.llm_check_after_turn = llm_check_after_turn
        self.llm_check_interval   = llm_check_interval
        self.fingerprints: list[str] = []
    
    def _fingerprint(self, tool_uses: list[dict]) -> str:
        """Stable hash of (tool_name, sorted_input) tuples."""
        calls = sorted([
            (tu.get("name", ""), json.dumps(tu.get("input", {}), sort_keys=True))
            for tu in tool_uses
        ])
        return hashlib.md5(json.dumps(calls).encode()).hexdigest()
    
    def record_turn(self, tool_uses: list[dict]) -> None:
        if tool_uses:
            self.fingerprints.append(self._fingerprint(tool_uses))
        else:
            self.fingerprints.append("__no_tools__")
    
    def is_heuristic_loop(self) -> bool:
        n = self.heuristic_threshold
        if len(self.fingerprints) < n:
            return False
        recent = self.fingerprints[-n:]
        return len(set(recent)) == 1 and recent[0] != "__no_tools__"
    
    def should_do_llm_check(self, turn_count: int) -> bool:
        if turn_count < self.llm_check_after_turn:
            return False
        return (turn_count - self.llm_check_after_turn) % self.llm_check_interval == 0
    
    def llm_loop_check(self, messages: list[dict], client) -> bool:
        """Ask claude-haiku if the conversation looks stuck. Returns True if looping."""
        summary = []
        for msg in messages[-20:]:
            role = msg["role"]
            content = msg.get("content", "")
            if isinstance(content, list):
                for block in content:
                    if isinstance(block, dict) and block.get("type") == "tool_use":
                        summary.append(f"{role}: called {block['name']}({str(block.get('input',''))[:80]})")
                    elif isinstance(block, dict) and block.get("type") == "text":
                        summary.append(f"{role}: {block['text'][:120]}")
            else:
                summary.append(f"{role}: {str(content)[:120]}")
        
        probe = (
            "Here is the recent conversation transcript:\n" +
            "\n".join(summary) +
            "\n\nIs the agent stuck in a loop, repeating the same tool calls "
            "without making progress? Reply ONLY with: LOOPING or NOT_LOOPING"
        )
        response = client.messages.create(
            model="claude-haiku-4-5-20251001",
            max_tokens=10,
            messages=[{"role": "user", "content": probe}]
        )
        answer = response.content[0].text.strip()
        return "LOOPING" in answer


# Usage in agent loop
def run_agent_with_loop_detection(prompt, session, config, cwd):
    import anthropic
    client = anthropic.Anthropic()
    loop_detector = LoopDetector()
    
    for turn_count in range(config.max_turns):
        # ... normal agent loop ...
        # After executing tools:
        tool_uses = []  # populated from response
        loop_detector.record_turn(tool_uses)
        
        if loop_detector.is_heuristic_loop():
            return "[Agent stopped: heuristic loop detected]"
        
        if loop_detector.should_do_llm_check(turn_count):
            if loop_detector.llm_loop_check(session.messages, client):
                return "[Agent stopped: LLM confirmed loop]"
```

---

## 8. Hook Runner

Hooks are external processes that receive a JSON event on stdin and return a JSON response on stdout. They can allow, deny, or modify tool calls.

### Python — run_hook

```python
import subprocess
import json
from typing import Literal

def run_hook(hook_cmd: str, event_type: str, payload: dict,
             timeout: int = 10) -> dict:
    """
    Run a hook subprocess. Returns the hook's response dict.
    If the hook fails or times out, returns {"allowed": True} (fail-open).
    """
    event = {"event": event_type, **payload}
    try:
        result = subprocess.run(
            hook_cmd,
            shell=True,
            input=json.dumps(event).encode(),
            capture_output=True,
            timeout=timeout
        )
        if result.returncode != 0:
            # Hook signalled a block
            return {"allowed": False, "reason": result.stderr.decode()[:200]}
        if result.stdout:
            return json.loads(result.stdout)
    except subprocess.TimeoutExpired:
        pass  # fail-open on timeout
    except Exception:
        pass
    return {"allowed": True}
```

### Hook Event Schemas

Each hook type receives a specific JSON payload on stdin:

**PreToolUse** — fired before every tool execution
```json
{
  "event": "PreToolUse",
  "tool": "bash",
  "input": "{\"command\": \"rm -rf /tmp/test\"}",
  "inputJson": {"command": "rm -rf /tmp/test"}
}
```
Expected stdout: `{"allowed": true}` or `{"allowed": false, "reason": "..."}` or `{"allowed": true, "updatedInput": {...}}`

**PostToolUse** — fired after tool execution
```json
{
  "event": "PostToolUse",
  "tool": "bash",
  "input": {"command": "ls"},
  "output": "file1.py\nfile2.py",
  "is_error": false
}
```
Expected stdout: `{"continue": true}` or `{"continue": false, "reason": "..."}` or `{"updatedOutput": "..."}`

**Stop** — fired when the agent loop ends normally
```json
{
  "event": "Stop",
  "session_id": "abc123",
  "turn_count": 12,
  "total_cost_usd": 0.042,
  "stop_reason": "end_turn"
}
```
Expected stdout: `{}` (output ignored)

**UserPromptSubmit** — fired after user types but before sending to API
```json
{
  "event": "UserPromptSubmit",
  "prompt": "Delete all test files",
  "session_id": "abc123"
}
```
Expected stdout: `{"allowed": true}` or `{"allowed": false, "reason": "..."}` or `{"updatedPrompt": "..."}`

**SessionStart** — fired once when the agent session is created
```json
{
  "event": "SessionStart",
  "session_id": "abc123",
  "cwd": "/home/user/project",
  "model": "claude-opus-4-6"
}
```

**SessionStop** — fired once when the session ends (Ctrl+C, /exit, or normal completion)
```json
{
  "event": "SessionStop",
  "session_id": "abc123",
  "turn_count": 12,
  "total_cost_usd": 0.042
}
```

### Example Hook: Audit Logger

```bash
#!/bin/bash
# ~/.agent/hooks/audit.sh
# Log every tool call to audit.log

read -r EVENT
TOOL=$(echo "$EVENT" | python3 -c "import sys,json; print(json.load(sys.stdin).get('tool',''))")
CMD=$(echo "$EVENT" | python3 -c "import sys,json; d=json.load(sys.stdin); print(d.get('inputJson',{}).get('command','')[:200])")

echo "$(date -Iseconds) TOOL=$TOOL CMD=$CMD" >> ~/.agent/audit.log
echo '{"allowed": true}'
```

---

## 9. Cost Controller

### Python — CostController

```python
MODEL_RATES = {
    "claude-opus-4-6": {
        "input":       15.00,  # per million tokens
        "output":      75.00,
        "cache_read":   1.50,
        "cache_write": 18.75,
    },
    "claude-sonnet-4-6": {
        "input":       3.00,
        "output":      15.00,
        "cache_read":   0.30,
        "cache_write":  3.75,
    },
    "claude-haiku-4-5-20251001": {
        "input":       0.80,
        "output":       4.00,
        "cache_read":   0.08,
        "cache_write":  1.00,
    },
}

class CostController:
    def __init__(self, budget_usd: float = 5.0, warn_at: float = 0.8):
        self.budget_usd   = budget_usd
        self.warn_at      = warn_at  # fraction of budget at which to warn
        self.total_cost   = 0.0
        self.input_tokens = 0
        self.output_tokens = 0
        self.turns        = 0

    def record(self, usage, model: str) -> float:
        rates = MODEL_RATES.get(model, MODEL_RATES["claude-opus-4-6"])
        turn_cost = (
            usage.input_tokens  * rates["input"]       / 1_000_000 +
            usage.output_tokens * rates["output"]      / 1_000_000 +
            getattr(usage, "cache_read_input_tokens",     0) * rates["cache_read"]  / 1_000_000 +
            getattr(usage, "cache_creation_input_tokens", 0) * rates["cache_write"] / 1_000_000
        )
        self.total_cost   += turn_cost
        self.input_tokens += usage.input_tokens
        self.output_tokens += usage.output_tokens
        self.turns        += 1
        return turn_cost

    @property
    def budget_fraction(self) -> float:
        return self.total_cost / self.budget_usd if self.budget_usd else 0.0

    def is_over_budget(self) -> bool:
        return self.total_cost >= self.budget_usd

    def should_warn(self) -> bool:
        return self.budget_fraction >= self.warn_at

    def status_line(self) -> str:
        return (f"${self.total_cost:.4f} / ${self.budget_usd:.2f} "
                f"({self.budget_fraction*100:.0f}%) "
                f"— {self.input_tokens} in / {self.output_tokens} out")
```

---

## 10. Concurrent Tool Execution

Run up to 10 tool calls in parallel when they are independent (e.g., reading multiple files). Only safe when tools are read-only or independently scoped (writing to different files).

### Python — asyncio + semaphore

```python
import asyncio
import subprocess
from typing import Any

async def execute_tool_async(name: str, inp: dict, semaphore: asyncio.Semaphore) -> tuple[str, bool]:
    """Async wrapper around tool execution, bounded by semaphore."""
    async with semaphore:
        # Run blocking I/O in thread pool
        loop = asyncio.get_event_loop()
        return await loop.run_in_executor(None, execute_tool_sync, name, inp)

def execute_tool_sync(name: str, inp: dict) -> tuple[str, bool]:
    """Synchronous tool executor (same logic as execute_tool above)."""
    try:
        match name:
            case "bash":
                r = subprocess.run(inp["command"], shell=True,
                                   capture_output=True, text=True, timeout=30)
                return (r.stdout + r.stderr)[:10000], r.returncode != 0
            case "read_file":
                return open(inp["path"]).read(), False
            case _:
                return f"Unknown tool: {name}", True
    except Exception as e:
        return str(e), True

async def execute_tools_parallel(tool_uses: list[dict], max_concurrency: int = 10):
    """Execute up to max_concurrency tools in parallel."""
    semaphore = asyncio.Semaphore(max_concurrency)
    tasks = [
        execute_tool_async(tu["name"], tu["input"], semaphore)
        for tu in tool_uses
    ]
    results = await asyncio.gather(*tasks)
    return [
        {
            "type": "tool_result",
            "tool_use_id": tu["id"],
            "content": output,
            **({"is_error": True} if is_error else {})
        }
        for tu, (output, is_error) in zip(tool_uses, results)
    ]

# Usage
tool_results = asyncio.run(execute_tools_parallel(tool_uses))
```

### TypeScript — pMap (concurrency: 10)

```typescript
import pMap from 'p-map';  // npm install p-map

async function executeToolsParallel(
    toolUses: Anthropic.ToolUseBlock[]
): Promise<Anthropic.ToolResultBlockParam[]> {
    return pMap(
        toolUses,
        async (tu) => {
            const [output, isError] = await executeToolAsync(tu.name, tu.input as any);
            return {
                type: 'tool_result' as const,
                tool_use_id: tu.id,
                content: output,
                ...(isError ? { is_error: true } : {}),
            };
        },
        { concurrency: 10 }
    );
}
```

### Rust — tokio::task::JoinSet

```rust
use tokio::task::JoinSet;

async fn execute_tools_parallel(tool_uses: Vec<ToolUse>) -> Vec<ToolResult> {
    let mut set: JoinSet<ToolResult> = JoinSet::new();
    
    for tu in tool_uses {
        set.spawn(async move {
            let (output, is_error) = execute_tool(&tu.name, &tu.input).await;
            ToolResult {
                tool_use_id: tu.id,
                content: output,
                is_error,
            }
        });
    }
    
    let mut results = Vec::new();
    while let Some(res) = set.join_next().await {
        results.push(res.unwrap());
    }
    results
}
```

> **Safety**: only run tools in parallel when they are read-only (read_file, glob_search, grep_search) or when their write targets are guaranteed to be independent. Never run two `bash` commands in parallel if either might modify shared state.

---

## 11. Tool Output Masking

When tool outputs in conversation history grow large, the model's context fills up with verbose outputs it no longer needs in detail. The masking pattern replaces old tool outputs with compact placeholders in the context sent to the LLM, while keeping the actual output stored locally for display.

### Python — Tool Output Masking

```python
MASK_THRESHOLD_CHARS = 2000   # mask outputs larger than this
MASK_HISTORY_DEPTH   = 5      # keep the last N tool outputs unmasked

def mask_old_tool_outputs(messages: list[dict],
                           threshold: int = MASK_THRESHOLD_CHARS,
                           keep_recent: int = MASK_HISTORY_DEPTH) -> list[dict]:
    """
    Returns a copy of messages with old large tool outputs replaced by placeholders.
    The original list is NOT modified.
    """
    # Collect indices of tool_result blocks in chronological order
    result_indices: list[tuple[int, int]] = []  # (msg_idx, block_idx)
    for mi, msg in enumerate(messages):
        if msg["role"] != "user":
            continue
        content = msg.get("content", [])
        if not isinstance(content, list):
            continue
        for bi, block in enumerate(content):
            if isinstance(block, dict) and block.get("type") == "tool_result":
                result_indices.append((mi, bi))
    
    # Only mask outputs older than the most recent `keep_recent`
    to_mask = result_indices[:-keep_recent] if len(result_indices) > keep_recent else []
    mask_set = set(to_mask)
    
    import copy
    masked = copy.deepcopy(messages)
    for mi, bi in mask_set:
        block = masked[mi]["content"][bi]
        output = block.get("content", "")
        if isinstance(output, str) and len(output) > threshold:
            size = len(output)
            block["content"] = f"[Tool output masked: {size} chars — stored locally]"
    
    return masked

# Usage in agent loop:
# context_messages = mask_old_tool_outputs(session.messages)
# response = client.messages.create(..., messages=context_messages)
```

---

## 12. REPL Implementation

### Pseudocode

```
function startREPL(config, session):
    history = []
    
    while true:
        input = readLine(
            prompt = buildPrompt(config),
            completions = getCompletions(),  // slash commands, @files
            history = history
        )
        
        if input in ["/exit", "/quit", ""]:
            break
        
        // Handle slash commands
        if input.startsWith("/"):
            result = handleSlashCommand(input, config, session)
            displayResult(result)
            continue
        
        // Handle @file mentions
        if "@" in input:
            input = expandAtMentions(input)
        
        // Run agent turn
        history.append(input)
        
        for event in agentLoop(input, session.messages, config):
            match event.type:
                "text"        → displayText(event.text)
                "tool_start"  → displayToolStart(event.tool, event.input)
                "tool_result" → displayToolResult(event.tool, event.result)
                "thinking"    → displayThinking(event.thought)
                "error"       → displayError(event.error)
                "cost"        → updateCostDisplay(event.usage)
    
    saveSession(session)
```

### Essential Slash Commands

```
/help           → show available commands
/clear          → clear conversation (new session)
/resume [id]    → resume previous session
/sessions       → list recent sessions
/compact        → manually compact context
/model [name]   → switch model mid-session
/permissions    → show/edit permission rules
/mcp            → list MCP servers and tools
/cost           → show token usage and cost so far
/exit           → quit

/[skill-name]   → invoke a skill (e.g., /review, /test)
```

### Python REPL

```python
def repl():
    cwd = os.getcwd()
    config = Config.load(cwd)
    
    history_file = Path.home() / ".agent" / "history"
    history_file.parent.mkdir(exist_ok=True)
    try:
        readline.read_history_file(history_file)
    except FileNotFoundError:
        pass
    readline.set_history_length(1000)
    
    session = Session(cwd)
    print(f"Agent ready. Session: {session.session_id[:8]}... (Ctrl+D to exit)")
    
    while True:
        try:
            user_input = input("\n> ").strip()
        except (EOFError, KeyboardInterrupt):
            print("\nGoodbye!")
            readline.write_history_file(history_file)
            break
        
        if not user_input:
            continue
        
        if user_input == "/sessions":
            for sid in Session.list_sessions(cwd):
                print(f"  {sid}")
            continue
        elif user_input.startswith("/resume "):
            sid = user_input.split(" ", 1)[1].strip()
            session = Session(cwd, sid)
            print(f"Resumed session {sid[:8]}...")
            continue
        elif user_input == "/clear":
            session = Session(cwd)
            print(f"New session: {session.session_id[:8]}...")
            continue
        elif user_input == "/help":
            print("Commands: /sessions, /resume <id>, /clear, /help, /cost, /exit")
            continue
        
        print()
        run_agent(
            user_input, session, config, cwd,
            on_text=lambda t: print(t, end="", flush=True),
            on_tool=lambda n, i: print(f"\n  [{n}: {str(i)[:60]}]", flush=True),
            on_cost=lambda c, u: None
        )
        print()

if __name__ == "__main__":
    import sys
    if len(sys.argv) > 1:
        cwd = os.getcwd()
        config = Config.load(cwd)
        session = Session(cwd)
        run_agent(
            " ".join(sys.argv[1:]), session, config, cwd,
            on_text=lambda t: print(t, end="", flush=True),
            on_tool=lambda n, i: print(f"\n  [{n}: {str(i)[:60]}]",
                                       file=__import__("sys").stderr)
        )
        print()
    else:
        repl()
```

---

## 13. Production Checklist

A comprehensive checklist of everything required before shipping an agent to production.

### Security

- [ ] API key never hardcoded (env var or OS keychain — see `02-authentication.md § 6`)
- [ ] Rate limit handling (429 → read `retry-after` header, sleep, retry)
- [ ] Permission deny rules configured for destructive operations
- [ ] Pre-tool hook for audit logging (every bash command logged)
- [ ] Tool schema validation before execution (reject extra keys)
- [ ] Sub-agent worktree isolation (each sub-agent gets its own temp git worktree)
- [ ] Path traversal prevention (`os.path.realpath()` before file access)

### Reliability

- [ ] Stream stall recovery (watchdog timer: if no SSE event in 30s, abort and retry)
- [ ] Fallback model on 529 overloaded (e.g., Opus → Sonnet, with notice to user)
- [ ] `max_tokens` recovery (when `stop_reason == "max_tokens"`, inject "continue" message and retry)
- [ ] `ToolUse`/`ToolResult` boundary guard in compaction (never compact between a tool_use and its tool_result)
- [ ] Tool result budget (evict oldest tool_result content when total chars > cap)
- [ ] Session JSONL persistence after every message (not just at the end)
- [ ] Compaction circuit breaker (disable compaction after 3 consecutive failures)
- [ ] Loop detection after N turns (heuristic + LLM check — see `§ 7`)
- [ ] Max turns enforced (prevent runaway loops)
- [ ] Graceful `Ctrl+C` / `SIGINT` handling (cancel current API request, save session, exit cleanly)

### Context Management

- [ ] System prompt cache boundary marker (static → `cache_control: ephemeral` → dynamic)
- [ ] Memory file (AGENT.md / CLAUDE.md) loaded from CWD upward at session start
- [ ] Token warning indicators at 80% and 95% of context window
- [ ] Context compaction triggered before hitting the limit

### Cost & Observability

- [ ] Cost tracking per turn with budget cap (see `§ 9 CostController`)
- [ ] Per-session cost logged to JSONL
- [ ] Tool call success/failure rates tracked
- [ ] Session ID included in all log lines

### Tool System

- [ ] MCP tool naming uses `mcp__<server>__<tool>` prefix convention
- [ ] Tool result size capped at 10K chars inline (larger → write to file, reference path)
- [ ] Concurrent tool execution only for read-only or independently-scoped writes (see `§ 10`)
- [ ] Tool output masking for old turns in long sessions (see `§ 11`)

### UX

- [ ] Streaming text output (not waiting for full response)
- [ ] Tool execution shown to user with name + truncated input
- [ ] Session resume capability (`/resume <id>`)
- [ ] Slash commands: `/help`, `/sessions`, `/clear`, `/cost`, `/exit`

---

*Last updated: 2026-04-18*

---

*[← Production Agents](03-production-agents.md)*
