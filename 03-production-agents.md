# Reference Implementations — Production Agents (Python & TypeScript)

> Sections: Streaming Agent · Production Python Agent · Production TypeScript Agent

Part of the [Reference Implementations](03-minimal-agents.md) series.

## 3. Streaming Agent (Python)

Full streaming with `on_text` / `on_tool` callbacks and three tools: `bash`, `read_file`, `write_file`.

```python
import anthropic
import subprocess
import json

client = anthropic.Anthropic()

def stream_agent(prompt: str, messages: list = None, on_text=None, on_tool=None):
    """Agent with streaming output. Calls on_text for each text delta."""
    messages = messages or []
    messages.append({"role": "user", "content": prompt})
    
    tools = [
        {"name": "bash", "description": "Run shell command",
         "input_schema": {"type": "object", "properties": {"command": {"type": "string"}}, "required": ["command"]}},
        {"name": "read_file", "description": "Read a file",
         "input_schema": {"type": "object", "properties": {"path": {"type": "string"}}, "required": ["path"]}},
        {"name": "write_file", "description": "Write a file",
         "input_schema": {"type": "object", "properties": {"path": {"type": "string"}, "content": {"type": "string"}}, "required": ["path", "content"]}},
    ]
    
    while True:
        content_blocks = []
        current_text = ""
        current_tool = None
        current_json = ""
        
        with client.messages.stream(
            model="claude-opus-4-6",
            max_tokens=4096,
            tools=tools,
            messages=messages
        ) as stream:
            for event in stream:
                match type(event).__name__:
                    case "ContentBlockStart":
                        if event.content_block.type == "tool_use":
                            current_tool = {"id": event.content_block.id, "name": event.content_block.name}
                            current_json = ""
                    case "ContentBlockDelta":
                        if event.delta.type == "text_delta":
                            current_text += event.delta.text
                            if on_text:
                                on_text(event.delta.text)
                        elif event.delta.type == "input_json_delta":
                            current_json += event.delta.partial_json
                    case "ContentBlockStop":
                        if current_tool:
                            current_tool["input"] = json.loads(current_json)
                            content_blocks.append({"type": "tool_use", **current_tool})
                            current_tool = None
                        elif current_text:
                            content_blocks.append({"type": "text", "text": current_text})
                            current_text = ""
        
        messages.append({"role": "assistant", "content": content_blocks})
        
        tool_uses = [b for b in content_blocks if b["type"] == "tool_use"]
        if not tool_uses:
            return messages
        
        results = []
        for tu in tool_uses:
            if on_tool:
                on_tool(tu["name"], tu["input"])
            
            match tu["name"]:
                case "bash":
                    try:
                        r = subprocess.run(tu["input"]["command"], shell=True,
                                         capture_output=True, text=True, timeout=30)
                        output = r.stdout + r.stderr
                    except Exception as e:
                        output = str(e)
                case "read_file":
                    try:
                        with open(tu["input"]["path"]) as f:
                            output = f.read()
                    except Exception as e:
                        output = str(e)
                case "write_file":
                    try:
                        with open(tu["input"]["path"], "w") as f:
                            f.write(tu["input"]["content"])
                        output = "File written successfully"
                    except Exception as e:
                        output = str(e)
                case _:
                    output = f"Unknown tool: {tu['name']}"
            
            results.append({"type": "tool_result", "tool_use_id": tu["id"], "content": output[:10000]})
        
        messages.append({"role": "user", "content": results})


# Usage
print("Agent> ", end="", flush=True)
stream_agent(
    "List all Python files in the current directory and show their sizes",
    on_text=lambda t: print(t, end="", flush=True),
    on_tool=lambda name, inp: print(f"\n[Running {name}: {str(inp)[:60]}]", flush=True)
)
print()
```

---

## 4. Production Agent Building Blocks (Python)

Full production agent with `Config`, `PermissionError`, permission checking, hooks, all five tools, `Session`, `CostTracker`, memory file loading, system prompt builder, and the main `run_agent` loop.

```python
#!/usr/bin/env python3
"""
Production-grade coding agent.
Features: streaming, session persistence, config, hooks, permissions, REPL
"""
import anthropic
import subprocess
import json
import os
import hashlib
import readline
import time
import uuid
from pathlib import Path
from dataclasses import dataclass, field
from typing import Callable, Optional

# ─── Configuration ──────────────────────────────────────────────────────────

@dataclass
class Config:
    model: str = "claude-opus-4-6"
    max_tokens: int = 4096
    max_turns: int = 100
    permission_mode: str = "default"  # readonly, acceptEdits, default, yolo
    allow_rules: list = field(default_factory=list)
    deny_rules: list = field(default_factory=list)
    pre_tool_hooks: list = field(default_factory=list)
    max_cost_usd: float = 5.0
    
    @classmethod
    def load(cls, cwd: str) -> "Config":
        """Load and merge config from all sources."""
        sources = [
            Path.home() / ".agent" / "settings.json",
            Path(cwd) / ".agent" / "settings.json",
            Path(cwd) / ".agent" / "settings.local.json",
        ]
        result = {}
        for path in sources:
            if path.exists():
                with open(path) as f:
                    data = json.load(f)
                result.update(data)
        return cls(**{k: v for k, v in result.items() if hasattr(cls, k)})

# ─── Permission System ───────────────────────────────────────────────────────

class PermissionError(Exception):
    pass

def check_permission(tool_name: str, tool_input: dict, config: Config) -> bool:
    """Returns True if allowed, raises PermissionError if denied, None if ask."""
    for rule in config.deny_rules:
        if matches_rule(rule, tool_name, tool_input):
            raise PermissionError(f"Denied by rule: {rule}")
    
    for rule in config.allow_rules:
        if matches_rule(rule, tool_name, tool_input):
            return True
    
    read_only_tools = {"read_file", "glob_search", "grep_search", "web_fetch"}
    
    if config.permission_mode == "readonly":
        if tool_name not in read_only_tools:
            raise PermissionError(f"Read-only mode: {tool_name} not allowed")
    elif config.permission_mode == "yolo":
        return True
    elif config.permission_mode == "default":
        if tool_name == "bash":
            return None  # ASK
    
    return True

def matches_rule(rule: str, tool_name: str, tool_input: dict) -> bool:
    if "(" in rule:
        rname, pattern = rule.split("(", 1)
        pattern = pattern.rstrip(")")
        if rname.strip() != tool_name:
            return False
        input_str = json.dumps(tool_input)
        import fnmatch
        return fnmatch.fnmatch(input_str, f"*{pattern}*")
    return rule.strip() == tool_name

def prompt_user(question: str) -> bool:
    response = input(f"\n{question} [y/N] ").strip().lower()
    return response in ("y", "yes")

# ─── Hook System ─────────────────────────────────────────────────────────────

def run_pre_tool_hook(hook_cmd: str, tool_name: str, tool_input: dict) -> dict:
    event = json.dumps({
        "event": "PreToolUse",
        "tool": tool_name,
        "input": str(tool_input)[:160],
        "inputJson": tool_input
    })
    try:
        result = subprocess.run(
            hook_cmd, shell=True, input=event.encode(),
            capture_output=True, timeout=10
        )
        if result.stdout:
            return json.loads(result.stdout)
    except Exception:
        pass
    return {"allowed": True}

# ─── Tools ───────────────────────────────────────────────────────────────────

TOOL_DECLARATIONS = [
    {
        "name": "bash",
        "description": "Execute a shell command in the current directory",
        "input_schema": {
            "type": "object",
            "properties": {
                "command": {"type": "string", "description": "Shell command to run"},
                "timeout": {"type": "integer", "description": "Timeout in seconds", "default": 30}
            },
            "required": ["command"]
        }
    },
    {
        "name": "read_file",
        "description": "Read the contents of a file",
        "input_schema": {
            "type": "object",
            "properties": {
                "path": {"type": "string"},
                "offset": {"type": "integer"},
                "limit": {"type": "integer"}
            },
            "required": ["path"]
        }
    },
    {
        "name": "write_file",
        "description": "Write content to a file (creates or overwrites)",
        "input_schema": {
            "type": "object",
            "properties": {
                "path": {"type": "string"},
                "content": {"type": "string"}
            },
            "required": ["path", "content"]
        }
    },
    {
        "name": "edit_file",
        "description": "Replace text in a file",
        "input_schema": {
            "type": "object",
            "properties": {
                "path": {"type": "string"},
                "old_string": {"type": "string"},
                "new_string": {"type": "string"}
            },
            "required": ["path", "old_string", "new_string"]
        }
    },
    {
        "name": "glob_search",
        "description": "Search for files matching a glob pattern",
        "input_schema": {
            "type": "object",
            "properties": {
                "pattern": {"type": "string"},
                "path": {"type": "string"}
            },
            "required": ["pattern"]
        }
    }
]

def execute_tool(name: str, inp: dict, config: Config) -> tuple[str, bool]:
    """Returns (output, is_error)."""
    # Permission check
    try:
        allowed = check_permission(name, inp, config)
        if allowed is None:  # ASK
            if not prompt_user(f"Allow {name}({json.dumps(inp)[:60]})?"):
                return "Permission denied by user", True
    except PermissionError as e:
        return str(e), True
    
    # Pre-tool hooks
    for hook in config.pre_tool_hooks:
        result = run_pre_tool_hook(hook, name, inp)
        if not result.get("allowed", True):
            return result.get("reason", "Denied by hook"), True
        if "updatedInput" in result:
            inp = result["updatedInput"]
    
    # Execute
    try:
        match name:
            case "bash":
                cmd = inp["command"]
                timeout = inp.get("timeout", 30)
                r = subprocess.run(cmd, shell=True, capture_output=True, text=True, timeout=timeout)
                output = r.stdout + r.stderr
                return output[:10000] or "(no output)", r.returncode != 0
            
            case "read_file":
                path = inp["path"]
                with open(path) as f:
                    content = f.read()
                offset = inp.get("offset", 0)
                limit = inp.get("limit")
                lines = content.splitlines()
                if limit:
                    lines = lines[offset:offset + limit]
                else:
                    lines = lines[offset:]
                return "\n".join(lines), False
            
            case "write_file":
                path = inp["path"]
                Path(path).parent.mkdir(parents=True, exist_ok=True)
                with open(path, "w") as f:
                    f.write(inp["content"])
                return f"Written {len(inp['content'])} bytes to {path}", False
            
            case "edit_file":
                path = inp["path"]
                with open(path) as f:
                    content = f.read()
                if inp["old_string"] not in content:
                    return f"old_string not found in {path}", True
                new_content = content.replace(inp["old_string"], inp["new_string"], 1)
                with open(path, "w") as f:
                    f.write(new_content)
                return f"Replaced in {path}", False
            
            case "glob_search":
                import glob
                base = inp.get("path", ".")
                pattern = os.path.join(base, inp["pattern"])
                matches = glob.glob(pattern, recursive=True)
                return "\n".join(matches[:100]) or "(no matches)", False
            
            case _:
                return f"Unknown tool: {name}", True
    
    except Exception as e:
        return str(e), True

# ─── Session ──────────────────────────────────────────────────────────────────

class Session:
    def __init__(self, cwd: str, session_id: str = None):
        cwd_hash = hashlib.sha256(os.path.realpath(cwd).encode()).hexdigest()[:16]
        session_dir = Path.home() / ".agent" / "projects" / cwd_hash / "sessions"
        session_dir.mkdir(parents=True, exist_ok=True)
        
        self.session_id = session_id or str(uuid.uuid4())
        self.path = session_dir / f"{self.session_id}.jsonl"
        self.messages = []
        
        if self.path.exists():
            with open(self.path) as f:
                for line in f:
                    if line.strip():
                        self.messages.append(json.loads(line))
    
    def push(self, message: dict):
        self.messages.append(message)
        with open(self.path, "a") as f:
            f.write(json.dumps(message) + "\n")
    
    @classmethod
    def list_sessions(cls, cwd: str) -> list:
        cwd_hash = hashlib.sha256(os.path.realpath(cwd).encode()).hexdigest()[:16]
        session_dir = Path.home() / ".agent" / "projects" / cwd_hash / "sessions"
        sessions = sorted(session_dir.glob("*.jsonl"), key=lambda p: p.stat().st_mtime, reverse=True)
        return [p.stem for p in sessions[:10]]

# ─── Cost Tracker ─────────────────────────────────────────────────────────────

MODEL_RATES = {
    "claude-opus-4-6": {"input": 15, "output": 75, "cache_read": 1.5, "cache_write": 18.75},
    "claude-sonnet-4-6": {"input": 3, "output": 15, "cache_read": 0.3, "cache_write": 3.75},
    "claude-haiku-4-5-20251001": {"input": 0.80, "output": 4, "cache_read": 0.08, "cache_write": 1.0},
}

class CostTracker:
    def __init__(self):
        self.total_cost = 0.0
        self.input_tokens = 0
        self.output_tokens = 0
    
    def record(self, usage, model: str):
        rates = MODEL_RATES.get(model, MODEL_RATES["claude-opus-4-6"])
        cost = (
            usage.input_tokens * rates["input"] / 1_000_000 +
            usage.output_tokens * rates["output"] / 1_000_000 +
            getattr(usage, "cache_read_input_tokens", 0) * rates["cache_read"] / 1_000_000 +
            getattr(usage, "cache_creation_input_tokens", 0) * rates["cache_write"] / 1_000_000
        )
        self.total_cost += cost
        self.input_tokens += usage.input_tokens
        self.output_tokens += usage.output_tokens
        return cost

# ─── Memory Files ─────────────────────────────────────────────────────────────

def load_memory_files(cwd: str, filename: str = "CLAUDE.md") -> str:
    """Walk upward from cwd and collect memory files."""
    parts = []
    total_chars = 0
    MAX_TOTAL = 12000
    
    path = Path(os.path.realpath(cwd))
    seen = set()
    
    while True:
        for name in [filename, f"{filename.replace('.md', '.local.md')}"]:
            fpath = path / name
            if fpath.exists():
                inode = fpath.stat().st_ino
                if inode not in seen:
                    seen.add(inode)
                    content = fpath.read_text()[:4000]  # 4000 chars per file max
                    if total_chars + len(content) <= MAX_TOTAL:
                        parts.append(f"# {fpath}\n{content}")
                        total_chars += len(content)
        
        parent = path.parent
        if parent == path:
            break
        path = parent
    
    # User global
    global_path = Path.home() / ".agent" / filename
    if global_path.exists():
        content = global_path.read_text()[:4000]
        if total_chars + len(content) <= MAX_TOTAL:
            parts.insert(0, f"# {global_path}\n{content}")
    
    return "\n\n".join(parts) if parts else ""

# ─── System Prompt ────────────────────────────────────────────────────────────

def build_system_prompt(config: Config, cwd: str, memory: str) -> list:
    """Returns list of content blocks for multi-block system prompt with caching."""
    static = """You are an expert software engineer and coding assistant.
You have access to tools that let you read and write files, run shell commands, and search code.

## Core Behaviors
- Always read files before editing them
- Run tests after making changes
- Explain your reasoning before taking actions
- Ask for clarification when the request is ambiguous
- Never delete files without explicit confirmation

## Tool Usage
- Use read_file before write_file or edit_file
- Prefer edit_file over write_file for targeted changes
- Run bash tests after code changes
- Use glob_search to find relevant files first"""
    
    dynamic = f"""## Environment
- Working directory: {cwd}
- Date: {time.strftime('%Y-%m-%d')}
- OS: {os.uname().sysname}
- Model: {config.model}

## Project Context"""
    
    # Add git context if available
    try:
        git_status = subprocess.run(
            "git log --oneline -5 2>/dev/null && echo '---' && git status -s 2>/dev/null",
            shell=True, capture_output=True, text=True, cwd=cwd
        ).stdout[:1000]
        if git_status:
            dynamic += f"\n```\n{git_status}\n```"
    except Exception:
        pass
    
    if memory:
        dynamic += f"\n\n## Memory Files\n{memory}"
    
    return [
        {
            "type": "text",
            "text": static,
            "cache_control": {"type": "ephemeral"}  # Cache the static part
        },
        {
            "type": "text",
            "text": dynamic
        }
    ]

# ─── Main Agent Loop ──────────────────────────────────────────────────────────

def run_agent(
    prompt: str,
    session: Session,
    config: Config,
    cwd: str,
    on_text: Callable = None,
    on_tool: Callable = None,
    on_cost: Callable = None
) -> str:
    """Run one agent turn. Returns final text."""
    client = anthropic.Anthropic()
    memory = load_memory_files(cwd)
    system_prompt = build_system_prompt(config, cwd, memory)
    cost_tracker = CostTracker()
    
    session.push({"role": "user", "content": prompt})
    
    turn_count = 0
    final_text = ""
    
    while turn_count < config.max_turns:
        turn_count += 1
        
        if cost_tracker.total_cost >= config.max_cost_usd:
            break
        
        content_blocks = []
        tool_uses = []
        current_text = ""
        current_tool = None
        current_json = ""
        
        with client.messages.stream(
            model=config.model,
            max_tokens=config.max_tokens,
            system=system_prompt,
            tools=TOOL_DECLARATIONS,
            messages=session.messages,
            betas=["prompt-caching-2024-07-31"]
        ) as stream:
            for event in stream:
                etype = type(event).__name__
                
                if etype == "ContentBlockStart":
                    cb = event.content_block
                    if cb.type == "tool_use":
                        current_tool = {"id": cb.id, "name": cb.name}
                        current_json = ""
                    elif cb.type == "text":
                        current_text = ""
                
                elif etype == "ContentBlockDelta":
                    delta = event.delta
                    if hasattr(delta, 'text'):
                        current_text += delta.text
                        if on_text:
                            on_text(delta.text)
                    elif hasattr(delta, 'partial_json'):
                        current_json += delta.partial_json
                
                elif etype == "ContentBlockStop":
                    if current_tool:
                        current_tool["input"] = json.loads(current_json) if current_json else {}
                        tool_uses.append(current_tool.copy())
                        content_blocks.append({"type": "tool_use", **current_tool})
                        current_tool = None
                    elif current_text:
                        final_text = current_text
                        content_blocks.append({"type": "text", "text": current_text})
                        current_text = ""
            
            final_msg = stream.get_final_message()
            if final_msg and final_msg.usage:
                cost = cost_tracker.record(final_msg.usage, config.model)
                if on_cost:
                    on_cost(cost_tracker.total_cost, final_msg.usage)
        
        session.push({"role": "assistant", "content": content_blocks})
        
        if not tool_uses:
            break
        
        tool_results = []
        for tu in tool_uses:
            if on_tool:
                on_tool(tu["name"], tu["input"])
            
            output, is_error = execute_tool(tu["name"], tu["input"], config)
            tool_results.append({
                "type": "tool_result",
                "tool_use_id": tu["id"],
                "content": output,
                **({"is_error": True} if is_error else {})
            })
        
        session.push({"role": "user", "content": tool_results})
    
    return final_text
```

---


---

*[← Minimal Agents](03-minimal-agents.md) | [Next: Utilities, MCP & Checklist →](03-utilities-mcp-checklist.md)*
