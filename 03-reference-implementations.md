# Reference Implementations: Minimal to Production Agents

> Working agent implementations in Python, TypeScript, Rust, Go, and Kotlin.
> From 50-line minimal agents to production-grade patterns: streaming, sessions, loop detection, cost control, hooks, and concurrent tool execution.

---

## 1. Prerequisites

```
1. Anthropic API key
   → console.anthropic.com → API Keys → Create Key
   → Set: export ANTHROPIC_API_KEY="sk-ant-..."

2. Language requirements
   → Python:     3.11+    pip install anthropic
   → TypeScript: Node 20+ npm install @anthropic-ai/sdk
   → Rust:       1.75+    cargo add reqwest serde serde_json tokio
   → Go:         1.22+    net/http + encoding/json (stdlib only)
   → Kotlin:     JVM 21+  ktor-client + kotlinx.serialization + coroutines

3. For session storage: read/write to ~/.agent/projects/<hash>/sessions/
4. For OS keychain:     see 02-authentication.md § 6
5. For MCP tools:       pip install mcp  /  npm install @modelcontextprotocol/sdk
```

---

## 2. Minimal Agent (50 Lines)

All five implementations share the same three-step loop:
1. Append user message to history
2. Call API with tools
3. If tool_use blocks exist: execute each, append results, go to 2

### Python

```python
#!/usr/bin/env python3
"""Minimal coding agent in 50 lines."""
import anthropic, os, subprocess, json

client = anthropic.Anthropic()
MODEL = "claude-opus-4-6"

TOOLS = [{
    "name": "bash",
    "description": "Execute a shell command",
    "input_schema": {
        "type": "object",
        "properties": {"command": {"type": "string"}},
        "required": ["command"]
    }
}]

def run_bash(command: str) -> str:
    result = subprocess.run(command, shell=True, capture_output=True, text=True, timeout=30)
    output = result.stdout + result.stderr
    return output[:10000] or "(no output)"

def agent(prompt: str, history: list = None) -> str:
    messages = (history or []) + [{"role": "user", "content": prompt}]
    
    while True:
        response = client.messages.create(
            model=MODEL, max_tokens=4096, tools=TOOLS, messages=messages
        )
        
        text = next((b.text for b in response.content if b.type == "text"), "")
        tool_uses = [b for b in response.content if b.type == "tool_use"]
        
        messages.append({"role": "assistant", "content": response.content})
        
        if not tool_uses:
            return text
        
        results = []
        for tu in tool_uses:
            print(f"  → Running: {tu.input.get('command', '')[:80]}")
            output = run_bash(tu.input["command"])
            results.append({"type": "tool_result", "tool_use_id": tu.id, "content": output})
        
        messages.append({"role": "user", "content": results})

if __name__ == "__main__":
    import sys
    prompt = " ".join(sys.argv[1:]) or input("What would you like to do? ")
    print(agent(prompt))
```

### TypeScript

```typescript
#!/usr/bin/env node
import Anthropic from '@anthropic-ai/sdk';
import { execSync } from 'child_process';

const client = new Anthropic();
const MODEL = 'claude-opus-4-6';

const tools: Anthropic.Tool[] = [{
  name: 'bash',
  description: 'Execute a shell command',
  input_schema: {
    type: 'object',
    properties: { command: { type: 'string' } },
    required: ['command']
  }
}];

async function agent(prompt: string, history: Anthropic.MessageParam[] = []): Promise<string> {
  const messages: Anthropic.MessageParam[] = [
    ...history,
    { role: 'user', content: prompt }
  ];

  while (true) {
    const response = await client.messages.create({
      model: MODEL, max_tokens: 4096, tools, messages
    });

    const text = response.content
      .filter(b => b.type === 'text')
      .map(b => (b as Anthropic.TextBlock).text)
      .join('');
    
    const toolUses = response.content.filter(b => b.type === 'tool_use') as Anthropic.ToolUseBlock[];
    
    messages.push({ role: 'assistant', content: response.content });
    
    if (!toolUses.length) return text;
    
    const results: Anthropic.ToolResultBlockParam[] = toolUses.map(tu => {
      const cmd = (tu.input as any).command;
      console.error(`  → Running: ${cmd.slice(0, 80)}`);
      let output: string;
      try {
        output = execSync(cmd, { encoding: 'utf8', timeout: 30000 });
      } catch (e: any) {
        output = e.stdout + e.stderr;
      }
      return { type: 'tool_result', tool_use_id: tu.id, content: output.slice(0, 10000) };
    });
    
    messages.push({ role: 'user', content: results });
  }
}

agent(process.argv.slice(2).join(' ') || 'hello').then(console.log);
```

### Rust

> Note: This uses a hypothetical `anthropic` crate for clarity. A real implementation uses `reqwest` directly to call the Anthropic REST API — the SSE streaming pattern is shown in the Go section which uses stdlib HTTP.

```rust
use anthropic::{Client, types::*};
use std::process::Command;

#[tokio::main]
async fn main() {
    let client = Client::new(std::env::var("ANTHROPIC_API_KEY").unwrap());
    let prompt = std::env::args().skip(1).collect::<Vec<_>>().join(" ");
    
    let result = agent(&client, &prompt, vec![]).await.unwrap();
    println!("{}", result);
}

async fn agent(client: &Client, prompt: &str, mut history: Vec<Message>) -> anyhow::Result<String> {
    history.push(Message::user(prompt));
    
    let tools = vec![Tool {
        name: "bash".into(),
        description: "Execute shell command".into(),
        input_schema: serde_json::json!({
            "type": "object",
            "properties": {"command": {"type": "string"}},
            "required": ["command"]
        }),
    }];
    
    loop {
        let response = client.messages().create(MessageRequest {
            model: "claude-opus-4-6".into(),
            max_tokens: 4096,
            tools: tools.clone(),
            messages: history.clone(),
            ..Default::default()
        }).await?;
        
        let text: String = response.content.iter()
            .filter_map(|b| if let ContentBlock::Text(t) = b { Some(t.text.clone()) } else { None })
            .collect();
        
        let tool_uses: Vec<_> = response.content.iter()
            .filter_map(|b| if let ContentBlock::ToolUse(tu) = b { Some(tu.clone()) } else { None })
            .collect();
        
        history.push(Message::assistant(response.content));
        
        if tool_uses.is_empty() {
            return Ok(text);
        }
        
        let mut results = vec![];
        for tu in tool_uses {
            let cmd = tu.input["command"].as_str().unwrap_or("");
            eprintln!("  → Running: {}", &cmd[..cmd.len().min(80)]);
            let output = Command::new("sh").arg("-c").arg(cmd).output()?;
            let text = String::from_utf8_lossy(&output.stdout).to_string()
                + &String::from_utf8_lossy(&output.stderr);
            results.push(ToolResult { tool_use_id: tu.id, content: text[..text.len().min(10000)].into() });
        }
        
        history.push(Message::tool_results(results));
    }
}
```

### Go

Uses only stdlib (`net/http`, `encoding/json`, `bufio`) with manual SSE parsing.

```go
package main

import (
	"bufio"
	"bytes"
	"encoding/json"
	"fmt"
	"io"
	"net/http"
	"os"
	"os/exec"
	"strings"
)

type Message struct {
	Role    string      `json:"role"`
	Content interface{} `json:"content"`
}

type ContentBlock struct {
	Type      string          `json:"type"`
	Text      string          `json:"text,omitempty"`
	ID        string          `json:"id,omitempty"`
	Name      string          `json:"name,omitempty"`
	Input     json.RawMessage `json:"input,omitempty"`
	ToolUseID string          `json:"tool_use_id,omitempty"`
	Content   string          `json:"content,omitempty"`
	IsError   bool            `json:"is_error,omitempty"`
}

var tools = []map[string]interface{}{{
	"name":        "bash",
	"description": "Execute a shell command",
	"input_schema": map[string]interface{}{
		"type": "object",
		"properties": map[string]interface{}{
			"command": map[string]interface{}{"type": "string"},
		},
		"required": []string{"command"},
	},
}}

func callAPI(apiKey string, messages []Message) ([]ContentBlock, error) {
	body, _ := json.Marshal(map[string]interface{}{
		"model":      "claude-opus-4-6",
		"max_tokens": 4096,
		"stream":     true,
		"messages":   messages,
		"tools":      tools,
	})

	req, _ := http.NewRequest("POST", "https://api.anthropic.com/v1/messages",
		bytes.NewReader(body))
	req.Header.Set("x-api-key", apiKey)
	req.Header.Set("anthropic-version", "2023-06-01")
	req.Header.Set("content-type", "application/json")

	resp, err := http.DefaultClient.Do(req)
	if err != nil {
		return nil, err
	}
	defer resp.Body.Close()

	if resp.StatusCode != 200 {
		b, _ := io.ReadAll(resp.Body)
		return nil, fmt.Errorf("API %d: %s", resp.StatusCode, string(b))
	}

	var blocks []ContentBlock
	var current ContentBlock
	var inputBuf strings.Builder

	scanner := bufio.NewScanner(resp.Body)
	for scanner.Scan() {
		line := scanner.Text()
		if !strings.HasPrefix(line, "data: ") {
			continue
		}
		data := strings.TrimPrefix(line, "data: ")

		var event map[string]interface{}
		if json.Unmarshal([]byte(data), &event) != nil {
			continue
		}

		switch event["type"] {
		case "content_block_start":
			cb := event["content_block"].(map[string]interface{})
			current = ContentBlock{Type: cb["type"].(string)}
			if id, ok := cb["id"].(string); ok { current.ID = id }
			if name, ok := cb["name"].(string); ok { current.Name = name }
			inputBuf.Reset()

		case "content_block_delta":
			delta := event["delta"].(map[string]interface{})
			switch delta["type"] {
			case "text_delta":
				current.Text += delta["text"].(string)
				fmt.Print(delta["text"].(string))
			case "input_json_delta":
				inputBuf.WriteString(delta["partial_json"].(string))
			}

		case "content_block_stop":
			if current.Type == "tool_use" {
				current.Input = json.RawMessage(inputBuf.String())
				fmt.Fprintf(os.Stderr, "\n  → [%s]\n", current.Name)
			}
			blocks = append(blocks, current)
		}
	}
	return blocks, scanner.Err()
}

func runBash(cmdStr string) string {
	out, _ := exec.Command("sh", "-c", cmdStr).CombinedOutput()
	result := string(out)
	if len(result) > 10000 { result = result[:10000] }
	return result
}

func agent(prompt string, apiKey string) string {
	messages := []Message{{Role: "user", Content: prompt}}
	finalText := ""

	for turn := 0; turn < 100; turn++ {
		blocks, err := callAPI(apiKey, messages)
		if err != nil {
			fmt.Fprintln(os.Stderr, "Error:", err)
			break
		}

		messages = append(messages, Message{Role: "assistant", Content: blocks})

		var toolUses []ContentBlock
		for _, b := range blocks {
			if b.Type == "tool_use" { toolUses = append(toolUses, b) }
			if b.Type == "text" { finalText = b.Text }
		}

		if len(toolUses) == 0 { break }

		var results []ContentBlock
		for _, tu := range toolUses {
			var input map[string]string
			json.Unmarshal(tu.Input, &input)
			output := runBash(input["command"])
			results = append(results, ContentBlock{
				Type: "tool_result", ToolUseID: tu.ID, Content: output,
			})
		}
		messages = append(messages, Message{Role: "user", Content: results})
	}
	return finalText
}

func main() {
	apiKey := os.Getenv("ANTHROPIC_API_KEY")
	if apiKey == "" { fmt.Fprintln(os.Stderr, "ANTHROPIC_API_KEY not set"); os.Exit(1) }
	prompt := strings.Join(os.Args[1:], " ")
	if prompt == "" { prompt = "List files in current directory" }
	fmt.Println()
	agent(prompt, apiKey)
	fmt.Println()
}
```

### Kotlin

Uses `ktor-client-cio`, `kotlinx.serialization`, and Kotlin coroutines. Streaming is handled via `HttpStatement.execute {}`.

```kotlin
// build.gradle.kts:
// implementation("io.ktor:ktor-client-core:2.3.7")
// implementation("io.ktor:ktor-client-cio:2.3.7")
// implementation("io.ktor:ktor-client-content-negotiation:2.3.7")
// implementation("io.ktor:ktor-serialization-kotlinx-json:2.3.7")
// implementation("org.jetbrains.kotlinx:kotlinx-serialization-json:1.6.3")

import io.ktor.client.*
import io.ktor.client.engine.cio.*
import io.ktor.client.plugins.contentnegotiation.*
import io.ktor.client.request.*
import io.ktor.client.statement.*
import io.ktor.http.*
import io.ktor.utils.io.*
import kotlinx.coroutines.*
import kotlinx.serialization.*
import kotlinx.serialization.json.*

val MODEL = "claude-opus-4-6"
val API_KEY = System.getenv("ANTHROPIC_API_KEY") ?: error("ANTHROPIC_API_KEY not set")

val json = Json { ignoreUnknownKeys = true }

val TOOLS = listOf(mapOf(
    "name" to "bash",
    "description" to "Execute a shell command",
    "input_schema" to mapOf(
        "type" to "object",
        "properties" to mapOf("command" to mapOf("type" to "string")),
        "required" to listOf("command")
    )
))

data class Block(
    val type: String,
    val text: String = "",
    val id: String = "",
    val name: String = "",
    val input: Map<String, String> = emptyMap(),
    val toolUseId: String = ""
)

data class Msg(val role: String, val content: Any)

fun runBash(command: String): String {
    val proc = ProcessBuilder("sh", "-c", command)
        .redirectErrorStream(true)
        .start()
    val output = proc.inputStream.bufferedReader().readText()
    proc.waitFor()
    return output.take(10000).ifEmpty { "(no output)" }
}

suspend fun callApi(messages: List<Map<String, Any>>): List<Block> {
    val client = HttpClient(CIO) { install(ContentNegotiation) { json(json) } }
    val blocks = mutableListOf<Block>()

    val body = mapOf(
        "model" to MODEL,
        "max_tokens" to 4096,
        "stream" to true,
        "messages" to messages,
        "tools" to TOOLS
    )

    client.preparePost("https://api.anthropic.com/v1/messages") {
        header("x-api-key", API_KEY)
        header("anthropic-version", "2023-06-01")
        contentType(ContentType.Application.Json)
        setBody(Json.encodeToString(body))
    }.execute { response ->
        val channel = response.bodyAsChannel()
        var currentType = ""
        var currentId = ""
        var currentName = ""
        var currentText = StringBuilder()
        var inputJson = StringBuilder()

        while (!channel.isClosedForRead) {
            val line = channel.readUTF8Line() ?: break
            if (!line.startsWith("data: ")) continue
            val data = line.removePrefix("data: ")
            val event = try { Json.parseToJsonElement(data).jsonObject } catch (_: Exception) { continue }

            when (event["type"]?.jsonPrimitive?.content) {
                "content_block_start" -> {
                    val cb = event["content_block"]!!.jsonObject
                    currentType = cb["type"]!!.jsonPrimitive.content
                    currentId   = cb["id"]?.jsonPrimitive?.content ?: ""
                    currentName = cb["name"]?.jsonPrimitive?.content ?: ""
                    currentText = StringBuilder()
                    inputJson   = StringBuilder()
                }
                "content_block_delta" -> {
                    val delta = event["delta"]!!.jsonObject
                    when (delta["type"]?.jsonPrimitive?.content) {
                        "text_delta" -> {
                            val t = delta["text"]!!.jsonPrimitive.content
                            currentText.append(t)
                            print(t)
                        }
                        "input_json_delta" -> inputJson.append(delta["partial_json"]!!.jsonPrimitive.content)
                    }
                }
                "content_block_stop" -> {
                    when (currentType) {
                        "text"     -> blocks.add(Block("text", text = currentText.toString()))
                        "tool_use" -> {
                            val inp = Json.decodeFromString<Map<String, String>>(inputJson.toString())
                            System.err.println("\n  → [$currentName]")
                            blocks.add(Block("tool_use", id = currentId, name = currentName, input = inp))
                        }
                    }
                }
            }
        }
    }
    client.close()
    return blocks
}

suspend fun agent(prompt: String): String {
    val history = mutableListOf<Map<String, Any>>(
        mapOf("role" to "user", "content" to prompt)
    )
    var finalText = ""

    repeat(100) { _ ->
        val blocks = callApi(history)
        val contentList = blocks.map { b ->
            when (b.type) {
                "text"     -> mapOf("type" to "text", "text" to b.text)
                "tool_use" -> mapOf("type" to "tool_use", "id" to b.id, "name" to b.name, "input" to b.input)
                else -> mapOf("type" to b.type)
            }
        }
        history.add(mapOf("role" to "assistant", "content" to contentList))

        val toolUses = blocks.filter { it.type == "tool_use" }
        finalText = blocks.firstOrNull { it.type == "text" }?.text ?: finalText

        if (toolUses.isEmpty()) return finalText

        val results = toolUses.map { tu ->
            val output = runBash(tu.input["command"] ?: "")
            mapOf("type" to "tool_result", "tool_use_id" to tu.id, "content" to output)
        }
        history.add(mapOf("role" to "user", "content" to results))
    }
    return finalText
}

fun main(args: Array<String>) = runBlocking {
    val prompt = args.joinToString(" ").ifBlank { "List files in current directory" }
    println()
    agent(prompt)
    println()
}
```

---

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
