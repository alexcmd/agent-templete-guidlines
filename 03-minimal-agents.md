# Reference Implementations — Quick Reference & Minimal Agents

> Sections: Quick Reference Card · Minimal Agent (~50 lines) in Python/TypeScript/Rust/Go/Kotlin/C++

Part of the [Reference Implementations](03-production-agents.md) series.

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


---

*[Next: Production Agents →](03-production-agents.md)*
